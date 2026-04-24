# course-cram — Context Management Reference

Read this file before dispatching any subagent. The prompt templates below include the no-loss clause and return contracts that must be used as specified.

---

## Why subagents own ingests

Visual PDF rendering costs ~1000-1500 tokens per page in the main conversation. A subagent reading the same page and writing a dense markdown transcription to disk returns only a short manifest (~50 tokens) to main context. Running all source reads through subagents:

- Keeps the main context footprint to: `INDEX.md` metadata + session state + current rendered response
- Makes ingests persistent — they survive across sessions; re-map is never needed unless materials change
- Enables per-topic granular updates — re-transcribe one topic without touching others

**Trade-off**: each teach/quiz/exam turn dispatches a subagent (adds latency). This is the correct trade-off: compaction-free sessions and cross-session persistence outweigh per-turn overhead.

---

## Directory layout

```
<course-root>/.course-cram/
├── ingests/
│   ├── INDEX.md              ← metadata only; main-agent-readable
│   ├── <topic-slug>.md       ← per-topic transcription; subagent-only
│   └── ...
└── session.md                ← session state; main-agent-readable
```

Main agent may read: `INDEX.md`, `session.md`, and the output of `ls ingests/`.
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

Budgets constrain the subagent's rendered return, not the ingest. The map transcription on disk stays complete regardless of budget. Use `clarify Q<N>` in the post-step menu to get deeper treatment of a specific answer without blowing the default budget.
