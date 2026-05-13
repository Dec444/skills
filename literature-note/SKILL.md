---
name: literature-note
description: |
  Turn a research paper (local PDF or arXiv URL) into a structured literature note + BibTeX entry. Writes `~/Documents/notes/literature/<citekey>.md` (YAML frontmatter + TL;DR / Problem / Method / Key claims / My take) and appends to `references.bib`.
  TRIGGER when: user says "make a lit note on this paper", "add this paper to my notes", "process X into a note", or asks for a persistent citeable record of an academic paper.
  SKIP when: user wants a chat-style summary, asks "what's in this paper", wants figures/tables/data extracted, or the document is not an academic paper.
---

# Literature note

## What this skill does

Takes one paper at a time. Produces two writes:

1. A structured markdown note at `~/Documents/notes/literature/<citekey>.md`
2. An appended BibTeX entry in `~/Documents/notes/literature/references.bib`

The user fills the "My take" section by hand later. Every other section is filled from the paper.

## When invoked

The user has either:

- Given a local PDF path (e.g. `~/Downloads/liu2024.pdf`)
- Given an arXiv URL (e.g. `https://arxiv.org/abs/2403.12345` or `https://arxiv.org/pdf/2403.12345.pdf` or with a `v2` suffix)
- Said "make a note on this paper" while a PDF is already attached / open in the conversation

If the input is ambiguous or no paper is identifiable, ask for the path or URL once. Do not guess.

## Step 1 — get the paper text

**Local PDF.** Extract text. Prefer `pdftotext -layout <path> -` (poppler-utils, usually pre-installed on macOS dev machines via homebrew). If unavailable, use `anthropic-skills:pdf` if installed, else tell the user to install poppler with `brew install poppler` and stop.

**arXiv URL.** Normalize the URL to `https://arxiv.org/pdf/<id>.pdf` (strip `/abs/`, keep any version suffix `v2`/`v3`). Download to a temp file with `curl -sL`. Extract as above. Delete the temp file when done.

Read the abstract, introduction, and conclusion sections in full. You don't need to read every page — the four LLM-filled sections can be drawn from those three.

## Step 2 — extract metadata

From page 1 (and arxiv metadata if you have it), determine:

- **authors**: list, in publication order, as `Last, First` strings
- **year**: publication year (or arxiv submission year if unpublished)
- **title**: as printed, including punctuation
- **venue**: conference / journal name + year, or `arXiv preprint` if unpublished
- **url**: arXiv abs URL (preferred), DOI, or publisher landing page
- **doi**: if available

If any field is genuinely missing from the paper (e.g. no venue because it's an arXiv preprint), leave it absent rather than guessing.

## Step 3 — compute citekey

Format: `<firstauthorlastname><year><firstcontentword>`, all lowercase, no separators, no diacritics.

- `firstauthorlastname`: surname only of the first listed author, lowercased, no accents (`López` → `lopez`)
- `year`: four digits
- `firstcontentword`: the first word of the title that is not an article / preposition (skip `a`, `an`, `the`, `on`, `of`, `for`, `in`, `to`, `with`). Lowercase, alphabetic only.

Example: Liu et al. (2024), "Adaptive Literature Notes for Researchers" → `liu2024adaptive`.

The citekey doubles as the filename: `~/Documents/notes/literature/liu2024adaptive.md`.

## Step 4 — collision check

If `~/Documents/notes/literature/<citekey>.md` already exists:

- If the user passed `--force` (or said "force" / "overwrite"), proceed and overwrite.
- Otherwise, refuse with: `Note already exists at <path>. Re-run with --force to overwrite, or delete the file first.` Do not silently merge or rename. Stop.

`mkdir -p ~/Documents/notes/literature` first if the directory does not exist.

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

One or two sentences in your own words. What the paper does and why it matters.

## Problem

What gap or question motivated this work? State it as the paper does, in 1–3 sentences.

## Method

How did the authors solve it? Algorithm, experimental design, theoretical framework. 3–6 sentences. No equations unless one is genuinely load-bearing.

## Key claims

- Bullet form. One claim per bullet.
- Evidence-bearing: what they showed, not what they hoped.
- 3–7 bullets.

## My take

<!-- Your turn. The skill does not write this. -->
```

Fill `tags` with 2–4 lowercase keywords drawn from the paper's stated keywords or topic. If none are obvious, leave the list empty.

`added` is today's date in `YYYY-MM-DD` format. Use the current date from the agent's environment.

Leave the `## My take` body as the literal HTML comment shown above — the user fills it.

## Step 6 — append BibTeX

Path: `~/Documents/notes/literature/references.bib` (create if absent).

Before appending: scan the existing file for `@<type>{<citekey>,`. If the citekey is already present, skip the append and tell the user. Do not produce duplicates.

Entry shape (use `@article` for journals, `@inproceedings` for conferences, `@misc` for arXiv-only preprints):

```bibtex
@inproceedings{liu2024adaptive,
  author    = {Liu, Lu and Doe, Jane},
  title     = {Adaptive Literature Notes for Researchers},
  booktitle = {NeurIPS},
  year      = {2024},
  url       = {https://arxiv.org/abs/2403.12345},
  doi       = {10.48550/arXiv.2403.12345},
}
```

Author names joined with ` and ` (BibTeX convention). Omit fields you don't have rather than emitting empty values.

## Step 7 — report

Tell the user:

- Path of the written note
- Whether a BibTeX entry was appended or skipped (and why)
- One-sentence reminder to fill the "My take" section

Do not summarize the paper's contents in the chat reply — the note is the summary. The chat reply is a receipt, not a duplicate.
