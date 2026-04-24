---
name: course-cram
description: Structured four-phase exam-prep pipeline for any course (map → teach → quiz → exam). Use when the user asks to study, cram, prep, or review for a test/midterm/final, or wants a guided walkthrough of course materials. Transcribes all course files to per-topic ingest files on disk via subagents, teaches at user-chosen depth, generates topic-scoped quizzes, and produces comprehensive mock exams that mirror past-paper formats. Every phase transition is user-driven via explicit menu prompts — never auto-advances.
---

# Course Cram

Guide the user through four phases — **map** → **teach** → **quiz** → **exam** — with explicit transition prompts between every phase. The user controls every transition: post the prompt, wait, act on the choice. Never advance phases on your own.

<HARD-GATE>
- Never read ingest files in main context. (`<course-root>/.course-cram/ingests/*.md` except `INDEX.md` are subagent-only paths.) All reads/writes of ingest content are performed by subagents. Main agent may only `ls` the ingests directory and read `INDEX.md` and `session.md`.
- Never ingest source materials (lecture PDFs, tutorial PDFs, past-paper PDFs, etc.) in main context. Source reading is done exclusively by subagents, which write transcription directly to `<course-root>/.course-cram/ingests/<topic>.md`.
- Never drop information during ingest. Ingest-writing subagent prompts include the literal "no information loss" clause from `CONTEXT.md`; subagent-written transcription must be complete. Rephrasing for density (tables, dense bullets, verbatim markdown) is permitted; summarization, paraphrase, or omission is not.
- Never write to disk outside `<course-root>/.course-cram/`. Source materials are read-only.
- Never invent content beyond what the ingests support. If a requested topic has no ingest, extend the map first — do not guess.
- Never reveal quiz or exam answers until the user explicitly requests them in the post-step prompt.
- Never skip a phase or chain phases automatically. Each transition requires an explicit user selection from the posted menu.
- Never reuse the user's past answers as "correct" — answer keys derive from the ingests (source materials), not from the user's attempts.
</HARD-GATE>

## Checklist

Use TodoWrite to track one task per phase invocation. Complete in order:

1. **Pre-entry check** — check for `<cwd>/.course-cram/session.md`; offer resume if found
2. **Entry prompt** — ask which phase to start from
3. **Execute selected phase** (map / teach / quiz / exam)
4. **Post-phase transition prompt** — user picks next action
5. **Loop to step 3** (execute next phase) or exit when user says done

## Process Flow

```dot
digraph course_cram {
    entry      [shape=circle,       label="Entry"];
    map        [shape=box,          label="Map\n(check disk; ingest or reuse)"];
    teach      [shape=box,          label="Teach\n(subagent reads topic ingest)"];
    quiz       [shape=box,          label="Quiz\n(subagent reads topic ingest)"];
    exam       [shape=box,          label="Exam\n(subagent reads all ingests)"];
    post_map   [shape=diamond,      label="After map:\nrepeat/update/add/remove/proceed?"];
    post_teach [shape=diamond,      label="After teach:\nrepeat/clarify/proceed?"];
    post_quiz  [shape=diamond,      label="After quiz:\nrepeat/clarify/answer/proceed?"];
    post_exam  [shape=diamond,      label="After exam:\nrepeat/clarify/answer/proceed?"];
    done       [shape=doublecircle, label="Done"];

    entry -> map   [label="start=map"];
    entry -> teach [label="start=teach"];
    entry -> quiz  [label="start=quiz"];
    entry -> exam  [label="start=exam"];

    map   -> post_map;
    teach -> post_teach;
    quiz  -> post_quiz;
    exam  -> post_exam;

    post_map -> map   [label="repeat/update/add/remove"];
    post_map -> teach [label="proceed teach"];
    post_map -> quiz  [label="proceed quiz"];
    post_map -> exam  [label="proceed exam"];
    post_map -> done  [label="done"];

    post_teach -> teach [label="repeat/clarify"];
    post_teach -> quiz  [label="proceed quiz"];
    post_teach -> exam  [label="proceed exam"];
    post_teach -> map   [label="remap"];
    post_teach -> done  [label="done"];

    post_quiz -> quiz  [label="repeat/clarify/answer"];
    post_quiz -> teach [label="back to teach"];
    post_quiz -> exam  [label="proceed exam"];
    post_quiz -> done  [label="done"];

    post_exam -> exam  [label="repeat/clarify/answer"];
    post_exam -> teach [label="back to teach"];
    post_exam -> quiz  [label="back to quiz"];
    post_exam -> done  [label="done"];
}
```

