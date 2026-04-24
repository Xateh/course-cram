# course-cram

A structured, four-phase exam preparation pipeline for any course. Replaces ad-hoc "help me study" prompts with a repeatable workflow that maintains depth, scope control, and exam realism across a full study session — including across multiple sessions.

---

## Overview

```
map  →  teach  →  quiz  →  exam
         ↑___↓      ↑___↓
```

Each phase feeds the next. You drive every transition — Claude never auto-advances. At the end of every phase you get a numbered menu; reply with a number or phase name to continue.

| Phase | What happens |
|-------|-------------|
| **Map** | Claude checks for existing per-topic ingests on disk. If found, offers to reuse or update. If not, dispatches subagents to read and transcribe all course files, writing one dense markdown ingest per topic. |
| **Teach** | A subagent reads the relevant topic ingest(s) and explains at your chosen depth: full re-teach, review, recap, or subtopic deep-dive. |
| **Quiz** | A subagent reads the topic ingest(s) and generates 5–8 scoped questions (conceptual + numerical), presented all at once. Answer key revealed only when you ask. |
| **Exam** | A subagent reads all mapped ingests, infers format from past-paper ingests, and generates a full mock paper (~30% past-paper anchor questions + ~70% new). |

---

## Invocation

In any Claude Code session:

```
/course-cram
```

or describe what you want (Claude will auto-trigger the skill):

```
help me cram for my course final
I want to prep for the midterm
let's do a mock exam for weeks 1-6
```

---

## Context Management

Course materials at full fidelity project to hundreds of thousands of tokens — too much for the main conversation. The skill solves this with **persistent, subagent-owned, topic-split ingests on disk**.

### How it works

**Ingests live at `<course-root>/.course-cram/ingests/`**

One `.md` file per topic (e.g. `week-03-portfolio-theory.md`), written by a subagent. The ingests are persistent — they survive across sessions. Map once; reuse forever (or until you update them).

**Only subagents read/write ingest content**

Claude's main conversation never opens source files or ingest files. Instead:

- **Map**: dispatches general-purpose subagents (one per topic) to read source files and write ingest files. Main conversation receives only manifests (file path, page count, flags).
- **Teach / Quiz / Exam**: dispatches Explore subagents that read the relevant topic ingest(s), generate the content within a word budget, and return the rendered response. Main conversation receives only the output.

Main context footprint stays: topic index metadata + session state + current rendered response — regardless of how large the course is.

**No information loss**

Every ingest-write subagent prompt includes a verbatim "no information loss" clause. Transcription rules cover: text verbatim, equations as LaTeX with original variable names, charts described fully (axes/labels/curves/takeaway), diagrams reconstructed node-by-node, tables cell-by-cell, screenshots transcribed, handwritten annotations verbatim or `[illegible]`, decorative images noted in one line.

**Output budgets**

Rendered responses have word caps:

| Response | Cap |
|---|---|
| Teach — Recap | ≤ 500 words |
| Teach — Review | ≤ 1500 words |
| Teach — Full re-teach | ≤ 3500 words |
| Quiz answer key | ≤ 80 words per question |
| Exam answer key | ≤ 150 words per long Q, ≤ 60 words per MCQ |

For deeper treatment of a specific question, use the **Clarify** option in the post-step menu. Budgets only shape display — the full ingest stays on disk.

---

## Phase Reference

### Entry

On first invocation Claude checks for a prior session snapshot. If found:

```
Previous session found (updated <timestamp>). Last phase: <phase>. Topics: <N>.
1. Resume    — jump back to where you left off (ingests already on disk)
2. Start fresh
3. View snapshot
```

If no snapshot, Claude posts:

```
Course Cram ready. Where would you like to start?
1. Map   — check/build topic ingests on disk
2. Teach — explain topic(s)
3. Quiz  — short scoped quiz
4. Exam  — comprehensive mock
```

---

### Phase 1 — Map

**What Claude does first**: checks `<course-root>/.course-cram/ingests/` for existing ingests.

#### If no ingests found:

```
No ingest found at <path>. Begin ingestion?
1. Begin ingest
2. Change root
3. Cancel
```

