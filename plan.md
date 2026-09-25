# Plan: Book PDF Content Extractor

> **Project rules:** Read [CLAUDE.md](CLAUDE.md) before starting any step. It holds the project's conventions, commands and decisions, and it overrides this plan if they disagree. The facts about the sample book in §2 are also recorded there.

---

## 1. Goal

Build a command-line application that:

1. Takes a **book PDF**, a **folder of PDFs**, or a **zip of PDFs**. The source tree looks like `Books/<Class>/<Subject>/*.pdf`.
2. Extracts the book's content (text, math, figures, tables), even when the PDF has **no usable text layer**.
3. Works out the **subject**, the **chapters** and the **exercises**.
4. Writes a folder tree where **subject** is the top folder and **chapter** folders sit inside it. Each chapter's **exercises** go in their own subfolder.
5. Saves each chapter in a neutral structured format (`content.json`). From that file it generates **HTML** and **plain text** now, and can generate other formats later without processing the PDF again.

---

## 2. What the sample book showed (Step 0, done 2026-09-25)

Sample: `Books/Class-10/Algebra/Class X - Algebra - WithTeachers.In.pdf` (24.8 MB, 128 pages, A4).

| Finding | Evidence | Consequence for the design |
|---|---|---|
| **Language is Odia** (Odisha Board, Class X Algebra). Math is in Latin letters and Arabic digits. English glosses appear in parentheses, e.g. `ସରଳ ସହସମୀକରଣ (LINEAR SIMULTANEOUS EQUATIONS)` | Rendered pages | All text is Unicode Odia. Folder slugs use the **English gloss**, and the Odia title is kept in JSON |
| **No usable text layer.** 122 of 128 pages hold only a 24-character watermark. Their text is **vector outlines** (600–1100 drawing paths per page), not fonts or images | PyMuPDF: `get_text()` ≈ 24 chars, `get_drawings()` ≈ 700–1100 | **OCR is required for every page.** Normal text extraction and font-size heading detection **cannot work** on this book |
| The 6 pages that do have text (e.g. 78, 80–82, 113, 127) have **broken font encoding**: the output is garbage | `get_text()` returns `!"#"!$%"#&'$` etc. | Never trust the text layer blindly. Validate it (Step 4) and fall back to OCR |
| No bookmarks/outline. Metadata title is `10th Algebra_2018 F.pdf` | `get_toc()` returns `[]` | Chapter boundaries come from the printed **contents page** and chapter title pages |
| **Contents page** (ସୂଚୀ) is on physical page 5 | Rendered p5 | Parse it to get the chapter list and page ranges |
| **Page offset is +6** (printed page 1 = physical page 7). The printed number `[ N ]` sits at the bottom centre | p8 shows `[ 2 ]`, p29 shows `[ 23 ]` | Map printed pages to physical pages. Remove `[ N ]` from the content |
| Footer watermark `https://withteachers.in/` on every page | Rendered pages | Remove it |
| **Chapter title page:** ordinal + `ଅଧ୍ୟାୟ` (e.g. `ପ୍ରଥମ ଅଧ୍ୟାୟ` = "First Chapter"), a QR code, the Odia title, and the English title in parentheses | p7, p29 | Chapter start marker |
| **Sections:** `1.1 ଉପକ୍ରମଣିକା (Introduction) :`, `1.2 …`, `2.2 …` | p7, p8, p30 | Heading pattern `^\d+\.\d+\s` |
| **Exercises:** `ଅନୁଶୀଳନୀ - 1 (c)` = "Exercise 1(c)", centred and bold. Questions are numbered `1.`, `2.`, … with sub-parts `(i)`, `(ii)`. Chapter 1 has 1(a), 1(b), 1(c) | p14 (1a), p23 (1b), p27 (1c) | Exercise folder `exercise-1c`. Questions split on numbering |
| **Worked examples:** `ଉଦାହରଣ - 12 :` ("Example 12") followed by `ସମାଧାନ :` ("Solution"). Some end with `(ଉତ୍ତର)` ("answer") | p25, p26 | Kept in the chapter body as `example` blocks, **not** treated as exercises |
| **Answers** section `ଉତ୍ତରମାଳା` on printed 119–122 (physical 125–128) | Contents page | Match answers to exercises. Save them to `_answers/` |
| Figures (graphs, diagrams) are vector drawings, not embedded images | p8 figure 1.1 | Crop figures by **rendering the page region** to PNG |
| Two-column layout used for **answer lists and sub-questions** only. Body text is single column | p24, p27 | OCR must keep reading order within these blocks |

