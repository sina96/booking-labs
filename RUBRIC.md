# Rubric: what "done" means for a lab

This is a learning repo, so experimenting freely is encouraged. The bar below only applies to calling a lab **done**.

## Proof levels

Every acceptance criterion from `/lab-brief` is tagged with the proof it needs:

| Tag | Meaning | Used for |
|---|---|---|
| `[test]` | Automated test (JUnit + Testcontainers) that reproduces the failure *and* passes after the fix | Correctness: lost/duplicate messages, DLQ routing, oversold seats, outbox guarantees |
| `[k6]` | Committed k6 script with thresholds + results summarised in `NOTES.md` | Latency, throughput, load levelling, cascading failure |
| `[observed]` | Seen manually (logs, Grafana trace, Kafbat UI, CLI output); what was seen goes in `NOTES.md` | Exploration, UI-only behaviour, one-off experiments |

**Minimum for "done":** every `[test]` criterion is green, `NOTES.md` has *prediction → observed → why* for each experiment, and there's an interview answer. Everything else is a bonus.

## Judge scoring (only when `/judge` is run)

| Dimension | 2: Shallow | 3: Solid | 4: Senior |
|---|---|---|---|
| **Correctness** | Happy path only | Handles the failure cases the lab names | Also handles unnamed ones (concurrency, restarts, partial failure) |
| **Proof** | Mostly `[observed]` where `[test]` was required | Required `[test]`/`[k6]` criteria met | Tests are deterministic, readable, catch regressions |
| **Understanding** | Observations only | Prediction → observed → why for every experiment | Explains broker/DB-level mechanisms, notes wrong predictions |
| **Trade-offs** | Names the pattern | When to use it + what it costs | Compares alternatives, failure modes, links to real experience |
| **Code quality** | Works but tangled | Clear boundaries and naming | What you'd approve in a code review at work |

**Verdicts:** PASS · PASS WITH NOTES · REVISE (with a specific list).
**Feedback format:** verdict → scores → top 3 gaps (each with why it matters) → one thing done well → one probing question, without the answer.