## Context Management

Course materials at full fidelity project to hundreds of thousands of tokens — too much for the main conversation. This skill solves that with **persistent, subagent-owned, topic-split ingests on disk**.

### 1. Ingests live at `<course-root>/.course-cram/ingests/`

- One `.md` file per topic (e.g. `week-03-portfolio-theory.md`), written by a subagent during Map.
- `INDEX.md` inside the same directory holds topic-level metadata only (source files, page counts, timestamps, flags). The main agent may read `INDEX.md` — it is metadata, not transcription.
- Source PDFs and text files are read **only** by subagents. Main agent never opens them.

### 2. Main agent never reads ingest content

Every content-bearing operation dispatches a subagent that reads the relevant ingest file(s), generates the output within budget, and returns only the rendered response. Main agent footprint stays: `INDEX.md` summary + session state + current rendered response.

| Operation | Subagent type | What subagent does |
|---|---|---|
| Map — ingest write | general-purpose | reads source files, writes `<topic>.md`, returns manifest only |
| Teach | Explore | reads `<topic>.md`, returns rendered teach content |
| Quiz generation | Explore | reads `<topic>.md`, returns questions only |
| Quiz marking | Explore | reads `<topic>.md` + receives user answers, returns marked key |
| Exam format inference | Explore | reads past-paper ingest(s), returns format spec |
| Exam generation | Explore | reads ingest(s) + spec, returns exam paper |
| Exam marking | Explore | reads ingest(s) + receives user answers, returns worked solutions |

### 3. Topic split

Map groups source files by topic (default: one topic per unit). One general-purpose subagent per topic writes one `.md` file. Updates are per-topic — re-transcribe one topic without touching others.

### 4. Output budgets (display-only; ingest stays complete on disk)

| Output | Cap |
|---|---|
| Scope-confirmation (pre-map) | ≤ 150 words |
| Map manifest (post-map prompt) | ≤ 400 words, strict template |
| Teach — Recap | ≤ 500 words |
| Teach — Review | ≤ 1500 words |
| Teach — Full re-teach | ≤ 3500 words (subagent splits; asks "continue?" if longer) |
| Teach — Subtopic deep-dive | ≤ 2000 words |
| Quiz presentation | numbered Qs only, no preamble |
| Quiz answer key | ≤ 80 words per question |
| Exam presentation | paper format only, no preamble |
| Exam answer key | ≤ 150 words per long Q, ≤ 60 words per MCQ |
| Post-step menus | exact numbered lists, no elaboration |

Budgets constrain the subagent's returned rendered response, not the ingest. The ingest on disk is always the complete transcription.

### Sidecar reference

**Read `CONTEXT.md` (in this skill directory) before dispatching any subagent.** It holds exact prompt templates (including the no-loss clause wording that must appear verbatim in ingest-write prompts), `INDEX.md` and `session.md` schemas, topic-grouping heuristics, and the output-budget table.

## Entry

### Pre-entry check

On invocation, run `ls <cwd>/.course-cram/session.md 2>/dev/null` via Bash. If present, read it and offer:

> **Previous session found** (updated `<timestamp>`). Last phase: `<phase>`. Topics mapped: `<N>`. Weak areas: `<list or "none">`.
> 1. **Resume** — jump to the phase you left off (ingests already on disk; no re-map needed)
> 2. **Start fresh** — ignore the snapshot, run normal Entry prompt
> 3. **View snapshot** — display `session.md`, then re-prompt