On confirm, Claude enumerates files, proposes topic grouping (one topic per unit by default), confirms the scope with you (≤ 150 words), then dispatches one general-purpose subagent per topic in parallel.

#### If ingests found:

```
Found <N> topic ingests (last updated <date>):
- <topic> — <N> files, <pages> pages, updated <date>
- ...

1. Use existing      — skip ingest; go to post-map menu
2. Update one topic  — re-dispatch one subagent
3. Update all        — re-dispatch all subagents
4. Add new topic     — dispatch one subagent for new files
5. Remove a topic    — delete file + index entry
6. Cancel
```

**Material categories detected automatically:**

| Category keyword | Matches |
|---|---|
| `lecture` | `lecture slides/` directory |
| `tutorial` | `tutorial/` directory (questions only) |
| `tutorial-solutions` | `tutorial solutions/` or `tutorial/answer/` |
| `midterm` | `midterm test/` directory |
| `final-exam` | `final exam/` directory |
| `admin` | `admin/` directory |
| `all` | Everything above |

**Post-map menu:**
```
1. Repeat            — re-dispatch subagents for every topic
2. Update scope      — re-enter map with different topics/weeks
3. Add materials     — new topic or files
4. Remove materials  — delete a topic ingest
5. Proceed to Teach
6. Proceed to Quiz
7. Proceed to Exam
8. Done
9. Snapshot & exit   — save session state and end
```

---

### Phase 2 — Teach

**What Claude asks for:**
- Which topic(s) to teach (from the `INDEX.md` topic list)
- Detail level:

| Level | What you get |
|---|---|
| **1. Full re-teach** | First principles, derivations, worked examples, common pitfalls |
| **2. Review** | Key concepts, formulas, representative worked examples |
| **3. Recap** | High-level summary, 5–10 key takeaways |
| **4. Subtopic deep-dive** | You name a specific subtopic; Claude covers only that |

A subagent reads the relevant topic ingest(s) and returns teaching content. Citations use `<source-filename>, page <N>` from ingest page markers.

**Post-teach menu:**
```
1. Repeat this teach    — same scope, same level
2. Clarify              — ask questions about what was taught
3. Proceed to Quiz      — uses this teach scope
4. Proceed to Exam
5. Back to Teach        — different topic or level
6. Back to Map
7. Done
8. Snapshot & exit
```

---

### Phase 3 — Quiz

**Input:** defaults to the last teach scope; you can override.

**What Claude generates (via subagent):**
- 5–8 questions covering the scope
- Mix of conceptual short-answer, numerical, and formula-application/interpretation
- All questions at once: numbered, marks per question, total at top
- **No answers shown**

**What to do:** Attempt all questions in your reply (or say "skip to answers").

**Post-quiz menu:**
```
1. Repeat quiz    — fresh questions, same scope
2. Clarify        — ask about a specific question
3. Show answers   — answer key + worked explanations + your score
4. Proceed to Exam
5. Back to Teach
6. Done
7. Snapshot & exit
```

**Answer reveal:** Per-question worked explanation (≤ 80 words each), your attempt marked, total score. Key derives from the ingest, not from your answers.

---

### Phase 4 — Exam

**What Claude asks for:**
- Exam type: `Midterm`, `Final`, or a custom name
- Scope: defaults to all topics; you can narrow

**Format inference:** A subagent reads the past-paper ingest(s) and extracts: question count, types (MCQ / short / long / numerical), marks distribution, time limit, rubric style, cover-page instructions. The mock exam matches this format.

**Question mix:**
- ~**30% anchor questions** — lifted verbatim from past-paper ingest content, labeled with source (e.g. `[Final Exam Practice Paper 2, Q1]`)
- ~**70% new questions** — generated to cover the full mapped scope, matching difficulty and format

**Exam presentation:** Single structured paper with title, instructions, section breakdown, question numbering, marks per question, and a stated (simulated) time limit. No answers shown.

**Post-exam menu:**
```
1. Repeat exam        — fresh paper, same scope
2. Clarify            — ask about a specific question
3. Show answers       — full worked solutions + marked attempt + weak areas
4. Back to Teach      — target identified weak areas
5. Back to Quiz
6. Back to Map
7. Done
8. Snapshot & exit
```

