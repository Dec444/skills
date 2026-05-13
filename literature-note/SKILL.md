---
name: literature-note
description: |
  Turn a written argumentative document (research paper, essay, manifesto, technical report, policy paper) into a structured literature note + BibTeX entry. Accepts a local PDF path, an arXiv URL, or any HTTPS URL that ends in `.pdf`. Writes `~/Documents/notes/literature/<citekey>.md` (YAML frontmatter + TL;DR / Problem / Method / Key claims / My take) and appends to `references.bib`.
  TRIGGER when: user says "make a lit note on this paper", "add this paper to my notes", "process X into a note", or asks for a persistent citeable record of any written argumentative work — including non-peer-reviewed essays, books, blog posts, or position papers, if the user asks explicitly.
  SKIP when: user wants a chat-style summary with no file written, asks "what's in this paper" / "explain this paper", wants figures / tables / data extracted, or the input is not a written argumentative document (dataset, code, image-only scan with no OCR layer, legal contract, executable).
---

# Literature note

## What this skill does

Takes one document at a time. Produces two writes:

1. A structured markdown note at `~/Documents/notes/literature/<citekey>.md`
2. An appended BibTeX entry in `~/Documents/notes/literature/references.bib`

The user fills the "My take" section by hand later. Every other section is filled from the document.

## When invoked

The user has either:

- Given a local PDF path (e.g. `~/Downloads/liu2024.pdf`)
- Given an arXiv URL (e.g. `https://arxiv.org/abs/2403.12345`, `/pdf/2403.12345.pdf`, with optional `v2`/`v3` suffix)
- Given any other HTTPS URL whose path ends in `.pdf` (publisher landing PDFs, self-hosted essays, technical reports — anything where curl can fetch a PDF directly)
- Said "make a note on this paper" while a PDF is already attached / open in the conversation

If the input is ambiguous, the URL doesn't end in `.pdf`, or no document is identifiable, ask for the path or URL once. Do not guess. Do not try to scrape HTML landing pages — defer DOI/journal-page parsing.

## Step 1 — get the paper text

### 1a. Resolve to a local PDF path

| Input type | Action |
|---|---|
| Local path | Use as-is. Confirm the file exists. |
| arXiv URL | Normalize: `https://arxiv.org/abs/<id>` → `https://arxiv.org/pdf/<id>.pdf`. Keep any `vN` suffix. Download to `/tmp/<citekey-fragment>.pdf` with `curl -sL -o`. |
| Other `*.pdf` URL | Download with `curl -sL -o /tmp/<slug>.pdf` (no normalization). |

After this step you have a local `.pdf` file. Delete any `/tmp/` downloads in Step 7 (the report).

### 1b. Extract text — fallback chain

Try in this order. **Never auto-install anything.** Stop at the first one that works.

1. **Python `pypdf`** (preferred — pure Python, no system deps):
   ```bash
   python3 -c "from pypdf import PdfReader" 2>/dev/null && echo OK
   ```
   If available, extract with:
   ```python
   from pypdf import PdfReader
   r = PdfReader("<path>")
   for i in range(len(r.pages)):
       print(r.pages[i].extract_text())
   ```
   Also pull metadata via `r.metadata` — this gives author/title/date for properly-typeset PDFs.

2. **`pdftotext`** (poppler-utils, system binary):
   ```bash
   command -v pdftotext >/dev/null && pdftotext -layout <path> -
   ```

3. **`anthropic-skills:pdf`** skill, if installed. Note that this skill itself often needs `pypdf` or `pdfplumber` to actually do the extraction — if it errors out, treat it as a miss.

4. **None of the above worked.** Stop with this exact message to the user:

   > Cannot extract PDF text. Install one of:
   > - `pip3 install --user pypdf` (recommended — lightest)
   > - `brew install poppler` (provides `pdftotext`)
   > - `pip3 install --user pdfplumber` (alternative Python lib)
   >
   > Then re-run the skill.

   Do not proceed past this point without text. Do not invent content.