### Chapter map (from the contents page)

| # | Odia title | English (for slug) | Printed pages | Physical pages |
|---|---|---|---|---|
| — | front matter (cover, preface, ସୂଚୀ) | — | — | 1–6 |
| 1 | ସରଳ ସହସମୀକରଣ | Linear Simultaneous Equations | 1–22 | 7–28 |
| 2 | ଦ୍ୱିଘାତ ସମୀକରଣ | Quadratic Equations | 23–41 | 29–47 |
| 3 | ସମାନ୍ତର ପ୍ରଗତି | Arithmetic Progression | 42–62 | 48–68 |
| 4 | ସମ୍ଭାବ୍ୟତା | Probability | 63–76 | 69–82 |
| 5 | ପରିସଂଖ୍ୟାନ | Statistics | 77–100 | 83–106 |
| 6 | ସ୍ଥାନାଙ୍କ ଜ୍ୟାମିତି | Coordinate Geometry | 101–118 | 107–124 |
| — | ଉତ୍ତରମାଳା (answers) | — | 119–122 | 125–128 |

The English glosses for chapters 3–6 are **assumed** from the Odia titles. Confirm each one against its chapter title page during OCR.

---

## 3. Target output layout

The output mirrors the source tree, so the same subject name in different classes can't collide: `output/<class>/<subject>/…`. The **subject folder is the top level for its content**. (If you want subject as the very top level, change `output.layout` in config. Nothing else changes.)

```
output/
└── class-10/
    └── algebra/                                 subject folder
        ├── subject.json                         subject metadata, language, chapter index
        ├── index.html                           table of contents (Odia + English)
        ├── style.css
        ├── _front-matter/                       cover, preface, contents (content.* only)
        ├── _answers/                            ଉତ୍ତରମାଳା, split per exercise
        │   └── answers.json
        ├── chapter-01-linear-simultaneous-equations/
        │   ├── content.json                     canonical structured content (source of truth)
        │   ├── content.html
        │   ├── content.txt
        │   ├── images/
        │   │   └── fig-1-1-i.png                cropped from the rendered page
        │   ├── pages/                           per-page OCR JSON (cache + audit trail)
        │   │   └── p007.json
        │   └── exercises/
        │       ├── exercise-1a/
        │       │   ├── exercise.json
        │       │   ├── exercise.html
        │       │   └── exercise.txt
        │       ├── exercise-1b/
        │       └── exercise-1c/
        ├── chapter-02-quadratic-equations/
        │   └── …
        └── _report.json                         warnings, page ranges, OCR stats, cost
```

Rules:
- Folder names are ASCII **slugs** built from the English gloss (`chapter-01-…`). The Odia title is kept in `content.json` and shown in the HTML.
- Exercises are named from the book's own label: `ଅନୁଶୀଳନୀ - 1 (c)` becomes `exercise-1c`. Unlabelled exercises fall back to `exercise-01`, `exercise-02`.
- Exercise text is **removed** from the chapter's `content.*` and replaced by an `exercise_ref` block, so nothing appears twice.
- Re-running is deterministic and **cheap**: OCR results are cached per page in `pages/`.

---

## 4. Technology choices