If no `session.md`, post the normal Entry prompt.

### Normal Entry prompt

> **Course Cram ready.** Where would you like to start?
> 1. **Map** — check/build topic ingests on disk
> 2. **Teach** — explain topic(s) (requires prior map or explicit scope)
> 3. **Quiz** — short scoped quiz (requires prior map or explicit scope)
> 4. **Exam** — comprehensive mock (requires prior map or explicit scope)
>
> Reply with a number or phase name.

If the user picks Teach (2), Quiz (3), or Exam (4):

1. Run `ls <cwd>/.course-cram/ingests/INDEX.md 2>/dev/null` via Bash.
2. **INDEX.md found** — read it (metadata only). Proceed directly to the selected phase using the mapped topics as the default scope. Do not ask about the map.
3. **INDEX.md not found** — post:

   > No map found at `<cwd>/.course-cram/ingests/`. The Map phase must be completed before topics can be loaded.
   > 1. **Run Map now** — go to Phase 1 (Map)
   > 2. **Provide explicit scope** — describe your course materials here; I will teach/quiz/examine based on what you provide without persisted ingests
   > 3. **Cancel** — return to Entry

## Phase 1 — Map

### Step 0 — Disk check

Run `ls <course-root>/.course-cram/ingests/ 2>/dev/null` via Bash. If the user hasn't specified a course-root, offer the current working directory as default.

#### Branch A — No ingests found (dir missing or empty)

Post:

> No ingest found at `<course-root>/.course-cram/ingests/`. Begin ingestion?
> 1. **Begin ingest** — specify scope, dispatch subagents to transcribe
> 2. **Change root** — specify a different course-root path
> 3. **Cancel** — return to Entry

On "Begin ingest": proceed to Step 1.

#### Branch B — Ingests found

Read `<course-root>/.course-cram/ingests/INDEX.md` (main-agent-safe; metadata only). Post:

> Found `<N>` topic ingests at `<path>` (last updated `<most-recent-date>`):
> - `<topic-label>` (`<source-file-count>` files, `<pages>` pages, updated `<date>`)
> - `<topic-label>` ...
>
> What next?
> 1. **Use existing** — skip ingest; mark map ready and proceed to post-map menu
> 2. **Update one topic** — pick a topic; re-dispatch its ingest subagent
> 3. **Update all** — re-dispatch subagents for every topic
> 4. **Add new topic** — specify new source files + topic label; dispatch one subagent
> 5. **Remove a topic** — pick a topic; delete its `.md` + INDEX entry (no subagent needed)
> 6. **Cancel**

Handle the chosen option, then proceed to Step 3 (post-map manifest).

### Step 1 — Enumerate & propose topic grouping

(Only runs when ingesting or adding topics.)

1. Use Glob/LS to enumerate matched files under the course-root for the user's scope.
2. Run batched `pdfinfo` in a single Bash call for all PDF page counts (metadata only — safe for main context).
3. Classify files by category (from directory names and filename patterns: `lecture slides/`, `tutorial/`, `midterm test/`, `final exam/`, etc.).
4. Propose topic grouping: default is one topic per unit (e.g., one slug per week), inferred from filename patterns (`Week N`, `Tutorial N`, `Lecture N`, `Topic N`). Cross-unit materials get dedicated slugs (`past-papers-midterm`, `past-papers-final`, `admin`).
5. Generate a slug for each topic: lowercase, hyphen-separated, ≤ 40 chars.
6. Post scope + grouping confirmation (≤ 150 words):

> About to ingest `<N>` files (~`<P>` total pages) into `<K>` topic ingests:
> - `<topic-slug-1>.md` ← `<file list>` (`<pages>` pages)
> - `<topic-slug-2>.md` ← ...
>
> Reply **go** to proceed, **regroup** to adjust topics, or **stop** to narrow scope first.

