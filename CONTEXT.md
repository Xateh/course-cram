# course-cram — Context Management Reference

Read this file before dispatching any subagent. The prompt templates below include the no-loss clause and return contracts that must be used as specified.

---

## Why subagents own ingests and cheatsheet artefacts

Visual PDF rendering costs ~1000-1500 tokens per page in the main conversation. A subagent reading the same page and writing a dense markdown transcription to disk returns only a short manifest (~50 tokens) to main context. Running all source reads through subagents:

- Keeps the main context footprint to: `INDEX.md` metadata + session state + current rendered response
- Makes ingests persistent — they survive across sessions; re-map is never needed unless materials change
- Enables per-topic granular updates — re-transcribe one topic without touching others

**Trade-off**: each teach/quiz/exam/cheatsheet turn dispatches a subagent (adds latency). This is the correct trade-off: compaction-free sessions and cross-session persistence outweigh per-turn overhead.

The same pattern applies to cheatsheet `.tex` artefacts: generation subagents write the file and return a manifest only; main agent never reads full ingest content. The Verify subagent reads only the narrow page ranges cited in the cheatsheet provenance comments — never the full ingest.

---

## Directory layout

```
<course-root>/.course-cram/
├── ingests/
│   ├── INDEX.md              ← metadata only; main-agent-readable
│   ├── <topic-slug>.md       ← per-topic transcription; subagent-only
│   └── ...
├── cheatsheets/              ← created on first cheatsheet generate
│   ├── INDEX.md              ← cheatsheet manifest; main-agent-written
│   └── <name>.tex            ← generated LaTeX; subagent-written
└── session.md                ← session state; main-agent-readable
```

Main agent may read: `INDEX.md`, `session.md`, `cheatsheets/INDEX.md`, generated `.tex` files (small, for Verify parsing), and the output of `ls ingests/` / `ls cheatsheets/`.
Main agent must NOT read: any `<topic-slug>.md` file.

---

## Topic grouping heuristics

