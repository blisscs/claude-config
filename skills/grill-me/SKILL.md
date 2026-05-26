---
name: grill-me
description: Before implementing, grill the user with a focused round of clarifying questions to pin down a vague request — scope, inputs/outputs, edge cases, success criteria, and constraints. Use when the request admits more than one reasonable interpretation, or when getting it wrong would mean rework. Bundle questions into ONE round, then proceed.
---

# grill-me

The user has asked you to grill them before writing code. Their previous message (or whatever they're about to type next) is a request that is too vague, too broad, or too load-bearing to implement on assumptions. Your job is to extract just enough specification to do the work correctly the first time — not to interrogate them into exhaustion.

## Operating rules

1. **One round only.** Bundle every question you need into a single `AskUserQuestion` call. Don't drip-feed. If you find yourself wanting a second round, you under-prepared the first.
2. **Cap at 4 questions.** If you have more, you don't understand the request well enough to pick the load-bearing ones — re-read the request and prioritize.
3. **Only ask what you cannot answer yourself.** Before each question, ask yourself: can I infer this from the codebase, the conventions in CLAUDE.md/AGENTS.md, or obvious defaults? If yes, don't ask — state your assumption in the answer instead.
4. **Frame as real choices, not yes/no.** "Should I use X?" wastes a turn. "Which of A, B, C?" or "Where does X live: A or B?" is actionable. Give 2–4 concrete options per question with a short description of the tradeoff. Include a recommended option (first, labeled `(Recommended)`) when you have a clear preference.
5. **Lead with the load-bearing question.** The one whose answer most changes what you'd build. If the user only answers one, that's the one that should pay off.

## What to grill for (pick the relevant ones)

- **Scope** — Where does the change live? What is explicitly out of scope? Is this a one-off or a reusable abstraction?
- **Inputs / outputs** — What are the exact types, shapes, formats? What about empty/null/error cases?
- **Behavior on edge cases** — Empty input, duplicates, conflicts, network failure, partial data, concurrent calls.
- **Success criteria** — How will the user know it's done? Specific test? Visible behavior? Performance target?
- **Constraints** — Backwards compatibility? Performance budget? Dependency restrictions? Deadline?
- **Audience / surface** — Internal helper vs public API? CLI flag vs config file vs env var?
- **Integration points** — Which existing module/file/function should this call into or replace?

## What NOT to ask

- Anything answered by reading 1–2 files in the repo. Read them first.
- Style/naming when the surrounding code makes the convention obvious.
- "Do you want tests?" — assume yes if the project has tests; if it doesn't, don't add a test framework.
- "Should I commit?" — never commit unsolicited; that's a project default, not a per-task choice.
- Permission to proceed. The user invoking `/grill-me` IS the permission — once you have answers, build it.

## After the answers come back

- State your understanding back in **one or two sentences**, then start work. Do not produce a multi-section "spec doc" — that's overhead the user didn't ask for.
- If an answer reveals a new ambiguity that is genuinely load-bearing and could not have been anticipated, you may ask **one** follow-up. Otherwise, proceed with the best reasonable interpretation and flag it as you go.
- If the answers contradict each other or reveal the task is much larger than the user thinks, surface that *before* writing code, in one short message.

## Anti-patterns

- ❌ Asking 8 questions across 3 turns.
- ❌ Asking yes/no questions you could answer yourself.
- ❌ Producing a requirements document instead of code.
- ❌ Refusing to start until every edge case is nailed down — pick a reasonable default and call it out.
- ❌ Re-asking things the user already implied.

Be precise, be brief, and earn the right to start coding in one round.
