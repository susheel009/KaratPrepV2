# Citi/Karat Java Practice Set — Master Index

Three companion documents, designed for active practice under interview-like conditions.

**Quick navigation:** [Debugging Problems](#debugging-problems-reference) | [Ladder Problems](#ladder-problems-reference) | [Solutions](#solutions-reference)

## The Three Documents

| Document | What's in it | When to use it | Quick Link |
|---|---|---|---|
| **Debugging Problems** | 20 predict-the-output / find-the-bug snippets | Day 1–2: drill these aloud, no code execution | [📖 Open](karat-debugging-problems.md) |
| **Ladder Problems** | 10 OOP debug-and-extend exercises (Course/Run/RunCollection style) | Day 2–3: 35-min timer per problem, talk out loud, run the tests | [📖 Open](karat-ladder-problems.md) |
| **Solutions** | Full answers, mechanism explanations, exact exception names, extension implementations | Only after attempting — don't peek | [📖 Open](karat-solutions.md) |

## Rules of Engagement

1. **No Copilot. No IntelliJ autocomplete. No ChatGPT.** Use a plain text editor or the Carrot Studio playground if they sent a link.
2. **Talk out loud the entire time.** Even alone. Especially alone.
3. **For debugging problems**: predict output before running. State the mechanism with exact terminology. Name the exact exception class.
4. **For ladder problems**: read the entire codebase before touching anything. Read the failing test data — it's a hint. Dry-run line by line.
5. **Time discipline**: 25–30 seconds per debugging problem in your head; 35 minutes per ladder problem with the test suite.

## Mapping Back to the Karat Question Bank

Patterns confirmed from real Citi/Karat candidate reports (see `Karat_Question_Bank.xlsx`):

| Pattern from question bank | Covered in |
|---|---|
| Mutable default argument | DBG-01 (Java equivalent: shared mutable state) |
| String + int concatenation | DBG-05 |
| Lambda closure / late binding | DBG-08 |
| `is` vs `==` | DBG-03 (Java equivalent: `==` vs `.equals`) |
| Shallow vs deep copy | DBG-14 |
| Context manager (`with`) | DBG-09 (Java equivalent: try-with-resources) |
| List comprehension | DBG-18 (Java equivalent: streams) |
| Threading and CPU-bound | DBG-20 + LADDER-09 |
| `final` keyword usage | DBG-19 |
| Generics wildcards | DBG-22 |
| Map keys mutability | DBG-23 |
| Obstacle course bug fix | LADDER-01, LADDER-04, LADDER-07 (similar pattern of "missing filter on complete state") |
| `bestOfBests` aggregation | LADDER-02, LADDER-08 (similar "min/max per index across collection") |
| `chanceOfPersonalBest` simulation | LADDER-10 (Monte Carlo extension) |
| Gym membership "wrong filter" bug | LADDER-03, LADDER-05 |
| Highway journey log calculation | LADDER-06 (route/journey aggregation) |

---

## Debugging Problems Reference

**20 predict-the-output & find-the-bug problems** — all in [`karat-debugging-problems.md`](karat-debugging-problems.md).

| # | Problem | Pattern | Link |
|:---:|---|---|:---:|
| 1 | Static List Accumulator | Shared mutable state | [→ DBG-01](karat-debugging-problems.md#dbg-01--static-list-accumulator) |
| 2 | Integer Cache Boundary | Autoboxing & Integer cache | [→ DBG-02](karat-debugging-problems.md#dbg-02--integer-cache-boundary) |
| 3 | String Pool Reference | `==` vs `.equals` for Strings | [→ DBG-03](karat-debugging-problems.md#dbg-03--string-pool-reference) |
| 4 | Map Lookup Auto-Unbox | NullPointerException on autoboxing | [→ DBG-04](karat-debugging-problems.md#dbg-04--map-lookup-auto-unbox) |
| 5 | String + Int Concatenation | Operator precedence in concat | [→ DBG-05](karat-debugging-problems.md#dbg-05--string--int-concatenation) |
| 6 | printf Format Mismatch | IllegalFormatConversionException | [→ DBG-06](karat-debugging-problems.md#dbg-06--printf-format-mismatch) |
| 7 | Remove While Iterating | ConcurrentModificationException | [→ DBG-07](karat-debugging-problems.md#dbg-07--remove-while-iterating) |
| 8 | Lambda in Loop Capture | Effectively-final / closure | [→ DBG-08](karat-debugging-problems.md#dbg-08--lambda-in-loop-capture) |
| 9 | Try-With-Resources Order | Reverse close order, suppressed exceptions | [→ DBG-09](karat-debugging-problems.md#dbg-09--try-with-resources-order) |
| 10 | Finally Overrides Return | Control flow in try/finally | [→ DBG-10](karat-debugging-problems.md#dbg-10--finally-overrides-return) |
| 11 | Switch Without Break | Fall-through | [→ DBG-11](karat-debugging-problems.md#dbg-11--switch-without-break) |
| 12 | Integer Division Truncation | Primitive arithmetic | [→ DBG-12](karat-debugging-problems.md#dbg-12--integer-division-truncation) |
| 13 | Floating-Point Equality | Float representation | [→ DBG-13](karat-debugging-problems.md#dbg-13--floating-point-equality) |
| 14 | Shallow Copy of Nested List | Reference vs value copying | [→ DBG-14](karat-debugging-problems.md#dbg-14--shallow-copy-of-nested-list) |
| 15 | Arrays.asList Add | UnsupportedOperationException | [→ DBG-15](karat-debugging-problems.md#dbg-15--arraysaslist-add) |
| 16 | Static Method "Override" | Method hiding vs overriding | [→ DBG-16](karat-debugging-problems.md#dbg-16--static-method-override) |
| 17 | String split Trailing Empties | String.split(regex, limit) | [→ DBG-17](karat-debugging-problems.md#dbg-17--string-split-trailing-empties) |
| 18 | Stream Reduce From Loop | Stream API decoding | [→ DBG-18](karat-debugging-problems.md#dbg-18--stream-reduce-from-loop) |
| 19 | final Keyword Three Ways | final on var/method/class | [→ DBG-19](karat-debugging-problems.md#dbg-19--final-keyword-three-ways) |
| 20 | Volatile Counter Race | Thread safety, atomicity vs visibility | [→ DBG-20](karat-debugging-problems.md#dbg-20--volatile-counter-race) |

**Bonus problems:** [DBG-21 (parseInt)](karat-debugging-problems.md#dbg-21--parseint-with-whitespace) | [DBG-22 (PECS)](karat-debugging-problems.md#dbg-22--pecs-wildcard) | [DBG-23 (Mutable Key)](karat-debugging-problems.md#dbg-23--mutable-hashmap-key)

---

## Ladder Problems Reference

**10 OOP debug-and-extend exercises** — all in [`karat-ladder-problems.md`](karat-ladder-problems.md).

| # | Problem | Bug Pattern | Extension | Link |
|:---:|---|---|---|:---:|
| 1 | Library Book Lending | Missing "returned" filter | Most-borrowed book | [→ LADDER-01](karat-ladder-problems.md#ladder-01--library-book-lending) |
| 2 | Restaurant Order History | Wrong status filter (paid vs all) | Best-selling combo (parallel min/max) | [→ LADDER-02](karat-ladder-problems.md#ladder-02--restaurant-order-history) |
| 3 | Fitness Tracker | Wrong field aggregated | Average duration per exercise | [→ LADDER-03](karat-ladder-problems.md#ladder-03--fitness-tracker) |
| 4 | Flight Booking System | Missing seat-availability filter | Revenue by route | [→ LADDER-04](karat-ladder-problems.md#ladder-04--flight-booking-system) |
| 5 | Quiz Grading System | Counts only one status (gym-membership-style) | Hardest question across attempts | [→ LADDER-05](karat-ladder-problems.md#ladder-05--quiz-grading-system) |
| 6 | Highway Journey Log | Includes incomplete journeys | Total distance per driver | [→ LADDER-06](karat-ladder-problems.md#ladder-06--highway-journey-log) |
| 7 | E-commerce Shopping Cart | Discount applied to wrong subset | Best price across carts | [→ LADDER-07](karat-ladder-problems.md#ladder-07--e-commerce-shopping-cart) |
| 8 | Music Streaming History | Missing finished-track filter | Top genre per user | [→ LADDER-08](karat-ladder-problems.md#ladder-08--music-streaming-history) |
| 9 | Bank Account Ledger | Missing posted-status filter | Highest spending category | [→ LADDER-09](karat-ladder-problems.md#ladder-09--bank-account-ledger) |
| 10 | Hotel Reservations | Missing confirmed filter | Probability simulation (chanceOfPersonalBest analog) | [→ LADDER-10](karat-ladder-problems.md#ladder-10--hotel-reservations--monte-carlo) |

---

## Solutions Reference

**Full worked solutions** — all in [`karat-solutions.md`](karat-solutions.md).

- **[Debugging Solutions](karat-solutions.md#debugging-solutions)** — output + mechanism + exception class + fix for each DBG
- **[Ladder Solutions](karat-solutions.md#ladder-solutions)** — bug location, root cause, Task 1 fix, Task 2 + Task 3 implementations for each LADDER

**Index by problem:**
- [DBG-01 through DBG-23 solutions](karat-solutions.md#debugging-solutions)
- [LADDER-01 through LADDER-10 solutions](karat-solutions.md#ladder-solutions)

---

## Recommended Practice Schedule (4 days)

- **Day 1**: [Debugging problems DBG-01 through DBG-12](karat-debugging-problems.md#index). Aim for 25 minutes total. Predict aloud, check [solutions](karat-solutions.md#debugging-solutions), write a one-liner per pattern.
- **Day 2**: [Debugging problems DBG-13 through DBG-23](karat-debugging-problems.md#index). Then start [LADDER-01](karat-ladder-problems.md#ladder-01--library-book-lending) (the canonical "missing filter" bug pattern — most likely Q1 in the real interview).
- **Day 3**: [LADDER-02 through LADDER-06](karat-ladder-problems.md#ladder-02--restaurant-order-history). Run them under timer. Record at least one full session and listen back.
- **Day 4**: [LADDER-07 through LADDER-10](karat-ladder-problems.md#ladder-07--e-commerce-shopping-cart). Light review of debugging patterns you fumbled. Stop early.

## What Each Problem Tests

The first column in the debugging document tags the **pattern category** so you know which Java mechanism is being probed. Recognize the category in 5 seconds; that's worth more than getting the exact output.

For ladder problems, every problem follows the same structure:
- Context paragraph (read it twice — wording is sometimes tricky)
- Three classes plus a `Solution` test harness
- **Task 1**: Fix the failing test (the bug is always one method that's logically wrong, not a syntax error)
- **Task 2**: Implement an extension method
- **Task 3** (optional, time-permitting): Either a harder extension or a Monte Carlo simulation

Solving Task 1 + Task 2 reliably is enough to pass. Don't burn yourself out chasing Task 3.

Go.