### Step 2 — Dispatch ingest subagents (parallel)

One subagent per topic. **Read `CONTEXT.md` first**, then launch all in a **single message with multiple Agent tool calls** (parallel).

Each subagent is **general-purpose** type (needs Write access). The prompt uses the **Ingest write template** from `CONTEXT.md`. Every ingest-write prompt must include:

- Topic slug and target ingest file path: `<course-root>/.course-cram/ingests/<slug>.md`
- Full list of source files assigned to this topic
- Instruction to create `<course-root>/.course-cram/ingests/` via `mkdir -p` before writing
- Instruction to write the transcription to the target path via the Write tool
- The **literal no-loss clause** copied verbatim from `CONTEXT.md`
- Return contract: manifest only (`{ingest_path, source_files, total_pages, flags}`) — no transcription content in the return message

The subagent reads each assigned source file in full (using Read tool's `pages` argument for PDFs > 10 pages), then writes the complete transcription as a single Write to `<slug>.md`.

### Step 3 — Main agent writes `INDEX.md`

After all subagents return, main agent composes `<course-root>/.course-cram/ingests/INDEX.md` from the returned manifests. Format: see `CONTEXT.md` → "INDEX.md schema".

### Step 4 — Post-map manifest

Post (≤ 400 words, strict template):

> **Map complete.** `<K>` topic ingests at `<course-root>/.course-cram/ingests/`:
> - `<topic-slug-1>.md` — `<source files>` (`<pages>` pages)
> - `<topic-slug-2>.md` — ...
>
> Subagent-flagged content: `<illegible/ambiguous items from manifests, or "none">`.
>
> What next?
> 1. **Repeat** the map (re-dispatch subagents for every topic)
> 2. **Update** scope (re-enter Step 0 Branch B)
> 3. **Add** materials (new topic or new files into an existing topic)
> 4. **Remove** materials (delete a topic ingest)
> 5. **Proceed to Teach**
> 6. **Proceed to Quiz**
> 7. **Proceed to Exam**
> 8. **Done**
> 9. **Snapshot & exit** — append session state to `<course-root>/.course-cram/session.md` and end

Do not add extra commentary, encouragement, or unsolicited suggestions.

## Phase 2 — Teach

### Input (prompt the user for)

- **Target topic(s)** — must match a slug in `INDEX.md` (main agent reads `INDEX.md` to validate and list options). If no prior map, ask the user for scope explicitly.
- **Detail level** (offer as a numbered choice):
  1. **Full re-teach** — from first principles, derivations, worked examples, common pitfalls
  2. **Review** — mid-depth recap with key concepts, formulas, representative examples
  3. **Recap** — high-level summary, 5-10 key takeaways
  4. **Subtopic deep-dive** — user names a specific subtopic; teach only that

### Execute

Read `CONTEXT.md` → "Teach template". Dispatch a **Teach subagent** (Explore type). Prompt includes:

- Path(s) to the relevant `<topic>.md` ingest file(s)
- Detail level + word budget (Recap 500 / Review 1500 / Full 3500 / Subtopic 2000)
- Rules: pull content exclusively from the ingests; cite as `<source-filename>, page <N>` using `## Page <N>` markers in the ingest; use worked examples from tutorial-solution sections in the ingest where relevant; if a topic has no ingest coverage, return `[SCOPE GAP: <topic>]` only — do not invent
- For Full re-teach expected to exceed 3500 words: end the first part at a natural stopping point with `— continue? (reply yes for next part)`; do NOT auto-continue

Main agent posts the returned content verbatim, then posts the post-teach menu.

Remember the **teach scope** — the next quiz defaults to it.

### Exit — Post-Teach Prompt

> **Teach complete on `<topic(s)>` at `<detail level>`.** What next?
> 1. **Repeat** this teach (same scope, same level)
> 2. **Clarify** — you have questions about what was taught
> 3. **Proceed to Quiz** (uses this teach scope)
> 4. **Proceed to Exam**
> 5. **Back to Teach** (different topic or level)
> 6. **Back to Map**
> 7. **Done**
> 8. **Snapshot & exit**

## Phase 3 — Quiz

### Input

Uses the **last teach scope** by default. If no prior teach, prompt the user for scope.

### Execute — Question generation

Read `CONTEXT.md` → "Quiz generation template". Dispatch **Quiz-generation subagent** (Explore type). Prompt includes:

- Path(s) to relevant `<topic>.md` ingest file(s)
- "Generate 5-8 questions covering the scope. Mix: conceptual short-answer, numerical (if the course is quantitative), formula application/interpretation. Present all questions at once: first line `**<topic> Quiz — Total: <M> marks**`, then questions numbered with marks bracketed at end of each. No preamble. No commentary. No answers."

Main agent posts the returned questions verbatim and waits for the user's attempt (or "skip to answers").

### Execute — Marking (on "Show answers")

Read `CONTEXT.md` → "Quiz marking template". Dispatch **Quiz-marking subagent** (Explore type). Prompt includes:

- Path(s) to the same ingest file(s)
- The questions (copied from the generated quiz)
- The user's answers
- "Return a per-question answer key: (a) the answer, (b) a 1-sentence justification citing `<source-filename>, page <N>`, (c) the marking point. Budget: ≤ 80 words per question. Compute and return score `<X>/<total>`."

Main agent posts the returned answer key verbatim.

### Exit — Post-Quiz Prompt

> **Quiz done.** What next?
> 1. **Repeat** quiz (fresh questions, same scope)
> 2. **Clarify** — ask about a specific question
> 3. **Show answers** — reveal answer key with explanations and mark the attempt
> 4. **Proceed to Exam**
> 5. **Back to Teach**
> 6. **Done**
> 7. **Snapshot & exit**

## Phase 4 — Exam

### Input (prompt the user for)

- **Exam type** — Midterm / Final / user-named
- **Scope** — defaults to all topics in `INDEX.md`; user may narrow

### Execute — Step 1: Format inference

Read `CONTEXT.md` → "Format inference template". Dispatch **Format-inference subagent** (Explore type). Prompt includes:

- Path(s) to past-paper ingest file(s) (e.g., `past-papers-midterm.md`, `past-papers-final.md`)
- "Return a complete structured format spec: total question count, type breakdown (MCQ/short/long/numerical counts), marks per question (list every value) and paper total, time limit verbatim, section structure (every section label + question range), representative question phrasings verbatim (≥ 2-3 per type), cover-page instructions verbatim, allowed materials verbatim, rubric text verbatim. Dense tables and bullets. No summarization."
- No-loss clause applies (verbatim elements must appear verbatim)

Main agent records the returned spec in context (bounded; it is format metadata, not transcription).

If no past-paper ingests exist for the requested exam type: prompt user to proceed with generic format or abort.

### Execute — Step 2: Exam generation

Read `CONTEXT.md` → "Exam generation template". Dispatch **Exam-generation subagent** (Explore type). Prompt includes:

- Path(s) to all topic ingest files for the exam scope
- The format spec from Step 1
- Mix strategy: ~30% **anchor questions** — lifted verbatim from past-paper ingest content, each labeled `[<paper-reference>, Q<N>]`; ~70% **new questions** — matching inferred format, difficulty, and marks distribution, covering the full scope
- Output: single structured paper (title header, instructions, section breakdown, numbered questions, marks per question, simulated time limit). No answers. No preamble.

Main agent posts the returned paper verbatim and waits.

### Execute — Step 3: Marking (on "Show answers")

Read `CONTEXT.md` → "Exam marking template". Dispatch **Exam-marking subagent** (Explore type). Prompt includes:

- Path(s) to the topic ingest files used in generation
- The exam paper
- The user's answers
- The format spec
- "Return: full worked solution for every question, mark the user's attempt, performance summary by topic, 2-3 topics most worth re-teaching. Budget: ≤ 150 words per long question, ≤ 60 words per MCQ."

Main agent posts the returned solutions verbatim. If a snapshot is in progress, update the cumulative weak-area list in `session.md`.

### Exit — Post-Exam Prompt

> **Exam submitted.** What next?
> 1. **Repeat** exam (fresh paper, same scope)
> 2. **Clarify** — ask about a specific question
> 3. **Show answers** — full worked solutions, mark the attempt, identify weak areas
> 4. **Back to Teach** (target identified weak areas)
> 5. **Back to Quiz**
> 6. **Back to Map**
> 7. **Done**
> 8. **Snapshot & exit**

## Snapshot & Resume

Two kinds of persistent state under `<course-root>/.course-cram/`:

| Path | Owner | Content |
|---|---|---|
| `ingests/<topic>.md` | general-purpose subagents (ingest write) | complete per-topic transcription |
| `ingests/INDEX.md` | main agent | topic metadata (source files, pages, timestamps, flags) |
| `session.md` | main agent | session state (map log, teach/quiz/exam logs, weak-area list) |

Ingests persist automatically across sessions. `session.md` is append-only; the pre-entry check reads it to offer resume.

### When to write `session.md`

- User picks **Snapshot & exit** from any post-step menu
- Before a long exam attempt (insurance against context compaction)
- User says "save progress" / "I'll continue later"

### Schema

See `CONTEXT.md` → "session.md schema".

### Resume flow

1. Pre-entry check reads `session.md`, offers resume
2. On resume: run `ls <course-root>/.course-cram/ingests/` and read `INDEX.md` to confirm topics present
3. If any topic ingest is missing: prompt the user to re-ingest that topic
4. Otherwise: jump to the last-active phase — no re-map needed

## Key Principles

- **User-driven transitions only.** Post the prompt, wait, act on the choice. No auto-chaining.
- **Subagents own ingest I/O.** Main agent sees metadata and rendered outputs only.
- **Ingests persist across sessions.** Map once; reuse forever until materials change.
- **Answer keys gated.** Revealed only when the user explicitly asks via the post-step menu.
- **Cite sources.** Subagents cite `<source-filename>, page <N>` using ingest page markers.

## Gotchas

- **Subagent type for writes**: Explore subagents lack Write access. Always use **general-purpose** type for ingest-write subagents. Explore is fine for all read-only subagents (teach/quiz/exam).
- **Large PDFs**: Read tool errors on PDFs > 10 pages without the `pages` argument. Ingest-write subagent prompts must explicitly instruct paging through (`pages: "1-10"`, `pages: "11-20"`, …). Do not skip tail pages.
- **Image-heavy slides**: lecture slides often carry 50%+ of content in charts, payoff diagrams, tables, and equation graphics. A PDF that parses to little text is NOT empty — the ingest-write subagent must visually render and transcribe every examinable element per the non-text rules in `CONTEXT.md`.
- **Equation fidelity**: never paraphrase a formula. The no-loss clause requires original variable names and structure. Paraphrased formulas produce wrong quiz/exam answers.
- **mkdir**: the ingest directory `<course-root>/.course-cram/ingests/` must exist before writing. Ingest-write subagent prompts include a `mkdir -p` instruction. If the main agent initiates a partial re-ingest, the directory already exists — no harm in re-running mkdir.
- **Anchor question labeling**: the exam-generation subagent must label every past-paper anchor with its source (paper reference + original question number as found in the ingest content).
- **No past papers in ingest**: if the requested exam type has no past-paper ingest, ask whether to proceed with a generic format or abort.
- **Scope gaps**: if a subagent returns `[SCOPE GAP: <topic>]`, main agent must not invent content. Offer to extend the map for that topic first.
- **Tutorial solutions as answer source**: when quiz/exam marking subagents derive answers, prefer the reasoning style and notation from tutorial-solution sections in the ingests over general knowledge.
