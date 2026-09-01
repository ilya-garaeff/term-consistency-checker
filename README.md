# termcheck

Checks a translated deliverable against a glossary. Flags terms rendered outside the approved list, the same term rendered inconsistently, numbers that changed, and untranslated fragments. Web UI and CLI.

## Run

```bash
pip install -r requirements.txt

make serve    # http://127.0.0.1:8000, click "Load sample"
make check    # CLI against the sample files
make eval     # 11 seeded-defect cases, currently 11/11 offline
make verify   # 16 assertions

export ANTHROPIC_API_KEY=...   # enables the adjudication pass
```

## What the code does

Four checks in `termcheck/checks.py`, all deterministic, all sub-10ms:

- **Glossary compliance, per aligned segment.** If a source segment contains an approved term and the matching target segment contains no approved rendering, that is an error. Alignment is positional; a segment-count mismatch is reported as a warning and the check falls back to whole-document scope.
- **Internal consistency.** The same source term rendered two different approved ways in one document. `_non_overlapping_hits()` prevents the false positive where `provider` always looks present inside `service provider`.
- **Numbers.** Present in one side and not the other. `1 500,00`, `1,500.00` and `1500` normalise equal.
- **Script leakage.** Cyrillic in a Latin target, CJK in a non-`zh` target.

`termcheck/semantic.py` is the only model call, and it only runs on flags that failed the literal glossary pass — typically two or three per document. It asks one question: does the target segment contain an acceptable inflected variant of the approved term, or a real deviation? This exists because `поставщик услуг` and `поставщику услуг` are the same term with no string overlap at the end, and stemming gets Russian aspect pairs wrong.

The adjudication prompt offers `unclear` as a first-class verdict and states what each kind of mistake costs, rather than asking "is this okay?".

`--no-model` skips the pass entirely. Everything still runs, with more false positives, at zero cost.

`termcheck/server.py` is a FastAPI app with one endpoint, `POST /api/check`. `termcheck/web/index.html` is the whole front end: three textareas, a findings rail, and a synchronized highlight — clicking a finding marks the term in the source and every candidate rendering in the target at once.

## Where things are

| File | Lines | What it holds |
| --- | --- | --- |
| `termcheck/checks.py` | ~230 | Glossary parsing, segmentation, alignment, the four checks |
| `termcheck/semantic.py` | ~75 | Adjudication prompt, segment-scoped payload, verdict handling |
| `termcheck/server.py` | ~65 | FastAPI app, `/api/check` |
| `termcheck/web/index.html` | ~290 | Full UI including the synchronized highlight |
| `termcheck/cli.py` | ~50 | Arguments, `--fail-on` exit codes for CI |
| `termcheck/report.py` | ~35 | Text and dict rendering |
| `evals/run_eval.py` | ~110 | 11 seeded cases, recall and false-positive counts |
| `tests/smoke.py` | ~75 | 16 assertions: each check fires, the overlap fix holds, plus the API |

## Evals

Ground truth by construction. Start from a target known to be clean, inject one specific defect, check whether it is reported.

Seven seeded defects: unapproved synonym, wrong legal term of art, dropped term, changed number, dropped number, inconsistent rendering, untranslated fragment. Four cases that must stay clean: the unmodified text, two harmless rewordings, and one substitution of an accepted alternative that should produce a `note` and never an error.

**Current result, offline: 7/7 recall, 4/4 clean.** That is against the offline stub, which approximates a stemmer. It is a real result for the deterministic layer, which does most of the work here — but it does not tell you how the model adjudication performs on genuine inflection, because the offline stub is not doing inflection. That comparison is unrun.

The eval is also a regression suite. The `provider` / `service provider` substring bug is a permanent case in it.

## Limits

- Alignment is positional. One merged or split sentence shifts every later pair; the tool warns and degrades to whole-document rather than reporting wrong segment numbers.
- Multi-word terms crossing a sentence boundary are missed.
- Numbers written as words ("thirty days" vs "30 days") are not matched.
- Homographs produce false positives the model pass only partly catches.
- Plain text input only. No `.docx`, no `.sdlxliff`. This is the main thing standing between "works" and "usable on real deliverables".
- The glossary is enforced, never validated. A wrong glossary gets enforced confidently.

## Model

Sonnet at temperature 0, one call per document at most. `--fail-on error|warning|note|never` controls the exit code for pipeline use.
