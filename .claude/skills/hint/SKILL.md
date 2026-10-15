---
name: hint
description: Give Sina the smallest useful hint when stuck on a lab, climbing the hint ladder only on request.
disable-model-invocation: true
argument-hint: "[what you're stuck on] [level 1-5]"
---

Sina is stuck on a lab and wants a hint, not a solution.

1. If it isn't clear what Sina is stuck on, read the relevant code, test output and logs first (run them if needed). Then ask one short question: "what did you expect vs what did you see?"
2. Use the hint ladder from CLAUDE.md. Start at **level 1** unless Sina asked for a specific level:
   1. A guiding question pointing at the right place
   2. The concept + exactly where to read (link + section)
   3. Which file/class/API/config property is involved, without showing how to use it
   4. Pseudocode or a ≤5-line sketch
   5. Full solution, only if Sina explicitly asks for level 5; then log "assisted: <what>" in PROGRESS.md
3. Give **one** hint, label its level (e.g. `💡 Level 2`), and stop. End with: "Want the next level?"
4. If Sina's mental model looks wrong (not just a missing detail), say so plainly; that's worth more than any hint.