| Concern | Choice | Why |
|---|---|---|
| Language | Python 3.12+ (3.14.7 is installed) | Best PDF tooling |
| Render pages, crop figures, read text layer | **PyMuPDF** (1.28.2 installed; use `import pymupdf`, not `fitz`) | Fast rasterising, drawing and text analysis |
| **OCR (primary)** | **Claude vision** through the Anthropic Python SDK, model `claude-opus-5` | Reads Odia script **and** math in one pass. Returns structured JSON (headings, questions, LaTeX math, figure boxes). Tesseract can't do math and is weak on Odia |
| OCR (offline fallback, optional) | Tesseract 5 + `ori` traineddata + `pytesseract` | Free and offline, but plain text only. **Not installed** |
| Bulk OCR | **Message Batches API** | 50% cheaper. 128 pages don't need a real-time response |
| Structured output | `client.messages.parse()` with a Pydantic page schema | The response is guaranteed to match the schema, so no fragile text parsing |
| Math in HTML | **KaTeX** (from a CDN, or vendored for offline use) | Renders the LaTeX that OCR returns |
| Odia font in HTML | Noto Sans Oriya | Correct rendering of conjuncts |
| HTML templates | `jinja2` | |
| CLI | `typer` | |
| Config | `pyyaml` | |
| Data models | `pydantic` v2 | Shared by the OCR schema, `content.json` and validation |
| Slugs | `python-slugify` | |
| Tests | `pytest` | |
| Lint/format | `ruff` | |

**Rough OCR cost for this book.** One page at ~1600 px is about 1.5–2k input tokens. The JSON output is about 2–3k tokens. For 128 pages that is roughly 250k input and 350k output tokens, or **about $10 at standard `claude-opus-5` prices ($5 / $25 per million tokens) and about $5 through the Batch API**. Thinking tokens add to this. Measure it on 5 pages before running the whole book (Step 5).

---

## 5. Proposed code layout

```
C:\new-workspace\
├── CLAUDE.md
├── plan.md
├── pyproject.toml
├── README.md
├── .env.example                     ANTHROPIC_API_KEY=
├── config/
│   ├── default.yaml                 generic defaults
│   └── books/
│       └── odisha-class10-algebra.yaml   markers, page offset, chapter map overrides
├── src/bookextract/
│   ├── cli.py                       `bookextract run|inspect|ocr|build|render`
│   ├── sources.py                   PDF / folder / zip input, safe unzip, class/subject from path
│   ├── inspect.py                   Step 0 report (text-layer health, drawings, page samples)
│   ├── render.py                    page rasterisation + region cropping
│   ├── textlayer.py                 text-layer extraction + garbage detection
│   ├── ocr/
│   │   ├── schema.py                Pydantic PageOCR model
│   │   ├── prompt.py                system prompt + per-book hints
│   │   ├── claude.py                single-page and batch OCR, caching, retries
│   │   └── tesseract.py             optional fallback
│   ├── cleaner.py                   drop watermark/page numbers, Unicode NFC, join across pages
│   ├── toc.py                       contents-page parsing + page-offset detection
│   ├── chapters.py                  chapter segmentation
│   ├── exercises.py                 exercise/question splitting, answer matching
│   ├── figures.py                   crop figures from bbox, number them
│   ├── models.py                    Book, Chapter, Block, Exercise, Question (pydantic)
│   ├── writer.py                    folder tree + JSON output (atomic writes)
│   ├── renderers/
│   │   ├── html.py
│   │   ├── text.py
│   │   └── templates/ chapter.html.j2, exercise.html.j2, index.html.j2, style.css
│   └── report.py
├── tests/
│   ├── fixtures/                    tiny generated PDFs + saved PageOCR JSON (no API calls in tests)
│   └── test_*.py
├── Books/                           source books (git-ignored)
├── .work/                           per-PDF scratch: renders, inspect reports, OCR cache (git-ignored)
└── output/                          generated results (git-ignored)
```

---

## 6. Step-by-step implementation plan