### 1c. Decide which pages to read

For a typical paper (<30 pages): read all of it.

For a long document (>30 pages): read the title page, table of contents (if present), the introduction or framing chapter, and the conclusion / parting thoughts. Sample 1–2 representative body pages per chapter you'll need to summarize. You do not need to read every page — the four LLM-filled sections can be drawn from framing + structure.

## Step 2 — extract metadata

Determine:

- **authors**: list, publication order, as `Last, First` strings. (Some PDFs embed this in PDF metadata — prefer that over title-page parsing when both agree.)
- **year**: publication year. For self-published essays, use the cover-page or PDF-metadata date. For arXiv preprints, the submission year.
- **title**: as printed on the title page, including punctuation and subtitle.
- **venue**: where it appeared.
  - Journal paper → journal name + year
  - Conference paper → conference name + year
  - arXiv preprint → `arXiv preprint`
  - Self-published essay / blog post / personal-site PDF → `Self-published (<domain>)` or `Self-published essay`
  - Tech report → `<organization> Technical Report`
  - If genuinely unclear, omit the field — don't guess.
- **url**: canonical landing URL. For arXiv, the `abs/` URL (not `pdf/`). For self-published, the document's home page if the PDF URL is hot-linked.
- **doi**: if available; otherwise omit.

If a field is genuinely missing from the document (e.g. no venue for an essay), leave it absent rather than guessing.

## Step 3 — compute citekey

Format: `<firstauthorlastname><year><firstcontentword>`, all lowercase, no separators, no diacritics.

- `firstauthorlastname`: surname only of the first listed author, lowercased, no accents (`López` → `lopez`).
- `year`: four digits.
- `firstcontentword`: the first word of the title that is not an article / preposition (skip `a`, `an`, `the`, `on`, `of`, `for`, `in`, `to`, `with`, `and`). Lowercase, alphabetic only.

Example: Liu et al. (2024), "Adaptive Literature Notes for Researchers" → `liu2024adaptive`.
Example: Aschenbrenner (2024), "Situational Awareness: The Decade Ahead" → `aschenbrenner2024situational`.

The citekey doubles as the filename: `~/Documents/notes/literature/<citekey>.md`.

## Step 4 — collision check

`mkdir -p ~/Documents/notes/literature` first if the directory does not exist.

If `~/Documents/notes/literature/<citekey>.md` already exists:

- If the user passed `--force` (or said "force" / "overwrite"), proceed and overwrite.
- Otherwise, refuse with: `Note already exists at <path>. Re-run with --force to overwrite, or delete the file first.` Do not silently merge or rename. Stop.

If two genuinely different documents would collide on the same citekey (different paper, same first-author-year-firstword), tell the user and ask them to disambiguate by suggesting `<citekey>b` or a different first-content-word. Do not pick silently.

## Step 5 — write the note

Template:

```markdown
---
citekey: liu2024adaptive
authors:
  - Liu, Lu
  - Doe, Jane
year: 2024
title: "Adaptive Literature Notes for Researchers"
venue: NeurIPS 2024
url: https://arxiv.org/abs/2403.12345
doi: 10.48550/arXiv.2403.12345
tags: []
added: 2026-05-13
---

# Adaptive Literature Notes for Researchers

## TL;DR

One or two sentences in your own words. What the document does and why it matters.

## Problem

What gap or question motivated this work? State it as the document does, in 1–3 sentences.

## Method

How did the authors solve it? Algorithm, experimental design, theoretical framework. 3–6 sentences. No equations unless one is genuinely load-bearing.

## Key claims

- Bullet form. One claim per bullet.
- Evidence-bearing: what they showed, not what they hoped.
- 3–7 bullets.

## My take

<!-- Your turn. The skill does not write this. -->
```

### Section-heading flexibility by document type

The `## Method` heading is the only one that flexes. Pick the heading that best fits the document:

