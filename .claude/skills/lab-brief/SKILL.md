---
name: lab-brief
description: Brief Sina on a lab before they start: goal, acceptance criteria, predictions to make, reading. No solutions.
disable-model-invocation: true
argument-hint: "<lab number or name>"
---

1. Read the lab in `docs/labs.md` (or `~/Desktop/plans (aka pit of despair)/backend-2026/`), `PROGRESS.md` (Decisions + earlier labs) and the relevant `learning-records/`.
2. Check current docs for any library/version the lab relies on. Don't rely on memory.
3. Output:
   - **Mission:** the interview question this lab answers, in one line.
   - **Acceptance criteria:** a numbered checklist of observable outcomes, each tagged `[test]`, `[k6]` or `[observed]` per RUBRIC.md. These define "done".
   - **Predictions to write in NOTES.md before you run anything:** one question per experiment ("What happens to the message when…?").
   - **Read first:** 1–3 primary sources with the exact section.
   - **Building blocks:** the names of the APIs/annotations/config Sina will probably need (names only, no usage).
   - **Pitfalls that cost people hours:** environment/setup gotchas only, not design answers.
   - **Size:** a rough time estimate, and where to cut scope if it runs long.
   - **Suggested sessions:** split the lab into 1–1.5 h chunks using the estimate in docs/labs.md.
4. Don't include any implementation code. Ask whether Sina wants to `/grill-me` their design before building.