### Step 0: Inspect the sample ✅ (done; results in §2)
Keep this as a real command, `bookextract inspect <pdf>`, so every new book gets the same check. It reports:
- page count, metadata, bookmarks
- the health of each page's text layer: characters, drawing count, image count, and a garbage score
- a verdict per page: `text-ok` / `garbled` / `vector-outlines` / `scanned-image`, which decides between text extraction and OCR
- thumbnail contact sheets written to a scratch folder so a person can see the layout

### Step 1: Project setup and CLAUDE.md ✅
1. `git init`. `.gitignore` covers `Books/`, `output/`, `.venv/`, `.env`, `__pycache__/`.
2. `python -m venv .venv`, then `pyproject.toml` with dependencies and a `bookextract` console script.
3. API credentials: the SDK reads `ANTHROPIC_API_KEY` from the environment (or an `ant auth login` profile). Never commit keys. `.env.example` documents the variable.
4. Keep **CLAUDE.md** up to date: commands, conventions, the output contract (§3) and the sample-book facts (§2).

**Done when:** `pip install -e .[dev]` works and `bookextract inspect` runs on the sample.

### Step 2: Input sources (`sources.py`) ✅
1. Accept a single PDF, a folder (searched recursively) or a zip.
2. For zips: **zip-slip protection** (reject absolute paths and `..`), nested zips one level deep, skip non-PDF files, handle cp437 filenames.
3. Take **class** and **subject** from the path `Books/<Class>/<Subject>/`. `Class-10/Algebra` gives `class-10` / `algebra`. A `--subject` / `--class` CLI override wins.
4. Order PDFs by natural sort when a subject has several.

**Done when:** tests pass for a PDF, a folder, a zip, a zip-slip attack zip and mixed file types.

### Step 3: Page rendering and text-layer check (`render.py`, `textlayer.py`) ✅
The render cache (`.work/<pdf-hash>/`) is built. It is first used by OCR in Step 5. Item 3 (drawing bounding boxes) is postponed to Step 9 (figures).

1. Render each page at **200 DPI** (~1650×2340 px for A4) to PNG, cached in a temp/work folder. This is the OCR input.
2. Try the text layer first. Accept it only if it is **real text**: a high share of Odia/Latin/digit characters, few control characters, and real words. On this book every page fails the check, so every page goes to OCR. That's expected.
3. Keep `get_drawings()` bounding boxes. They help find figure regions later.

### Step 4: Per-book config (`config/books/odisha-class10-algebra.yaml`)
Put the §2 facts in config, not in code:
```yaml
language: or                 # ISO 639-1 Odia
page_offset: 6               # physical = printed + 6
contents_page: 5             # physical
strip_patterns:
  - 'https?://withteachers\.in/?'
  - '^\[\s*\d+\s*\]$'        # printed page number
markers:
  chapter:  '(ପ୍ରଥମ|ଦ୍ୱିତୀୟ|ତୃତୀୟ|ଚତୁର୍ଥ|ପଞ୍ଚମ|ଷଷ୍ଠ)\s*ଅଧ୍ୟାୟ'
  section:  '^\d+\.\d+\s'
  exercise: 'ଅନୁଶୀଳନୀ\s*[-–]\s*(\d+)\s*\(([a-z])\)'
  example:  'ଉଦାହରଣ\s*[-–]\s*(\d+)'
  solution: 'ସମାଧାନ\s*:'
  answers:  'ଉତ୍ତରମାଳା'
ordinals: { ପ୍ରଥମ: 1, ଦ୍ୱିତୀୟ: 2, ତୃତୀୟ: 3, ଚତୁର୍ଥ: 4, ପଞ୍ଚମ: 5, ଷଷ୍ଠ: 6 }
```
The OCR model also labels block types itself (next step). These regexes are a **cross-check**, not the only detector. Note that Odia text needs Unicode NFC normalisation before any regex is matched.

### Step 5: OCR with Claude vision (`ocr/`)
This is the core step for this book.

