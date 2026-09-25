# CLAUDE.md

## Project
`bookextract`: a Python CLI that turns textbook PDFs (a single PDF, a folder or a zip) into a folder tree per subject: chapters, each chapter's exercises in subfolders, and each chapter's content saved as `content.json`, from which HTML and plain text are generated. The full step-by-step plan is in [plan.md](plan.md). Follow it, and tick off milestones there as they are completed.

## Status
- M0 + M1 done (2026-09-25): setup, input sources, text-layer check, `bookextract inspect`.
- Next: M2, the OCR pilot (plan.md Steps 4-5). It needs `ANTHROPIC_API_KEY`, which is not set on this machine yet.

## Key facts about the sample book (don't re-derive these)
- `Books/Class-10/Algebra/Class X - Algebra - WithTeachers.In.pdf`: 128 pages, **Odia language** (Odisha Board), math in LaTeX-able Latin notation.
- **No usable text layer.** Text is vector outlines. The few text pages are garbled. **Every page needs OCR** (Claude vision, plan.md Step 5).
- Exercises in chapter 1: 1(a) p14, 1(b) p23, 1(c) p27 (physical pages).
- Physical page = printed page + 6. The contents page is physical page 5. Answers (`ଉତ୍ତରମାଳା`) are on physical pages 125–128.
- Markers: chapter `… ଅଧ୍ୟାୟ`, section `1.2`, exercise `ଅନୁଶୀଳନୀ - 1 (c)`, worked example `ଉଦାହରଣ - 12`, solution `ସମାଧାନ :`.
- Remove on every page: `https://withteachers.in/` watermark, `[ N ]` page number.

## Conventions
- Python 3.12+ (3.14.7 installed). `import pymupdf`, not `fitz` (deprecated).
- Pydantic v2 models in `src/bookextract/models.py`. Type hints everywhere. `logging`, not `print`.
- Book-specific patterns and numbers go in `config/books/*.yaml`, never hard-coded.
- Normalise all text to Unicode NFC before any regex match.
- `content.json` is the source of truth. Renderers never read PDFs.
- OCR results are cached per page. Never re-OCR a page unless `--force` is given (it costs money).
- Claude API: model `claude-opus-5`, `client.messages.parse()` with the Pydantic schema, Batch API for bulk runs. Tests must never call the API. They use saved OCR JSON fixtures.

## Output contract (plan.md §3; change it only by updating the plan)
`output/<class>/<subject>/chapter-NN-<english-slug>/{content.json,content.html,content.txt,images/,pages/,exercises/exercise-<label>/}`, plus `subject.json`, `index.html`, `_front-matter/`, `_answers/`, `_report.json`.

## Commands
- Setup: `python -m venv .venv` → `.venv\Scripts\activate` → `pip install -e .[dev]`
- Test: `pytest` · Lint: `ruff check .` · Format: `ruff format .`
- Inspect: `bookextract inspect Books/Class-10/Algebra [--no-sheets] [--subject X] [--class Y]` (takes ~45 s on the sample; report and thumbnail sheets go to `.work/<pdf-hash>/inspect/`)
- Scratch folder: `.work/` (override it with the `BOOKEXTRACT_WORK` environment variable). Book configs are auto-matched by the `match:` globs in `config/books/*.yaml`.

## Gotchas
- Windows git ignores case: keep ignore rules root-anchored (`/Books/`), or `config/books/` gets ignored too.

## Don'ts
- Don't commit `Books/`, `output/`, `.env` or API keys.
- Don't translate or "correct" book text during OCR. Transcribe it exactly.
