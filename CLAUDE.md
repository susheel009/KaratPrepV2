# CLAUDE.md — project rules for KaratPrepV2

> **For Claude:** This file is your single source of truth for how to work in this repo. It overrides any conflicting instruction in your memory or training. Re-read it whenever the user asks you to "follow CLAUDE.md" or "do it the right way".
> **For the user:** Edit this file freely. Whatever's here is what Claude will follow.

---

## 1. Project context

This repo is a personal study workspace for a **Karat technical screening interview**. The stack covered:

- **Core Java** — concurrency, collections, OOP, exceptions, strings, Java 8+ features, JVM internals
- **Spring** — Core/DI, Boot, REST, AOP/Data
- **Kafka** — fundamentals, consumer groups, exactly-once
- **Practice & debugging** — bug taxonomy, 20 debugging problems, ladder problems, data structures

Study files live in `v2/`. Practice problems in `v2/Practice/`. Data structures in `v2/DataStructures/`.

## 2. The user

- Preparing for a Karat technical screen, **foundational level**. Their concurrency doubts asked about kitchen analogies, what the "main thread" is, whether `synchronized` is "a modifier like static". Don't assume terminology like "monitor", "happens-before", "memory barrier" without grounding it in plain language first.
- Email: susheel009@gmail.com (per session context — confirm before relying on it for anything operational).

## 3. How to teach (HARD RULE — never deviate)

When writing or rewriting any study material in `v2/`, follow this order. **Never lead with code.**

1. **The Problem** — story-style, no code, no Java keywords. One real-world analogy that you stick with. Land the "this is why X exists" claim.
2. **Walkthrough** — one tiny scenario narrated step by step. Pseudo-code or English at most. Reader should be able to predict the code shape before seeing it.
3. **First Code** — minimal working example, every non-obvious line commented. ≤25 lines if possible.
4. **Build Up** — practical patterns one at a time. Each subsection: problem → fix → why it works.
5. **Going Deep** — interview-level depth (memory model, JIT effects, edge cases, version differences). Only after the foundation is solid.
6. **Memory Aids** — mnemonics, decision trees, "if asked X, think Y".
7. **Cheat Sheet** — rapid-fire Q&A for pre-interview review (≤30s per Q).
8. **Self-Test** — easy → medium → hard.
9. **Glossary** — plain-English definitions.

The reference template is `.claude/templates/study_file_template.md`. The first file in this style is `v2/01_concurrency.md` — match its tone and depth gradient.

## 4. Accuracy bar (HARD RULE)

- Be maximally accurate. **If you are uncertain about anything, say so explicitly and verify with WebSearch before answering. Never guess.**
- For Java behaviour, **always clarify version applicability** when the answer differs across Java 8 / 11 / 17 / 21. Examples that bite: virtual threads (Java 21+), `synchronized` carrier pinning (still in Java 21, mostly fixed in Java 24+), records (Java 16+), sealed classes (Java 17+), text blocks (Java 15+).
- Cite differences between textbook theory and actual JVM/JIT behaviour where relevant (e.g. `volatile` reordering rules, `Thread.State` having no `RUNNING`, double-checked locking and JMM).
- Prefer precision over brevity. If an answer needs a caveat, include it.

## 5. Doubts workflow (HARD RULE — read-only)

- Doubts files (`v2/doubts/<NN>_<topic>_doubts.md`) are **the user's log**. They are **read-only to you**. Never add ✅ markers, status notes, "addressed" lines, or any other modifications.
- The doubts file is the calibration signal: it tells you what the user actually finds unclear vs what they've already grasped. Read it before rewriting the matching study file. Use the analogies it already uses.
- Three slash commands implement the workflow (defined in `.claude/commands/`):
  - `/add-doubt <topic> <question>` — append a Q to the right doubts file (Claude writes the entry).
  - `/update-files [topic]` — read doubts files, rewrite matching `v2/NN_topic.md` files. **Never** edit doubts files.
  - `/push-doubts` — stage `v2/doubts/`, commit, push to `origin/develop`. Confirm with user before pushing.

## 6. Navigation header (HARD RULE for `v2/*.md` files)

Every rewritten file in `v2/` starts with this nav header (see `v2/01_concurrency.md` for a working example):

```markdown
# NN — <Topic Title>

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/NN_topic_doubts.md) · [← Prev: NN-1 …](./NN-1_….md) · [Next: NN+1 … →](./NN+1_….md)

**Priority:** <icon> <level> · **Related topics:** <links to 2–4 most-related files>

## Contents
1. [The Problem](#1-the-problem)
2. [Walkthrough](#2-walkthrough)
3. [First Code](#3-first-code)
4. [Build Up](#4-build-up--practical-patterns)
   - 4.1 …
   - 4.2 …
5. [Going Deep](#5-going-deep--interview-level)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet](#7-cheat-sheet)
8. [Self-Test](#8-self-test)
9. [Glossary](#9-glossary)

[At the end of the file, repeat the nav line for easy return.]
```

## 7. Git conventions

- Default branch: `develop`. Doubts get pushed here (via `/push-doubts`).
- `main` is the published branch — do not push to it without explicit instruction.
- Never `--force` push, never skip hooks (`--no-verify`), never push if pre-commit fails.
- Never `git add -A` or `git add .` — always stage by path. Especially: never re-stage `.claude/settings.json` (a leaked PAT exists in commit `9288c1e`; user is handling cleanup separately).
- Commit message style: imperative, prefixed by area — `docs(doubts): …`, `docs(v2): rewrite 02_collections`, `chore(commands): …`.

## 8. Tone

- Concise. The user reads markdown comfortably; don't pad with summaries that just restate the diff.
- For exploratory questions, give a 2-3 sentence recommendation with the trade-off — don't dive in until they agree.
- For task-style questions, say what you'll do in one line, do it, report back.

## 9. Repo layout snapshot

```
KaratPrepV2/
├── CLAUDE.md                    ← (this file)
├── README.md                    ← syllabus overview
├── v2 Prep Plan.md
├── v2/
│   ├── CLAUDE.md                ← rules specific to working inside v2/
│   ├── 00_INDEX.md              ← topic index
│   ├── 01_concurrency.md … 17_debugging_problems.md
│   ├── doubts/                  ← per-topic Q logs (READ-ONLY for Claude)
│   │   ├── README.md
│   │   └── 01_concurrency_doubts.md
│   ├── DataStructures/
│   └── Practice/
└── .claude/                     ← gitignored — local-only
    ├── Instructions.md          ← legacy session instructions (still authoritative on accuracy)
    ├── commands/                ← /add-doubt, /update-files, /push-doubts
    └── templates/               ← study_file_template.md
```