1. **Schema (`ocr/schema.py`)**: one `PageOCR` per page:
   ```python
   class Block(BaseModel):
       type: Literal[
           "chapter_title",
           "section_heading",
           "paragraph",
           "math",
           "example_heading",
           "solution",
           "exercise_heading",
           "question",
           "sub_question",
           "answer_list",
           "table",
           "figure",
           "caption",
           "page_number",
           "watermark",
           "other",
       ]
       text: str  # Odia/English Unicode; inline math as $...$
       latex: str | None  # for display math blocks
       label: str | None  # "1.2", "1 (c)", "12", "(ii)"
       bbox: list[float] | None  # normalised 0–1 [x0,y0,x1,y1], used for figure crops
       rows: list[list[str]] | None  # tables


   class PageOCR(BaseModel):
       physical_page: int
       printed_page: int | None
       blocks: list[Block]  # in reading order
       continues_from_previous: bool
       notes: str | None  # uncertainty flagged by the model
   ```
2. **Prompt (`ocr/prompt.py`)**: a fixed system prompt, placed first so it can be **prompt-cached**, that says:
   - transcribe Odia exactly in Unicode (no translation, no transliteration)
   - write all mathematics as LaTeX (`$…$` inline, `latex` field for display equations and matrices)
   - label blocks using the book's markers (give the Odia marker words from config as hints)
   - give figures a bbox and caption, and don't transcribe their internal labels as paragraphs
   - mark the watermark and page number so the cleaner can drop them
   - report doubtful glyphs in `notes` and never invent text
3. **Calls (`ocr/claude.py`)**:
   - model `claude-opus-5`, adaptive thinking, `client.messages.parse(..., output_format=PageOCR)` for guaranteed-valid JSON
   - one page image per request, plus the previous page's last block as context for text that runs across pages
   - check `stop_reason` before reading content (`refusal`, `max_tokens`). Retry on 429/5xx with the SDK's built-in retries
   - **cache**: `pages/pNNN.json`, keyed by the PDF hash + page + prompt version. Never OCR a page twice
4. **Bulk mode**: `bookextract ocr <pdf> --batch` sends all pages uncached so far through the **Message Batches API** (50% cost). It polls until done, then stores results by `custom_id` = page number. Results come back in any order.
5. **Pilot first:** OCR pages 5 (contents), 7 (chapter start), 8 (section + figure), 27 (exercise) and 125 (answers) with real-time calls. Check the quality by eye and record the measured tokens and cost in `_report.json`. Only then run the batch.
6. **Quality checks**: flag pages with low Odia-character share, empty blocks, or a `printed_page` that doesn't equal physical − offset.

**Done when:** all 128 pages have a cached `PageOCR`, and the pilot pages read correctly when compared side by side with the page image.

### Step 6: Cleaning (`cleaner.py`)
1. Drop `watermark` and `page_number` blocks and anything matching `strip_patterns`.
2. Unicode NFC. Normalise Odia look-alike code points (e.g. `ୱ` vs `ଵ`) according to a small mapping table.
3. Join a paragraph or question that `continues_from_previous` onto the last block of the page before.
4. Keep the LaTeX exactly as returned (no reformatting).

### Step 7: Contents parsing and chapter segmentation (`toc.py`, `chapters.py`)
1. Parse the OCR of the contents page (physical 5) into rows of `{ordinal, odia_title, printed_start, printed_end}`.
2. **Check the page offset automatically**: compare OCR'd `printed_page` against physical page numbers across the book. If the result differs from config, warn.
3. For each chapter, find the physical page with a `chapter_title` block and check it matches the contents page. Take the **English gloss** from the parentheses on the title page, for the slug.
4. Everything before chapter 1 goes to `_front-matter/`. The `ଉତ୍ତରମାଳା` pages go to `_answers/`.

**Done when:** 6 chapters are found with the exact physical ranges in §2's chapter map.

