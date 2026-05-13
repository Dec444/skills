---
name: latex-check
description: |
  Lint a LaTeX paper for common semantic-vs-presentational misuse. Reads `.tex` file(s), auto-detects loaded packages (natbib / biblatex / amsthm / siunitx), and reports findings as `file:line: message`. Read-only — does not modify files.
  TRIGGER when: user asks to check / lint / review a `.tex` or LaTeX file for style, semantic issues, citation usage, or correct package usage.
  SKIP when: user wants compile errors (use `latexmk` / `chktex`), wants spelling/grammar (use a prose linter), or wants the file rewritten (this skill is read-only — auto-fix is out of scope for v1).
---

# LaTeX semantic check

## What this skill does

Reports semantic-vs-presentational misuse in LaTeX source. Always read-only. Output is `file:line: message` lines to stdout, plus a summary count at the end.

## When invoked

The user names a path. Either:

- A single `.tex` file → check just that file
- A directory → check every `**/*.tex` under it, sorted by path
- No path given → check `*.tex` in the current working directory

If the directory has no `.tex` files, say so and stop. Do not search above the named path.

## Step 1 — collect files and read each one

For each file, read the full contents. You'll need both line numbers and surrounding context (1–2 sentences before/after each finding) for the citation rule.

## Step 2 — detect loaded packages

Scan each file's preamble (the region above `\begin{document}`) for `\usepackage[...]{<name>}` calls. The four packages that matter for conditional rules:

- `natbib` → enables citation rule (textual vs parenthetical)
- `biblatex` → also enables the citation rule, with a different vocabulary (`\textcite` / `\parencite` instead of `\citet` / `\citep`)
- `amsthm` → enables theorem-environment rule
- `siunitx` → enables units rule

If you encounter `\input{<path>}` or `\include{<path>}` inside the preamble, follow it **one hop** to look for package loads. Do not recurse further.

If detection fails (e.g. the preamble is in an unreferenced file, or you're checking a fragment), fall back to a **hedged message** for the conditional rules: prefix findings with `(natbib?)`, `(amsthm?)`, etc. so the user can ignore false positives.

## Step 3 — run the core 5 rules

These always fire, regardless of preamble.

### Rule E1 — emphasis with `\textit` / `\textbf`

Flag `\textit{...}` and `\textbf{...}` when used for emphasis. Heuristic: it's almost certainly emphasis if the brace contents are short (≤ 4 words) and not a recognized title pattern (book titles, journal names — those are legitimate `\textit`).

> Message: `line N: \textit used for emphasis — prefer \emph{...} which adapts to surrounding context.`

### Rule E2 — straight quotes

Flag any standalone `"` character that's not inside a verbatim / lstlisting / minted environment.

> Message: `line N: straight quote — use \`\`...'' for double quotes in LaTeX.`

### Rule E3 — hand-typed cross-references

Flag patterns like:

- `Figure\s+\d+` (e.g. `Figure 3`)
- `Section\s+\d+`
- `Table\s+\d+`
- `Equation\s+\d+`
- `Fig\.\s+\d+`

…when they appear in body text (not inside `\caption` or comments).

> Message: `line N: hand-typed reference 'Figure 3' — use \autoref{fig:...} or \cref{fig:...} with a \label so numbering stays correct.`

Also flag `Figure~\ref{...}` (manual prefix + `\ref`) — prefer `\autoref` / `\cref` which manage the prefix word automatically.

### Rule E4 — display math with `$$ ... $$`

Flag any `$$ ... $$` block.

> Message: `line N: $$...$$ is plain TeX — prefer \[ ... \], or use an equation / align environment if the math should be numbered.`

### Rule E5 — `\\` for paragraph break

Flag `\\` when it appears at end of a line in body text (outside any `tabular`, `array`, `align`, `aligned`, `cases`, `matrix`, or similar environment).

> Message: `line N: \\ used for a paragraph break — use a blank line. \\ is for line breaks within tables/arrays/aligned math only.`

## Step 4 — run conditional rules (preamble-aware)

### Rule C1 — citation style (natbib / biblatex)

Fires only if `natbib` or `biblatex` is loaded (or in hedged mode if detection failed).

For each `\cite{<key>}` (or `\cite[...]{<key>}`):

1. Read 1–2 sentences of context around the cite.
2. Classify as **textual** (cite acts as a noun in the sentence — e.g. *"As shown by \cite{liu2024}, X works"*) or **parenthetical** (cite is an aside — e.g. *"X works (\cite{liu2024})"* or *"This builds on prior work \cite{liu2024}."*).
3. If textual + natbib: recommend `\citet{<key>}`. If parenthetical + natbib: recommend `\citep{<key>}`.
4. If textual + biblatex: recommend `\textcite{<key>}`. If parenthetical + biblatex: recommend `\parencite{<key>}`.
5. If you can't confidently classify, emit: `line N: \cite{<key>} ambiguous — review whether textual (\citet/\textcite) or parenthetical (\citep/\parencite).`

Do not flag `\cite{}` if the user is using plain LaTeX bibliographic commands (no natbib / biblatex loaded and no hedge).

### Rule C2 — theorem environments (amsthm)

Fires only if `amsthm` is loaded (or hedged).

Flag patterns where a theorem-like declaration is rendered with `\textbf{...}` instead of an environment:

- `\textbf{Theorem.}` / `\textbf{Theorem 1.}` / `\textbf{Lemma.}` / `\textbf{Proposition.}` / `\textbf{Corollary.}` / `\textbf{Definition.}` / `\textbf{Proof.}`
- `\noindent\textbf{Theorem 1}` etc.

> Message: `line N: theorem-like declaration as \textbf — use a \begin{theorem}...\end{theorem} environment (declare with \newtheorem{theorem}{Theorem}).`

For `\textbf{Proof.}` specifically, suggest the `proof` environment from amsthm.

### Rule C3 — units (siunitx)

Fires only if `siunitx` is loaded (or hedged).

Flag bare number-unit pairs in body text. Unit vocabulary (case-sensitive, with optional prefix):

> `m, cm, mm, km, s, ms, μs, ns, g, mg, kg, Hz, kHz, MHz, GHz, K, °C, °F, V, mV, A, mA, W, mW, kW, J, mJ, N, Pa, mol, L, mL, dB, %, ppm`

Pattern: a number (`\d+(\.\d+)?`) followed by optional whitespace followed by one of those units, not inside math mode and not inside `\SI{}{}`.

> Message: `line N: bare unit '5 mg/kg' — use \SI{5}{\milli\gram\per\kilo\gram} (siunitx) for correct spacing and locale-aware formatting.`

If `siunitx` isn't loaded but you see lots of bare units, emit one informational note at the end: `consider \usepackage{siunitx} for unit formatting`.

## Step 5 — output

For each finding, emit one line:

```
<path>:<line>: <message>
```

Sorted by `(path, line)`. Multiple findings on one line print on consecutive lines.

After all findings, print a summary:

```
--
<N> findings across <M> files
```

If `N == 0`, print only `<M> files clean.` and exit.

## What this skill does NOT do

- Does not modify files. No `--fix` flag in v1.
- Does not check compile correctness — that's `latexmk` / `chktex`'s job.
- Does not check spelling, grammar, or prose style — use a prose linter.
- Does not enforce a house style (line length, sentence-per-line, etc.).

If the user asks for any of the above, suggest the appropriate tool and stop.
