# v2/CLAUDE.md — rules for study files

> Scoped rules for anything inside `v2/`. Inherits from `/CLAUDE.md` at repo root.

## When working in this directory

1. **Always read the matching doubts file first.** For `v2/NN_topic.md`, that's `v2/doubts/NN_topic_doubts.md`. The doubts are the calibration signal for the user's level on this topic.
2. **Doubts files are READ-ONLY.** Never edit them — no ✅, no status, nothing.
3. **Follow the 9-section template.** Reference: `.claude/templates/study_file_template.md`. The reference implementation is `01_concurrency.md`.
4. **Lead with the problem, never the code.** §1 and §2 contain zero Java syntax.
5. **Preserve the navigation header.** Every file starts and ends with the nav line described in `/CLAUDE.md` §6.

## Per-topic "related topics" map

When writing the **Related topics** part of the nav header, use this map. Pick 2–4 most-related entries:

| Topic | Most-related |
|-------|--------------|
| 01 Concurrency | 02 Collections (concurrent maps), 07 JVM (memory model), 16 Debugging (race conditions) |
| 02 Collections | 01 Concurrency (concurrent collections), 03 OOP (equals/hashCode, generics), 06 Java 8+ (Streams) |
| 03 OOP & Language | 06 Java 8+ (functional interfaces, records), 09 SOLID, 04 Exceptions |
| 04 Exceptions | 03 OOP, 10 Testing, 12 Spring Boot (error handling) |
| 05 Strings | 03 OOP, 06 Java 8+ (text blocks), 07 JVM (String pool) |
| 06 Java 8+ | 02 Collections (Streams), 03 OOP, 01 Concurrency (CompletableFuture) |
| 07 JVM | 01 Concurrency (memory model), 02 Collections (heap layout), 16 Debugging |
| 08 Design Patterns | 09 SOLID, 11 Spring Core, 03 OOP |
| 09 SOLID | 08 Design Patterns, 03 OOP, 11 Spring Core |
| 10 Testing | 04 Exceptions, 11 Spring Core, 12 Spring Boot |
| 11 Spring Core | 12 Spring Boot, 14 AOP & Data, 09 SOLID |
| 12 Spring Boot | 11 Spring Core, 13 Spring REST, 14 AOP & Data |
| 13 Spring REST | 12 Spring Boot, 04 Exceptions, 10 Testing |
| 14 Spring AOP & Data | 11 Spring Core, 12 Spring Boot, 01 Concurrency (`@Transactional`, `@Async`) |
| 15 Kafka | 12 Spring Boot, 01 Concurrency (consumers), 16 Debugging |
| 16 Debugging | 17 Debugging Problems, 01 Concurrency, 02 Collections |
| 17 Debugging Problems | 16 Debugging, 01 Concurrency, 02 Collections |

## Practice/DataStructures references

When relevant, link to:
- `v2/DataStructures/DataStructures.md` — for collections and core algorithm topics
- `v2/Practice/karat-debugging-problems.md` — for debugging topics
- `v2/Practice/karat-ladder-problems.md` — for general practice
- `v2/Practice/karat-solutions.md` — solutions to the above

## Section depth budget (rough guidelines)

- §1 The Problem: 150–250 words
- §2 Walkthrough: 200–400 words (a table or step list, no prose-only walls)
- §3 First Code: ≤30 lines of code + ≤80 words of recap
- §4 Build Up: 4–8 subsections, each 100–200 words + small code snippet
- §5 Going Deep: 4–10 subsections, each 80–200 words; this is where dense reference material lives
- §6 Memory Aids: 1 decision tree + 1 mnemonic table + 1 "anchor problems" list
- §7 Cheat Sheet: 15–25 rapid-fire Q&A entries
- §8 Self-Test: ~4 easy / ~5 medium / ~5 hard
- §9 Glossary: every term used in §1–§5 with a plain-English definition

If a section is much shorter than this, the topic might not warrant the full template — flag it to the user before forcing it.
