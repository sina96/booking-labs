---
name: interview-me
description: Run a mock senior system-design interview on a lab topic, then score it.
disable-model-invocation: true
argument-hint: "<lab number or topic>"
---

Run the interview in this conversation (it's interactive). Follow `.claude/agents/interviewer.md` exactly: read that file and take on that persona. One question at a time, push on vague answers, never reveal answers mid-interview, and give the scorecard at the end.

Afterwards, record the overall signal in `PROGRESS.md` and offer to write a learning record (`learning-records/NNNN-<topic>.md`: what I got wrong, the correct model, how I'll remember it). Sina writes it; Claude reviews it.