**Answer reveal:** Full worked solutions (≤ 150 words per long Q, ≤ 60 words per MCQ), your attempt marked, performance summary by topic, and the 2–3 topics most worth re-teaching.

---

## Session Persistence

The skill creates and maintains `<course-root>/.course-cram/` automatically:

```
<course-root>/.course-cram/
├── ingests/
│   ├── INDEX.md                   ← topic metadata (main-agent-readable)
│   ├── week-01-intro.md           ← per-topic transcription (subagent-only)
│   ├── week-02-time-value.md
│   ├── past-papers-final.md
│   └── ...
└── session.md                     ← session state (main-agent-readable)
```

**Ingests persist automatically** — no snapshot needed. Next time you invoke `/course-cram` in the same directory, Map finds them and offers to reuse.

**`session.md`** (state only, no content) is written when you pick **Snapshot & exit** from any post-step menu. On the next invocation, Claude detects it and offers to resume from your last phase.

---

## Workflow Examples

### Full linear run (recommended before a final)
```
/course-cram
→ Map: <course-root>, weeks 1-13, all
  → (first time: ingest subagents run, ~100 pages per topic file written to disk)
→ Proceed to Teach: "Portfolio theory", Review
→ Proceed to Quiz
→ Show answers
→ Proceed to Teach: "Capital structure", Full re-teach
→ Proceed to Quiz
→ Show answers
→ Proceed to Exam: Final, weeks 1-13
→ Show answers
→ Back to Teach: [target weak areas identified in exam review]
→ Done
```

### Second session (ingests already on disk)
```
/course-cram
→ Map: (ingests found) → Use existing
→ Proceed to Teach: [topic], Recap
→ Proceed to Exam: Final
→ Show answers
→ Done
```

### Quick topic review
```
/course-cram
→ Map → Use existing (or ingest just the relevant week)
→ Teach: "Options pricing", Recap
→ Clarify: "what does N(d1) represent intuitively?"
→ Proceed to Quiz
→ Show answers
→ Done
```

### Midterm crunch (2 hours before)
```
/course-cram
→ Map: weeks 1-6, lecture + tutorial-solutions + midterm
→ Proceed to Exam: Midterm, weeks 1-6
→ Show answers
→ Back to Teach: [2 worst topics], Review
→ Proceed to Quiz
→ Show answers
→ Done
```

---

## Context and Limitations

### PDF reading
Ingest-write subagents use the Read tool with the `pages` argument to page through PDFs. Very large documents (50+ pages per topic file) may take several tool calls and a noticeable pause during the Map phase. This cost is paid once — subsequent teach/quiz/exam turns read the compact ingest files.

### Non-text content
Subagents read PDFs visually. Charts, payoff diagrams, equations rendered as images, embedded tables, diagrams, and screenshots are all transcribed to text in the ingest file. If a scan is low-resolution or handwriting is unclear, the subagent flags it in the manifest; you can supply clarification at the post-map menu.

### Answer quality
Quiz and exam answer keys derive from the ingest files, which reflect the course's notation, reasoning style, and official tutorial solutions. If a topic wasn't mapped, Claude will not generate questions for it.

### Past papers required for exam format
The exam phase infers format from past-paper ingests. If no past papers were included in the map scope, Claude will ask whether to proceed with a generic format or abort. For best results, always include `midterm` or `final-exam` (or both) in your map scope.

### Update vs. re-ingest
If you add new course files mid-semester, use **Add new topic** or **Update one topic** from the map menu — no need to re-ingest everything. The `INDEX.md` updates automatically with new timestamps.

---

## Files

```
course-cram/
├── SKILL.md      ← Claude's operating instructions (not human-facing)
├── CONTEXT.md    ← Subagent prompt templates, schemas, output budgets (not human-facing)
└── README.md     ← this file
```

At runtime, the skill creates:
```
<course-root>/.course-cram/
├── ingests/      ← per-topic transcription files (subagent-written)
└── session.md    ← session state (written on Snapshot & exit)
```

No configuration files, no scripts, no external dependencies.
