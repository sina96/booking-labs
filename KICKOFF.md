# Kickoff

## 1. Before the first session (5 min, by you)
1. Move this folder to wherever your projects live on the Desktop. The `CLAUDE.md` + `.claude/` folder come along, so mentor mode keeps working.
2. `git init`, first commit, create the GitHub repo (`gh repo create booking-labs --public --source . --push`).
3. Open Claude Code **inside the repo folder** (so `CLAUDE.md` and the `.claude/` skills load).
4. Check that `/grill-me` and `/lab-brief` show up when you type `/`.

## 2. The first prompt (paste this)

```
Hi! This is my booking-labs repo. Read CLAUDE.md, PROGRESS.md, RUBRIC.md and
docs/labs.md first. You're my mentor, not my implementer.

I'm about to start Lab 0. Before giving me any steps:
1. Ask me 3–4 short questions to find out what I already know about Docker
   Compose health checks, Gradle multi-project builds and how a Spring Boot
   app connects to an AWS emulator, so you can pitch the help at my level.
2. Then run /lab-brief 0, splitting the lab into 1–1.5 h sessions.

From then on: give me steps, not code. When I'm stuck, give me the smallest
hint, and tell me straight away if I'm heading the wrong way.
```

## 3. Prompts for later sessions

| Situation | Say |
|---|---|
| Starting a session | "Read PROGRESS.md. Where was I, and what are the next 2–3 steps for this session?" |
| Stuck | `/hint <what I'm stuck on>`, or "I'm stuck: I expected X, I see Y" |
| Want more help | "Next hint level" |
| Want it done for me | "Implement <specific thing> for me and explain the key lines" |
| Boilerplate | "Boilerplate: <thing>." (Claude checks you understand it, then writes it) |
| Before building a design | `/grill-me <my design in a few sentences>` |
| Think a lab is done | `/judge <n>` |
| Interview practice | `/interview-me <n>` |
| Ending a session | "Update PROGRESS.md with where I stopped and what's next." |

## 4. Before going public
`docs/labs.md` opens with a personal note about the interview. Rewrite or remove it before you push, if you'd rather it wasn't public.
