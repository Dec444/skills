# CLAUDE.md

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `Dec444/skills` via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Five canonical triage roles using their default label strings (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout — one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Skill prerequisites

### `literature-note`

Needs a PDF text-extractor. The skill tries them in this order and stops at the first one that works — install any one of:

- `pip3 install --user pypdf` (recommended — pure Python, no system deps)
- `brew install poppler` (provides `pdftotext`)
- `pip3 install --user pdfplumber` (used by `anthropic-skills:pdf` fallback)

Also requires `curl` (universal) for arXiv / direct-PDF URL inputs.

### `latex-check`

No external dependencies. Reads `.tex` files directly.
