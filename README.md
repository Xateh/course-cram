# course-cram

You have a pile of lecture slides, tutorial sheets, and past papers. You have limited time before your exam. You need to go deep on some topics and wide on others, and you need to practice under exam conditions. This skill replaces ad-hoc "help me study" prompts with a repeatable four-phase pipeline — map your course once, then teach, quiz, and test yourself across as many sessions as you need.

---

## How it works

```
map  →  teach  →  quiz  →  exam
         ↑___↓      ↑___↓
```

The four phases match a real study arc: understand the material, test yourself on it, then simulate the exam. You drive every transition — Claude never auto-advances. At the end of every phase you get a numbered menu; reply with a number or phase name to continue.

| Phase | What happens |
|-------|-------------|
| **Map** | Claude checks for existing per-topic ingests on disk. If found, offers to reuse or update. If not, dispatches subagents to read and transcribe all your course files, writing one dense markdown ingest per topic. |
| **Teach** | A subagent reads the relevant topic ingest(s) and explains at your chosen depth: full re-teach, review, recap, or subtopic deep-dive. |
| **Quiz** | A subagent reads the topic ingest(s) and generates 5–8 scoped questions matched to your course type, presented all at once. Answer key revealed only when you ask. |
| **Exam** | A subagent reads all mapped ingests, infers format from past-paper ingests, and generates a full mock paper (~30% past-paper anchor questions + ~70% new). The format, style, and phrasing match your real exam exactly because they're inferred from your past papers. |

---

## Quick start

### First time (mapping a new course)
1. Invoke `/course-cram` or describe what you need (e.g., "help me cram for my final")
2. Select `Map` and point Claude to your course directory
3. Confirm the proposed topic grouping
4. Let the ingestion run (takes a few minutes for large courses, but only happens once)
5. Choose your next phase: Teach, Quiz, or Exam

### Returning to a mapped course
1. Invoke `/course-cram` in the same directory
2. Map detects your existing ingests and offers to reuse them
3. Skip to Teach, Quiz, or Exam immediately — no re-mapping needed
4. If you want to resume from where you left off, look for the resume menu

### Natural language triggers
You don't need to type `/course-cram`. Any of these will auto-trigger the skill:
- "help me cram for my course final"
- "I want to prep for the midterm"
- "let's do a mock exam for weeks 1-6"

---

## Phase reference

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

On confirm, Claude enumerates files, proposes topic grouping (one topic per unit by default), confirms the scope with you, then dispatches one general-purpose subagent per topic in parallel.

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
| **1. Full re-teach** | First principles, detailed explanations, examples, common pitfalls |
| **2. Review** | Key concepts, core methods, representative examples |
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
- Mix of question types matched to your course (conceptual, applied, analytical, or numerical)
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

**Answer reveal:** Per-question worked explanation, your attempt marked, total score. Key derives from the ingest, not from your answers.

---

### Phase 4 — Exam

**What Claude asks for:**
- Exam type: `Midterm`, `Final`, or a custom name
- Scope: defaults to all topics; you can narrow

**Format inference:** A subagent reads the past-paper ingest(s) and extracts: question count, types (MCQ / short / long / numerical), marks distribution, time limit, rubric style, cover-page instructions. The mock exam matches this format exactly.

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

**Answer reveal:** Full worked solutions, your attempt marked, performance summary by topic, and the 2–3 topics most worth re-teaching.

---

## What gets stored

The skill creates and maintains `<course-root>/.course-cram/` automatically:

```
<course-root>/.course-cram/
├── ingests/
│   ├── INDEX.md                   ← topic metadata
│   ├── week-01-intro.md           ← per-topic transcription
│   ├── week-02-topic-name.md
│   ├── past-papers-final.md
│   └── ...
└── session.md                     ← session state
```

**Ingests persist automatically** — no snapshot needed. Next time you invoke `/course-cram` in the same directory, Map finds them and offers to reuse.

**`session.md`** (state only, no content) is written when you pick **Snapshot & exit** from any post-step menu. On the next invocation, Claude detects it and offers to resume from your last phase.

**Nothing is sent anywhere** — it's all local files in your course directory.

**No information loss:** Every ingest includes text verbatim, equations as LaTeX with original variable names, charts described fully (axes, labels, curves, takeaway), diagrams reconstructed node-by-node, tables cell-by-cell, and screenshots transcribed. You're not getting a paraphrase; you're getting a complete transcription.

---

## Workflow examples

### Two weeks out (recommended approach)
```
/course-cram
→ Map: <course-root>, weeks 1-13, all
  → (first time: ingest subagents run, written to disk)
→ Proceed to Teach: [topic A], Review
→ Proceed to Quiz
→ Show answers
→ Proceed to Teach: [topic B], Full re-teach
→ Proceed to Quiz
→ Show answers
→ Proceed to Exam: Final, weeks 1-13
→ Show answers
→ Back to Teach: [weak areas identified in exam review]
→ Done
```

### Coming back after a break
```
/course-cram
→ Map: (ingests found) → Use existing
→ Proceed to Teach: [topic], Recap
→ Proceed to Exam: Final
→ Show answers
→ Done
```

### One topic in 20 minutes
```
/course-cram
→ Map → Use existing (or ingest just the relevant week)
→ Teach: [topic], Recap
→ Clarify: "can you explain [concept] in more detail?"
→ Proceed to Quiz
→ Show answers
→ Done
```

### Two hours before the midterm
```
/course-cram
→ Map: weeks 1-6, all
→ Proceed to Exam: Midterm, weeks 1-6
→ Show answers
→ Back to Teach: [2 worst topics], Review
→ Proceed to Quiz
→ Show answers
→ Done
```

---

## Limitations and tips

### PDF reading
Ingest-write subagents use the Read tool with the `pages` argument to page through PDFs. Very large documents (50+ pages per topic file) may take several tool calls and a noticeable pause during the Map phase. This cost is paid once — subsequent teach/quiz/exam turns read the compact ingest files.

### Non-text content
Subagents read PDFs visually. Charts, diagrams, equations rendered as images, embedded tables, and screenshots are all transcribed to text in the ingest file. If a scan is low-resolution or handwriting is unclear, the subagent flags it in the manifest; you can supply clarification at the post-map menu.

### Answer quality
Quiz and exam answer keys derive from the ingest files, which reflect the course's notation, reasoning style, and any provided solutions. If a topic wasn't mapped, Claude will not generate questions for it.

### Past papers required for exam format
The exam phase infers format from past-paper ingests. If no past papers were included in the map scope, Claude will ask whether to proceed with a generic format or abort. For best results, include your past papers when mapping.

### Adding files mid-semester
If you add new course files after the initial map, use **Add new topic** or **Update one topic** from the map menu — no need to re-ingest everything. The `INDEX.md` updates automatically with new timestamps.

---

## Files

```
course-cram/
├── SKILL.md      ← Claude's operating instructions (not human-facing)
├── CONTEXT.md    ← Subagent prompt templates, schemas, output budgets (not human-facing)
└── README.md     ← this file
```

No configuration files, no scripts, no external dependencies.