### Step 8: Exercises (`exercises.py`)
1. An exercise starts at an `exercise_heading` block (`ଅନୁଶୀଳନୀ - 1 (c)`). It ends at the next exercise heading, the next section heading, the next chapter or the end of the book part.
2. Split it into questions on `question` blocks (`1.`, `2.`, …) with nested `sub_question`s (`(i)`, `(ii)`, `(a)`). Math and figures inside a question stay with it.
3. **Worked examples** (`ଉଦାହରଣ`) and their solutions stay in the chapter body as `example` blocks. They are *not* exercises. (A later option can export them to `examples/` too.)
4. **Answer matching**: split the `ଉତ୍ତରମାଳା` OCR by exercise label (`1 (a)`, `1 (b)`, …) and question number. Attach answers to `exercise.json` → `questions[n].answer`. List unmatched answers in `_report.json`.
5. In the chapter body, replace each exercise with an `exercise_ref` block.

**Done when:** chapter 1 gives `exercise-1a`, `exercise-1b`, `exercise-1c`, and each has all its questions (compare counts against the pages by eye) with answers attached.

### Step 9: Figures (`figures.py`)
1. For each `figure` block, crop its bbox from a **300 DPI** render of the page (vector graphics make crops sharp) with a small margin. Save as `images/fig-<chapter>-<n>.png`.
2. Use the caption (`ଚିତ୍ର 1.1 (i)`) for the filename and alt text when present.
3. Skip the QR code on chapter title pages (a config flag).

### Step 10: Data model and `content.json`
```jsonc
{
  "schema_version": 1,
  "class": "Class-10", "subject": "Algebra", "language": "or",
  "chapter": { "number": 1,
               "title": "ସରଳ ସହସମୀକରଣ", "title_en": "Linear Simultaneous Equations",
               "slug": "chapter-01-linear-simultaneous-equations",
               "printed_pages": [1, 22], "physical_pages": [7, 28] },
  "blocks": [
    { "type": "heading", "level": 2, "label": "1.1", "text": "ଉପକ୍ରମଣିକା (Introduction)", "page": 7 },
    { "type": "paragraph", "text": "… $ax + b = 0$ …", "page": 7 },
    { "type": "math", "latex": "a_1x + b_1y + c_1 = 0", "page": 7 },
    { "type": "example", "label": "12", "blocks": [ … ], "solution": [ … ] },
    { "type": "image", "src": "images/fig-1-1-i.png", "caption": "ଚିତ୍ର 1.1 (i)", "page": 8 },
    { "type": "exercise_ref", "id": "exercise-1c", "path": "exercises/exercise-1c/" }
  ],
  "extraction": { "method": "claude-ocr", "model": "claude-opus-5", "prompt_version": 1, "warnings": [] }
}
```
`exercise.json` uses the same block types plus `questions: [{number, blocks, sub_questions, answer?}]`.
**`content.json` is the source of truth.** HTML, text and any future format are generated from it.

### Step 11: Renderers (`renderers/`)
1. **HTML**
   - `<html lang="or">`, the Noto Sans Oriya font, and **KaTeX** auto-render for `$…$` and display math
   - semantic markup (`<article>`, `<section>`, `<figure>`, `<ol>` for questions)
   - English glosses shown next to Odia headings
   - exercise refs link to `exercises/<id>/exercise.html`. Answers sit in a collapsed `<details>` block
   - `index.html` per subject. Jinja autoescape on
   - `--fragment` mode outputs only the `<article>` body, for embedding in another site
   - `--offline` vendors KaTeX and the font into `output/…/assets/`
2. **Plain text**: UTF-8, headings numbered, **math kept as LaTeX source** (`$x^2$`), figures shown as `[ଚିତ୍ର 1.1: fig-1-1-i.png]`, tables as aligned columns.
3. `bookextract render <output/class-10/algebra> --format html|txt` re-renders from JSON with no OCR and no API calls.

### Step 12: Writer and report (`writer.py`, `report.py`)
1. Build the tree from §3 with atomic writes (temp file, then rename).
2. `--clean` wipes the subject folder first. Without it, the command refuses to overwrite unless `--force` is given.
3. `_report.json` covers: chapter ranges, exercise and question counts, unmatched answers, pages flagged by OCR QA, token usage and **actual cost**.

