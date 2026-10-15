# Progress

## Decisions
- 2026-10-15: **Mentor mode.** I write the code; agents give steps, hints that climb in steps, and corrections. Implementation only on explicit request; boilerplate after a quick check that I understand it. (I want to learn by doing, not by prompting.)
- 2026-10-15: **Stack: latest versions.** Java 25 + Spring Boot 4.x, Gradle Kotlin DSL multi-project; Kotlin and Go for some services. (Portfolio showcase.)
- 2026-10-15: **Domain:** event ticketing with timed seat holds.
- 2026-10-15: **Proof levels:** `[test]` for correctness failures, `[k6]` for load/latency, `[observed]` for exploration (see RUBRIC.md).
- 2026-10-15: **Tracing:** basic OTel + Grafana LGTM from Lab 0; deep dive in Lab 8.
- 2026-10-15: **Pace:** self-paced, no deadlines. Roughly 1–1.5 h on weekdays + weekend blocks when possible. Labs have hour estimates in `docs/labs.md`.
- 2026-10-15: **Services and languages:** `booking` + `payment` Java 25 · `inventory` Kotlin + coroutines · `notification` Go (from Lab 1). Everything traced end to end.
- 2026-10-15: **Lab 0.5 event design** before Lab 1: `docs/events.md` is the contract later labs are checked against.
- 2026-10-15: **Lab 0 done =** healthy `compose up` from a fresh clone · Boot 4 SQS round trip on MiniStack (Testcontainers) · visible as one trace in Grafana · `./gradlew build` green.
- 2026-10-15: **Editors:** IntelliJ IDEA 2026.2 for creating and editing Java/Kotlin services (New Project wizard; clean up its nested Gradle files, see docs/dev-setup.md) · Zed + terminal for Go, Docker/compose, infra, scripts. Formatting is owned by the build (Spotless, gofmt) + `.editorconfig`, not the editor.
- 2026-10-15: **Layout:** one evolving system in `services/`, notes in `labs/lab-NN/NOTES.md`, git tag `lab-NN-done` per finished lab. Go is outside the Gradle build.
- 2026-10-15: **Docker:** Colima via terminal/Zed. To do: `~/.testcontainers.properties` pointing at the Colima socket so tests run from IntelliJ and the terminal hit the same daemon.
- 2026-10-15: **Gates:** a lab is "done" when it meets the RUBRIC minimum. `/judge`, `/interview-me`, `/recap` are optional and only run when I invoke them.

## Environment (checked 2026-10-15)
MacBook M3 Pro, 36 GB · JDK 25/21/17 (default `java` is 17, so point Gradle's toolchain at 25) · Maven · Go · k6 · gh · Docker CLI + Colima (not running) · missing: kind, terraform.

## Labs

| Lab | Est. | Status | Verdict | Notes |
|---|---|---|---|---|
| 0 Setup | 4–6 h | ⬜ | | |
| 0.5 Event design | ~1.5 h | ⬜ | | |
| 1 Sync vs async | 6–8 h | ⬜ | | |
| 2 Retries / DLQ | 8–12 h | ⬜ | | |
| 3 Idempotency | 5–7 h | ⬜ | | |
| 4 Outbox | 10–14 h | ⬜ | | |
| 5 Sagas | 10–14 h | ⬜ | | |
| 6 Concurrency | 5–7 h | ⬜ | | |
| 7 SQS vs Kafka | 8–10 h | ⬜ | | |
| 8 Tracing (deep) | 6–10 h | ⬜ | | |

## Assisted
_Anything an agent implemented for me. Date: what, and why I asked._

## Recap log
_Date: topics, score, what to revisit._