- Default: **one topic per unit** (week, lecture, tutorial set), inferred from filename patterns:
  - `Week N`, `Wk N`, `W0N` → slug `week-0N-<label>`
  - `Tutorial N`, `Tut N`, `T0N` → slug `week-0N-tutorial` (grouped with the same unit's lecture)
  - `Lecture N`, `Lec N` → slug `week-0N-lecture` (or merged with tutorial under `week-0N`)
- Merge lecture + tutorial for the same unit into a single slug unless the user requests separation
- Cross-unit materials get dedicated slugs:
  - Past-paper midterms → `past-papers-midterm`
  - Past-paper finals → `past-papers-final`
  - Admin/course info → `admin`
- Slug rule: lowercase, hyphens only, ≤ 40 chars (e.g., `week-05-capital-structure`)

---

## Subagent prompt templates

Every subagent prompt is a self-contained instruction block. Copy the relevant template, fill in the placeholders, and use it as the Agent tool prompt.

### No-information-loss clause (required verbatim in templates 1 and 5)

```
**CRITICAL — NO INFORMATION LOSS**: The output must be a complete transcription with NO information loss. Rephrasing for dense formatting is permitted; summarization, paraphrase, or omission is not. If a page contains 30 items, your transcription lists 30 items. If a chart has 5 annotations, all 5 appear. Every equation transcribed with original variable names and structure. Every diagram reconstructed with every node, edge, label, and arrow. Every table transcribed cell-by-cell with all headers and row labels intact.
```

---

### Template 1 — Ingest write (map phase)

**Subagent type**: general-purpose (needs Read + Write + Bash)

```
Task: Read the following course source files and write a complete dense transcription to disk.

Target ingest file: <course-root>/.course-cram/ingests/<slug>.md
Topic label: <topic label>

Source files to read:
- <path-1>
- <path-2>
- ...

Instructions:

1. Run: mkdir -p <course-root>/.course-cram/ingests

2. Read every source file in full using the Read tool.
   - For PDFs longer than 10 pages, use the `pages` argument and page through explicitly:
     pages: "1-10", then pages: "11-20", and so on until you reach the end.
     Do NOT stop before the final page.

3. For each page, transcribe every element to dense markdown:
   - **Text**: verbatim, preserving headings, bullets, and numbering
   - **Equations** (including equations rendered as images): transcribe as LaTeX or plain math.
     Use the EXACT original variable names and structure — do not paraphrase, rename, or simplify.
     Example: if the source shows `X = (A/V)·Ra + (B/V)·Rb·(1−T)`, write exactly that — same variable names, same structure.
   - **Charts / graphs / plots**: describe in full — axes (labels, units, scale), curve/bar shapes,
     intercepts, peaks/troughs, every annotation, and the takeaway the chart communicates.
     Write as if explaining to someone who cannot see the image.
     Example: "payoff diagram: x-axis = stock price (S), y-axis = profit; kink at strike K=50;
     slope 0 for S<K, slope 1 for S>K; y-intercept = −premium"
   - **Diagrams** (timelines, decision trees, flowcharts, capital structure, org charts):
     reconstruct in ASCII or text — every node, edge, label, arrow direction, and numerical annotation.
   - **Tables** (including tables rendered as images): transcribe as markdown tables.
     Every header, every row label, every cell value. Do not drop any row or column.
   - **Screenshots** (Excel, software output, UI): transcribe the content (numbers, formulas,
     labels, text). Ignore pure chrome (window decorations, toolbars).
   - **Handwritten annotations / margin notes**: transcribe verbatim if legible.
     Mark as `[illegible]` if not.
   - **Decorative images** (logos, stock photos, backgrounds): note in one line — no description needed.

4. Organize the transcription as one markdown file:
   - Top header: `# <topic label> — Course Cram Ingest`
   - For each source file, a section: `## Source: <filename>`
   - Within each source file, page boundaries: `### Page <N>`
   - Use dense notation: prefer tables over prose, bullet lists over paragraphs.

5. Write the complete transcription to `<course-root>/.course-cram/ingests/<slug>.md` using the Write tool.
   A single Write call for the entire file. Do NOT write partial content.

**CRITICAL — NO INFORMATION LOSS**: The output must be a complete transcription with NO information loss. Rephrasing for dense formatting is permitted; summarization, paraphrase, or omission is not. If a page contains 30 items, your transcription lists 30 items. If a chart has 5 annotations, all 5 appear. Every equation transcribed with original variable names and structure. Every diagram reconstructed with every node, edge, label, and arrow. Every table transcribed cell-by-cell with all headers and row labels intact.

Return ONLY this manifest (no transcription content):
- ingest_path: <course-root>/.course-cram/ingests/<slug>.md
- source_files: <list>
- total_pages: <N>
- flags: <list of [illegible]/[ambiguous] items, or "none">
```

---

### Template 2 — Teach

**Subagent type**: Explore

```
Task: Read the following course ingest file(s) and generate a teaching response.

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug-1>.md
- <course-root>/.course-cram/ingests/<slug-2>.md  (if multiple topics)

Topic(s): <topic label(s)>
Detail level: <Recap | Review | Full re-teach | Subtopic deep-dive>
Word budget: <500 | 1500 | 3500 | 2000>

Instructions:

1. Read the ingest file(s) in full using the Read tool.

2. Generate teaching content according to the detail level:
   - **Recap**: high-level summary, 5-10 key takeaways only
   - **Review**: key concepts, all important formulas, 1-2 representative worked examples
   - **Full re-teach**: first principles, derivations, all formulas with derivation where present,
     multiple worked examples covering different question types, common pitfalls
   - **Subtopic deep-dive**: focus only on `<specific subtopic>`, cover it exhaustively

3. Pull content **exclusively** from the ingest files. Do not invent material.
   - If a topic has no coverage in the ingest, return only: `[SCOPE GAP: <topic>]`
   - Citation format: `<source-filename>, page <N>` using `### Page <N>` markers in the ingest

4. For worked examples: prefer examples from tutorial-solution sections in the ingest.
   Cite their source file and question number.

5. Stay within the word budget. If Full re-teach would exceed 3500 words, end the first part
   at a natural stopping point and close with: `— continue? (reply yes for next part)`

Return the teaching content as your response. No preamble, no meta-commentary.
```

---

### Template 3 — Quiz generation

**Subagent type**: Explore

```
Task: Read the following course ingest file(s) and generate a short quiz.

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug-1>.md

Topic(s): <topic label(s)>

Instructions:

1. Read the ingest file(s) in full using the Read tool.

2. Generate 5-8 questions covering the topic(s). Mix:
   - Conceptual short-answer (test understanding of definitions, relationships, conditions)
   - Numerical (calculations using formulas from the ingest, with given data)
   - Formula application / interpretation (apply or explain a formula's components)

3. Present all questions in this exact format:
   - First line: `**<topic(s)> Quiz — Total: <M> marks**`
   - Each question: `<N>. <question text> [<marks> marks]`
   - No preamble. No commentary. No hints. No answers.

4. Assign marks per question: typically 2-4 marks for conceptual, 4-6 for numerical.

Return only the formatted quiz. Nothing else.
```

---

### Template 4 — Quiz marking

**Subagent type**: Explore

```
Task: Read the following course ingest file(s) and mark a quiz attempt.

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug-1>.md

Questions:
<paste the quiz questions here>

User's answers:
<paste the user's answers here>

Instructions:

1. Read the ingest file(s) in full using the Read tool.

2. For each question, produce an answer key entry:
   (a) The correct answer
   (b) A 1-sentence justification citing `<source-filename>, page <N>` from the ingest
   (c) The marking point (what earns the mark)

3. Mark the user's answer against the key:
   - Full marks: correct and complete
   - Partial marks: partially correct (note what was missing)
   - Zero: incorrect or missing

4. Format:
   - `**Q<N>. [<awarded>/<total> marks]**`
   - Answer: <correct answer>
   - Source: <filename>, page <N>
   - User's answer: <brief description of what user wrote>
   - Budget: ≤ 80 words per question entry.

5. Final line: `**Score: <X>/<total> marks**`

Return the answer key + marked attempt. No preamble.
```

---

### Template 5 — Format inference (exam phase step 1)

**Subagent type**: Explore

```
Task: Read the following past-exam ingest file(s) and return a complete structured format spec.

Ingest files to read:
- <course-root>/.course-cram/ingests/past-papers-<type>.md

Instructions:

1. Read the ingest file(s) in full using the Read tool.

2. Return a complete structured format spec. Include ALL of the following:
   - Total question count (per paper, per section)
   - Type breakdown: MCQ count, short-answer count, long-answer count, numerical count, other types
   - Marks per question: list every value observed; note if consistent or variable
   - Paper total marks
   - Time limit — quote verbatim if stated
   - Section structure: every section label, question range per section, marks per section
   - Representative question phrasings: transcribe at least 2-3 questions verbatim per type
     (more if fewer types); preserve the exact wording, data values, and formatting
   - Cover-page instructions: transcribe verbatim
   - Allowed materials (formula sheet, calculator, open book, etc.): transcribe verbatim
   - Any rubric or marking notes: transcribe verbatim
   - Observed difficulty patterns (note with examples if present)

3. Format compactly: markdown tables for structured data, dense bullets for lists.

**CRITICAL — NO INFORMATION LOSS**: The output must be a complete transcription with NO information loss. Rephrasing for dense formatting is permitted; summarization, paraphrase, or omission is not. If a page contains 30 items, your transcription lists 30 items. If a chart has 5 annotations, all 5 appear. Every equation transcribed with original variable names and structure. Every diagram reconstructed with every node, edge, label, and arrow. Every table transcribed cell-by-cell with all headers and row labels intact.

Return the format spec only. No commentary, no analysis.
```

---

### Template 6 — Exam generation

**Subagent type**: Explore

```
Task: Read the following course ingest files and generate a mock exam paper.

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug-1>.md
- <course-root>/.course-cram/ingests/<slug-2>.md
- ...

Exam type: <Midterm | Final | <custom>>
Scope: <topic list or "all topics above">

Format spec (from format inference):
<paste the format spec here>

Instructions:

1. Read all ingest files in full using the Read tool.

2. Generate the exam paper using this mix strategy:
   - ~30% **anchor questions**: lift questions verbatim from past-paper ingest content.
     Label each anchor: `[<paper name>, Q<original question number>]`
     Choose anchors that are representative of the format and difficulty.
   - ~70% **new questions**: generate fresh questions covering the full scope.
     Match the inferred format, difficulty level, and marks distribution exactly.
     Draw content from the ingest files — do not invent facts or formulas.

3. Present as a single structured paper:
   - Title header with exam type and simulated date
   - Instructions section (match the format spec's cover-page instructions)
   - Section breakdown as specified in the format spec
   - Questions numbered, marks bracketed at end of each question
   - Simulated time limit
   - No answers. No answer spaces. Paper format only.

4. No preamble before the paper title. No commentary after.

Return the exam paper only.
```

---

### Template 7 — Exam marking

**Subagent type**: Explore

```
Task: Read the following course ingest files and mark an exam attempt.

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug-1>.md
- <course-root>/.course-cram/ingests/<slug-2>.md
- ...

Exam paper:
<paste the full exam paper here>

User's answers:
<paste the user's answers here>

Format spec:
<paste the format spec here>

Instructions:

1. Read all ingest files in full using the Read tool.

2. For each question:
   (a) Provide the full worked solution, drawn from the ingest files.
       Cite `<source-filename>, page <N>` for key facts and formulas.
   (b) Mark the user's attempt: full / partial / zero marks (explain why for partial/zero).
   Budget: ≤ 150 words per long-answer/numerical question, ≤ 60 words per MCQ.

3. Performance summary: list every topic covered in the exam, score in that topic.

4. Identify 2-3 topics most worth re-teaching based on the user's performance.
   For each: name the topic, note the specific gaps observed.

Format:
- `**Q<N>. [<awarded>/<total> marks]**` followed by worked solution and marking.
- `**Performance by topic:**` table at the end.
- `**Re-teach recommendations:**` bullet list.

Return the marked solutions + performance summary. No preamble.
```

---

---

### Template 8 — Cheatsheet spec inference

**Subagent type**: Explore

```
Task: Read the following past-exam ingest file(s) and extract any rules governing student cheat sheets or formula sheets.

Ingest files to read:
- <course-root>/.course-cram/ingests/past-papers-midterm.md   (if exists)
- <course-root>/.course-cram/ingests/past-papers-final.md     (if exists)

Instructions:

1. Read the ingest file(s) in full using the Read tool.

2. Search for any rules about permitted materials, cheat sheets, formula sheets, or help sheets.
   Look for phrases like: "cheat sheet", "crib sheet", "formula sheet", "help sheet",
   "allowed materials", "open book", "permitted aids", "one A4 sheet", "double-sided",
   "font size", "typeset", "handwritten", "no lecture slides".

3. For every rule found, transcribe it VERBATIM and cite the source:
   - Verbatim quote of the rule
   - Source: <paper-ref>, page <N>

4. From the observed rules, derive a `recommended_defaults` block with concrete values
   satisfying every constraint:
   - paper_size: A4 / Letter
   - sides: 1 / 2
   - sheet_count: <N>
   - font_size_floor: <N>pt or null
   - typeset_allowed: true / false
   - other_restrictions: <list>

5. If no rules are found in any ingest file, return exactly: [NO CONSTRAINTS OBSERVED]

**CRITICAL — VERBATIM ONLY**: Every rule in the output must be a direct quote from the ingest.
Do not paraphrase, infer, or add rules not present in the source.

Return only the observed_constraints block and recommended_defaults block. No preamble.
```

---

### Template 8.5 — Cheatsheet page-fit estimator + font auto-tune

**Subagent type**: Explore

```
Task: Estimate the content volume for a cheatsheet and determine the optimal font size within a given range.

Ingest files to read (structural scan only — do NOT read full body text):
<list of in-scope slug paths>

Cheatsheet spec:
- Paper: <A4 | Letter>
- Orientation: landscape
- Sides: <1 | 2>
- Sheet count: <N>
- Columns per side: <cols>
- Font size range: [<min>pt, <max>pt]
- Detail level: <Skeleton | Standard | Detailed>

Instructions:

1. For each ingest file, perform a STRUCTURAL READ ONLY:
   - Count top-level sections (## headings)
   - Count subsections (### headings)
   - Count formula blocks (lines containing \[ \] or $$ or \begin{equation})
   - Sample 5 random content blocks and measure their character length
   - Compute estimated_total_chars for this detail level:
     * Skeleton: formulas + definitions only → estimated chars ≈ formula_count × 120 + section_count × 80
     * Standard: above + 1-line notes → ≈ formula_count × 200 + section_count × 120
     * Detailed: above + mini-examples → ≈ formula_count × 300 + section_count × 200
   Sum across all ingest files for total_estimated_chars.

2. Use the following calibration table (chars per column, landscape A4 at 0.3in margins):
   | font_pt | chars_per_col (2-col) | chars_per_col (3-col) | chars_per_col (4-col) |
   |---------|----------------------|----------------------|----------------------|
   |   6.0   |        4800          |        3000          |        2200          |
   |   7.0   |        3600          |        2200          |        1650          |
   |   8.0   |        2900          |        1750          |        1300          |
   |   9.0   |        2300          |        1400          |        1050          |
   |  10.0   |        1900          |        1150          |         860          |
   For Letter paper multiply by 0.92. For portrait orientation multiply by 0.65.
   total_capacity(font_pt) = chars_per_col(font_pt, cols) × cols × sides × sheet_count

3. For each candidate font size in [min, max] at 1pt granularity (and optionally 0.5pt if range ≤ 2pt):
   - overflow_pct = max(0, (total_estimated_chars - total_capacity) / total_capacity × 100)
   - If overflow_pct == 0:
       fill_pct = total_estimated_chars / total_capacity × 100
       last_col_fill = fill_pct % (100 / total_columns)   (fill of final column)
       NOTE: last_col_fill must be ≥ 83.3% (5/6) to qualify
   - Record: fits (overflow_pct == 0), last_col_fill, qualifies (fits AND last_col_fill ≥ 83.3%)

4. Selection rule: recommended_font = largest qualifying font size.
   If no size qualifies, identify:
   - largest_fitting: largest size where overflow_pct == 0 (even if last_col_fill < 83.3%)
   - smallest_overflowing: smallest size where overflow_pct > 0

5. Return:

Font sweep (<paper>, landscape, <sides>-sided × <sheet_count> sheets, <cols> col/side, <detail>):
  <font>pt — capacity <cap>, content ~<est> → <fits|overflow ~X%>, last-col fill ~<Y>% [✓|✗ (<reason>)]
  ...
Recommended: <font>pt  (<reason: fits AND last-col fill ≥ 5/6>)
OR
No qualifying size found in [<min>, <max>]:
  Largest fitting: <font>pt (last-col fill ~<Y>%, under 5/6)
  Smallest overflowing: <font>pt (overflow ~<X>%)

Return only the sweep table and recommendation. No preamble.
```

---

### Template 9 — Cheatsheet generation

**Subagent type**: general-purpose (needs Read + Write + Bash)

```
Task: Read the following course ingest files and write a complete LaTeX cheatsheet to disk.

Target file: <course-root>/.course-cram/cheatsheets/<filename>.tex

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug-1>.md
- <course-root>/.course-cram/ingests/<slug-2>.md
- ...

Cheatsheet spec (FINAL — do not deviate):
- Paper: <A4 | Letter>
- Orientation: landscape
- Sides: <1 | 2>
- Sheet count: <N>
- Columns per side: <cols>
- Font size: <chosen_pt>pt  ← locked by auto-tune; use exactly this value
- Detail level: <Skeleton | Standard | Detailed>
- Section ordering: <by-lecture | custom list>
- Geometry margins: 0.3in all sides

LaTeX preamble to use (fill in font and columns from spec):

\documentclass[<chosen_pt>pt]{extarticle}
\usepackage[landscape,margin=0.3in]{geometry}
\usepackage{multicol}
\usepackage{amsmath,amssymb}
\usepackage{enumitem}
\setlength{\columnseprule}{0.4pt}
\setlength{\parindent}{0pt}
\setlength{\parskip}{1pt}
\setlist{noitemsep,topsep=0pt,parsep=0pt,partopsep=0pt}
\begin{document}
\begin{multicols}{<cols>}

[CONTENT HERE]

\end{multicols}
\end{document}

Instructions:

1. Run: mkdir -p <course-root>/.course-cram/cheatsheets

2. Read every ingest file in full using the Read tool.

3. For each topic in the specified section order, generate cheatsheet content at the chosen detail level:
   - Skeleton: formulas (as \[ ... \] or inline $...$) and definitions only; no prose paragraphs.
   - Standard: formulas, definitions, 1-line worked steps, one-line "when to use" per technique.
   - Detailed: Standard plus boxed mini-examples drawn from tutorial-solution sections of the ingest.

4. **PER-ITEM PROVENANCE (MANDATORY)**: Above every formula, definition, theorem, algorithm step,
   and worked-example fragment, write a LaTeX comment:
     % src: <slug>.md p<N>
   where <N> is the SINGLE page in the ingest where this item appears.
   Use % src: <slug>.md p<A-B> ONLY when the item genuinely spans two consecutive pages.
   Also write % section-src: <slug>.md at the start of each \section* or \subsection*.
   These comments are machine-readable provenance — never omit them.

5. **CRITICAL — NO ADDITION OF CONTENT**: Every formula, definition, theorem, algorithm,
   and worked-example fragment in the output must be present in one of the listed ingest files.
   No paraphrase, no synthesised "obvious" content, no general-knowledge supplements.
   If something feels missing, record [SCOPE GAP: <topic>] in the manifest — do not fill it in.
   Variable names, equation structure, and notation must match the ingest exactly.

6. If content volume exceeds the estimated column budget for the chosen font size, trim in this order:
   (1) full prose paragraphs first
   (2) verbal restatements of formulas
   (3) derivations
   (4) secondary worked examples
   Never drop a formula or primary definition to fit. Record any dropped section in the manifest.

7. Count total emitted content blocks (formulas, definitions, algorithm steps, worked examples).
   Count those with a preceding % src: comment within 3 lines above.
   Compute provenance_coverage_pct = (blocks_with_src / total_blocks) × 100.
   If < 100, add missing % src: comments before writing. Do not write the file with < 100% coverage.

8. Write the complete .tex file using the Write tool. Single Write call for the entire file.

Return ONLY this manifest (no .tex content):
- tex_path: <course-root>/.course-cram/cheatsheets/<filename>.tex
- sections_written: <list of section headings>
- sections_dropped_for_fit: <list or "none">
- scope_gaps: <list of [SCOPE GAP: <topic>] items or "none">
- provenance_coverage_pct: <integer 0–100>
```

---

### Template 10 — Cheatsheet revision (single section)

**Subagent type**: general-purpose (needs Read + Write)

```
Task: Revise a single section of an existing LaTeX cheatsheet.

Cheatsheet file: <course-root>/.course-cram/cheatsheets/<filename>.tex
Section to revise: <section heading>

Ingest files to read:
- <course-root>/.course-cram/ingests/<slug>.md  (slug for this section)

Discrepancy context (from Verify, if applicable):
<paste discrepancy block or "none">

Instructions:

1. Read the existing .tex file and the relevant ingest file(s) in full.

2. Locate the section bounded by \section*{<heading>} ... next \section* or \end{multicols}.

3. Re-generate ONLY that section's content following the same spec (detail level, font, columns)
   as the rest of the file. Apply the no-addition-of-content rule: every item must be in the ingest.
   If the discrepancy context lists MISMATCHes, fix each one to match the ingest exactly.

4. Update % src: provenance comments for every item in the revised section.

5. Write the updated .tex file (full file with the section replaced).

Return:
- revised_section: <section heading>
- items_fixed: <list from discrepancy context, or "none">
- new_scope_gaps: <list or "none">
- provenance_coverage_pct: <integer>
```

---

### Template 11 — Cheatsheet compile-fix

**Subagent type**: Explore

```
Task: Diagnose a pdflatex error in a cheatsheet and propose a minimal fix.

Error block:
<paste the pdflatex error output here>

Cheatsheet file: <course-root>/.course-cram/cheatsheets/<filename>.tex (read this file)

Ingest files potentially relevant (read ONLY if needed to verify a formula):
<list if known>

Instructions:

1. Read the .tex file.

2. Locate the line(s) causing the pdflatex error (use the error line number).

3. Propose the minimal LaTeX-syntax fix that resolves the error.
   - Fix LaTeX syntax errors only (unclosed braces, undefined commands, missing \end, etc.)
   - Do NOT add, remove, or rephrase any mathematical content.
   - Do NOT remove % src: provenance comments.
   - If fixing requires understanding a formula's structure, read the cited ingest page
     (use % src: comment to find it), then fix the LaTeX to render it correctly.

4. Return the proposed fix as a unified diff:
   - Context: 3 lines before and after the change
   - Label: file path and line numbers

Return the diff only. No preamble. No content additions.
```

---

### Template 12 — Cheatsheet verify

**Subagent type**: Explore

```
Task: Verify a set of cheatsheet content blocks against specific pages of a course ingest file.

Ingest file: <course-root>/.course-cram/ingests/<slug>.md

Items to verify (from % src: comments in the cheatsheet):
<table of (page_range, content_block) pairs — one row per item>
| Page range | Content block (verbatim from .tex, excluding % comments) |
|------------|----------------------------------------------------------|
| p<N>       | <content>                                                |
| p<A-B>     | <content>                                                |
...

Instructions:

1. For each unique page range in the table above, read ONLY those pages from the ingest
   using the Read tool with the `offset` and `limit` arguments to reach the ### Page <N> markers.
   Do NOT read pages not referenced in the table.
   Do NOT read the full ingest file.

2. For each content block, classify it as one of:
   - MATCH: the block's mathematical content (formulas, variable names, structure) is present
     on the cited page(s) and matches the ingest. Minor LaTeX formatting differences are OK.
   - MISMATCH: the content is on the cited page but differs in a meaningful way (wrong variable
     name, wrong coefficient, wrong structure). Quote both the ingest version and the .tex version
     side-by-side.
   - MISSING: the content is not found anywhere on the cited page(s). It may exist elsewhere in
     the ingest (do not search) — just report MISSING with the cited range.
   - OUT_OF_RANGE: the cited page range has no ### Page <N> section in the ingest, or the section
     is empty. Do not invent a verdict.

3. Format:
   | Page range | Status | Notes |
   |------------|--------|-------|
   | p<N>       | MATCH  |       |
   | p<N>       | MISMATCH | Ingest: `<excerpt>` / .tex: `<excerpt>` |
   ...
   Summary: <matched>/<total> MATCH. Non-MATCH items: <list with one-line action per item>.

Return the verification table and summary only. No preamble.
```

---

## Parallel dispatch (map phase)

To launch ingest subagents in parallel, issue multiple Agent tool calls in a **single message**. Each subagent runs in its own context and returns independently. The main agent waits for all results before writing `INDEX.md`.

Example — two topics in parallel:
```
[Agent call 1: week-01 ingest]
[Agent call 2: week-02 ingest]
```

Both calls in one message → parallel execution.

Grouping heuristic:
- ≤ ~60 pages total: one subagent for everything
- 60–300 pages: one subagent per category (lecture group, tutorial group, past-papers)
- > 300 pages: one subagent per unit (~100 pages max each), parallel batches

---

## INDEX.md schema

Path: `<course-root>/.course-cram/ingests/INDEX.md`

```markdown
# Course Cram Ingest Index

Course: <course name>
Root: <absolute path>
Updated: <ISO timestamp>

| Topic | Slug | Ingest file | Source files | Pages | Last updated | Flags |
|---|---|---|---|---|---|---|
| <label> | <slug> | <slug>.md | <comma-separated filenames> | <N> | <ISO date> | <flags or —> |
```

Main agent writes this after every ingest or update. It is the only ingest-related file the main agent writes directly.

---

## session.md schema

Path: `<course-root>/.course-cram/session.md`

Append-only. Multiple sessions stack; most recent last.

```markdown
# Course Cram — Session Log

Course: <name>
Root: <absolute path>

<!-- Entries append below, most recent last -->

---

## Session <ISO timestamp>

### Map
- Scope: <categories>, <units>
- Topics: <N> (see INDEX.md for full list)
- Last subagent dispatch: <ISO>, <topic slugs updated, or "full re-ingest">

### Teach log
- <timestamp>: <topic> — <level>

### Quiz log
- <timestamp>: <topic> — <score>/<total>. Weak: <list>

### Exam log
- <timestamp>: <type> — <score>/<total>. Weak topics: <list>

### Cumulative weak areas
- <topic> (from <exam ref>, recurrence count: <N>)
```

This file contains **state only** — no source content, no transcription, no generated questions or answers. If the user wants to preserve generated content (exam papers, answer keys), they copy manually.

---

---

## `cheatsheets/INDEX.md` schema

Path: `<course-root>/.course-cram/cheatsheets/INDEX.md`

Main agent writes/updates this after every successful cheatsheet generation or revision.

```markdown
# Course Cram Cheatsheet Index

Course: <course name>
Root: <absolute path>
Updated: <ISO timestamp>

| Name | tex_path | Scope | Paper | Sides | Cols | Font range | Font chosen | Detail | Scope gaps | Last-col fill | Last updated |
|---|---|---|---|---|---|---|---|---|---|---|---|
| <filename>.tex | <abs path> | <slug list> | A4 landscape | <N>-sided × <M> | <cols> | [<min>, <max>]pt | <chosen>pt | <tier> | <gaps or —> | ~<pct>% | <ISO date> |
```

Each row stores both `font_range` (original user-supplied range) and `font_chosen` (auto-tuned result), so a later Regenerate can re-tune if scope changes.

---

## Output budgets (summary)

| Output | Cap |
|---|---|
| Scope-confirmation (pre-map) | ≤ 150 words |
| Map manifest (post-map prompt) | ≤ 400 words |
| Teach — Recap | ≤ 500 words |
| Teach — Review | ≤ 1500 words |
| Teach — Full re-teach | ≤ 3500 words (split if longer) |
| Teach — Subtopic deep-dive | ≤ 2000 words |
| Quiz presentation | numbered Qs only, no preamble |
| Quiz answer key | ≤ 80 words per question |
| Exam presentation | paper format only, no preamble |
| Exam answer key | ≤ 150 words per long Q, ≤ 60 words per MCQ |
| Post-step menus | exact numbered lists, no elaboration |
| Cheatsheet spec form | ≤ 250 words |
| Cheatsheet fit sweep table | ≤ 200 words |
| Cheatsheet generation manifest | ≤ 200 words |
| Cheatsheet verify summary | ≤ 300 words |
| Cheatsheet compile report | ≤ 150 words |
| Cheatsheet section-revise manifest | ≤ 400 words |

Budgets constrain the subagent's rendered return, not the ingest. The map transcription on disk stays complete regardless of budget. Use `clarify Q<N>` in the post-step menu to get deeper treatment of a specific answer without blowing the default budget.
