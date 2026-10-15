---
name: judge
description: Judge a finished lab against RUBRIC.md using the read-only judge agent, then record the verdict.
disable-model-invocation: true
argument-hint: "<lab number>"
---

1. Spawn the `judge` subagent (Agent tool, subagent_type "judge") with the lab number and folder path. Tell it to follow its instructions and RUBRIC.md.
2. Show its feedback to Sina as-is. Don't soften it.
3. Ask Sina whether they agree with each top gap. A disagreement backed by a good argument is a learning win; note it.
4. Update the lab row in `PROGRESS.md` (status, verdict, date). On REVISE, list the items to fix under Notes.