| Document type | Use this heading instead of `## Method` |
|---|---|
| Empirical research paper, methods paper, algorithm paper | `## Method` (default) |
| Theory paper, math paper | `## Method` (describe the theoretical framework / proof strategy) |
| Position paper, essay, manifesto, op-ed | `## Argument structure` |
| Book, book chapter, standard, specification | `## Contents` |
| Review article, survey | `## Coverage` |
| Policy paper, white paper | `## Argument structure` |

For non-`## Method` headings, the body still describes *how* the document makes its case — just in language that fits an essay/book/etc. (e.g. "Three linked claims, one per chapter…" rather than "We trained a model on…").

The other four sections (`TL;DR`, `Problem`, `Key claims`, `My take`) always keep their names.

### Other template details

Fill `tags` with 2–4 lowercase keywords drawn from the document's stated keywords or topic. If none are obvious, leave the list empty.

`added` is today's date in `YYYY-MM-DD` format. Use the current date from the agent's environment.

Leave the `## My take` body as the literal HTML comment shown above — the user fills it.

## Step 6 — append BibTeX

Path: `~/Documents/notes/literature/references.bib` (create if absent).

Before appending: scan the existing file for `@<type>{<citekey>,`. If the citekey is already present, skip the append and tell the user. Do not produce duplicates.

### Entry type selection

Pick the BibTeX entry type by document type:

| Document type | Entry type | Required fields beyond author/title/year/url |
|---|---|---|
| Journal article | `@article` | `journal`, `volume`, `number` (if known), `pages` (if known) |
| Conference / workshop paper | `@inproceedings` | `booktitle` (conference short name) |
| arXiv preprint (not yet at a venue) | `@misc` | `archivePrefix = {arXiv}`, `eprint = {<id>}`, `primaryClass = {<cat>}` |
| Book | `@book` | `publisher` |
| Book chapter | `@incollection` | `booktitle`, `publisher`, `editor` (if known) |
| Tech report | `@techreport` | `institution`, `number` (if known) |
| PhD / master's thesis | `@phdthesis` / `@mastersthesis` | `school` |
| Self-published essay, blog post, personal-site PDF | `@misc` | `howpublished = {Self-published essay, \url{...}}` |
| Standard / specification | `@misc` | `howpublished = {<org> standard <number>}` |
| Uncertain | `@misc` | `howpublished` with a free-text description |

### Examples

```bibtex
@inproceedings{liu2024adaptive,
  author    = {Liu, Lu and Doe, Jane},
  title     = {Adaptive Literature Notes for Researchers},
  booktitle = {NeurIPS},
  year      = {2024},
  url       = {https://arxiv.org/abs/2403.12345},
  doi       = {10.48550/arXiv.2403.12345},
}

@misc{aschenbrenner2024situational,
  author       = {Aschenbrenner, Leopold},
  title        = {Situational Awareness: The Decade Ahead},
  year         = {2024},
  month        = jun,
  howpublished = {Self-published essay, \url{https://situational-awareness.ai/}},
  url          = {https://situational-awareness.ai/},
}

@misc{vaswani2017attention,
  author        = {Vaswani, Ashish and others},
  title         = {Attention Is All You Need},
  year          = {2017},
  archivePrefix = {arXiv},
  eprint        = {1706.03762},
  primaryClass  = {cs.CL},
  url           = {https://arxiv.org/abs/1706.03762},
}
```

Author names joined with ` and ` (BibTeX convention). Omit fields you don't have rather than emitting empty values.

## Step 7 — report

Tell the user:

- Path of the written note
- Whether a BibTeX entry was appended or skipped (and why)
- Which extraction tool was used (so the user knows what to keep installed)
- One-sentence reminder to fill the "My take" section

Delete any `/tmp/` PDFs you downloaded in Step 1a.

Do not summarize the document's contents in the chat reply — the note is the summary. The chat reply is a receipt, not a duplicate.