### Step 13: CLI (`cli.py`)
```
bookextract inspect <pdf|folder|zip>                    # Step 0 report + thumbnails
bookextract ocr     <src> [--pages 5,7-9] [--batch]     # fill the per-page OCR cache
bookextract build   <src> [--out output/] [--config …]  # cache → chapters/exercises/JSON
bookextract render  <output/class/subject> --format html,txt [--offline] [--fragment]
bookextract run     <src>                               # ocr + build + render
```
Each stage reads the previous stage's files, so a bug in exercise splitting is fixed with `build` + `render`, with **no new OCR spend**.

### Step 14: Testing
1. **No API calls in tests.** Fixtures are saved `PageOCR` JSON for a few real pages (pilot results) plus small generated PDFs.
2. Unit tests: zip safety, path → class/subject, text-layer garbage detection, strip patterns, Odia NFC plus regex markers, contents parsing, page-offset detection, exercise and question splitting, answer matching, slugify.
3. Golden test: fixture OCR → `build` → compare the output tree and JSON against saved expected files.
4. Manual check: open chapter 1 and `exercise-1c` HTML next to PDF pages 7–28. Check the Odia, the math rendering and the question count.

### Step 15: Documentation
- `README.md`: install, API key setup, usage, output layout, cost note.
- Update **CLAUDE.md** whenever commands, conventions or the output contract change.
- Tick off milestones below as they are completed.

---

## 7. Milestones

| # | Milestone | Steps | Acceptance check |
|---|---|---|---|
| M0 | Sample inspected | 0 | ✅ findings in §2 |
| M1 | Setup + inspect command | 1, 2, 3 | ✅ 2026-09-25: 122 `vector-outlines` + 6 `garbled` = 128/128 need OCR. 23 tests pass |
| M2 | OCR pilot | 4, 5 (pilot) | The 5 pilot pages read correctly. Cost per page is measured |
| M3 | Full OCR | 5 (batch), 6 | 128 cached pages, QA flags reviewed |
| M4 | Structure | 7, 8, 9, 10 | 6 chapter folders with exact ranges, exercise folders, figures, answers attached |
| M5 | Output formats | 11, 12, 13 | HTML shows Odia and math correctly in a browser. TXT readable. `render` works offline |
| M6 | Hardening | 14, 15 | Tests pass. README and CLAUDE.md current |

---

## 8. Risks and how to handle them

| Risk | Mitigation |
|---|---|
| OCR errors in Odia conjuncts or math | Pilot plus side-by-side review. `notes` field for doubtful glyphs. Keep the per-page JSON so single pages can be re-OCR'd (`ocr --pages N --force`) |
| The model "fixes" or invents text | The prompt forbids it. Compare the question count against the numbering sequence. Flag gaps (e.g. questions 1, 2, 4) |
| Cost overrun | Pilot measures the real cost first. Batch API. Per-page cache. The prompt stays unchanged between runs so it gets cached |
| Other books use different markers or layout | Every marker lives in `config/books/*.yaml`. Run `inspect` + a pilot on each new book |
| Some books *do* have a good text layer | The text-layer check (Step 3) uses it and skips paid OCR for those pages |
| Chapter title glosses 3–6 assumed | Take them from the OCR'd title pages. Stop with an error if a title page has no gloss, unless config supplies one |
| Two-column answer lists read in the wrong order | The prompt asks for column-wise reading. Answer matching checks that the numbers form a sequence |
| Copyright of the source book | `Books/` and `output/` are git-ignored. Output is for personal/teaching use |
| API key leakage | Environment variable or `ant auth login` only. `.env` is git-ignored |

## 9. Later extensions (out of scope for v1)
- Export worked examples (`ଉଦାହରଣ`) to their own `examples/` folders
- English translation of content (a separate, clearly labelled output next to the original)
- Markdown / EPUB renderers over `content.json`
- A search index or small web viewer over `output/`
- Tag question types (MCQ, short answer, proof) and difficulty
