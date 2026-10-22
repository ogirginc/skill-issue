---
name: interview
description: Interview the user in depth about a plan, then write the decisions up as a spec next to the plan file. Use when the user asks to be interviewed or grilled about a plan, wants open questions in a plan surfaced before implementation, or wants a plan turned into a spec.
---

# Interview

Interview me in depth about a plan, then write the decisions up as a spec.

## Step 1: Find the plan

Read the plan file I pointed you to.

- If I didn't name one, or the file does not exist, look for likely plan files
  (e.g. `*.plan.md`, `PLAN.md`, files in `docs/plans/`) and ask me which one to use.

## Step 2: Interview me

Use your harness's structured question tool (e.g. AskUserQuestion in Claude Code)
if it has one; otherwise ask in plain messages. Ask about anything the plan leaves
open or assumes: technical implementation, UI & UX, edge cases, concerns, tradeoffs,
sequencing, what's explicitly out of scope.

Rules for the interview:

- Ask 2–4 questions per round — group each round around a theme
  (scope → technical approach → UX → risks/tradeoffs).
- Skip questions the plan already answers. Never ask the obvious; probe the
  decisions I haven't realized I'm making.
- Offer multi-select when options aren't mutually exclusive.
- Let my answers drive follow-up rounds — dig into anything surprising or vague.
- Stop when a new round would only re-confirm what's already been decided.

## Step 3: Write the spec

1. Summarize the key decisions back to me in a short list.
2. Write the finished spec to `<plan-name>.spec.md` next to the plan file
   (e.g. `docs/plans/auth.md` → `docs/plans/auth.spec.md`). The spec should
   capture the plan plus every decision made during the interview, including
   what was ruled out and why.
