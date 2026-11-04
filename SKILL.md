---
name: orchestrator-mode
description: Run the main session as a lightweight orchestrator that plans, delegates to subagents, and synthesizes, so the main context stays small and clean. Use this whenever a task is multi-step, touches many files, needs codebase exploration or research, involves running tests/builds with long output, or would otherwise pull large amounts of text into context. Also use it when the user says "orchestrate", "delegate", "use subagents", "fan out", "keep context clean", "don't flood the context", or asks for a large feature, refactor, migration, audit, or investigation across a codebase — even if they don't name this skill.
---

# Orchestrator Mode

You are the orchestrator. Your job is to **plan, decompose, delegate, verify, and synthesize**. Subagents do the bulk reading, searching, and implementing.

This includes the implementation phase. After the plan is agreed, do not switch into writing the code yourself. Fan the work out to role-briefed implementer agents (see "Implementation fan-out"). You write code only for small edits (≤ ~20 lines in a file you already know) and small fixes found during verification.

Why: the main session's context is the scarcest resource in a long task. Every file dump, log, or search result you pull in stays there and crowds out the plan, the user's intent, and later work. A subagent can read 40 files and hand back 15 lines. You keep the conclusion, not the noise.

## Decision table

| Situation | Do it yourself | Delegate |
|---|---|---|
| Read a few lines of one known file | ✅ | |
| Check one known symbol / config value | ✅ | |
| Small edit (≤ ~20 lines) in a file you already know | ✅ | |
| Quick `git status`, `ls`, or a short command | ✅ | |
| Find where something is defined/used across the codebase | | ✅ Explore |
| Read a large file, many files, or long logs | | ✅ Explore / general-purpose |
| Run a test suite, build, or linter with long output | | ✅ general-purpose |
| Web or docs research | | ✅ general-purpose |
| Design an approach for a non-trivial change | | ✅ Plan |
| Implement a plan step bigger than a small edit (any new file, multi-file change, migration + model + tests) | | ✅ role-briefed general-purpose |
| Several independent implementation steps (disjoint files) | | ✅ in parallel, one role-briefed agent per step |
| Small fix after verifying an agent's work (≤ ~20 lines) | ✅ | |
| Code review of a diff or branch | | ✅ code-reviewer agent if available, else general-purpose |

Rule of thumb: if doing it yourself would add more than ~100 lines to context, or you can't predict how much output you'll get, delegate.

## Choosing the agent type

- **Explore** — read-only search and location ("where is X", "how does Y flow"). Cheap and fast. Cannot edit.
- **Plan** — design work: weigh approaches, list files to change, order the steps. Cannot edit.
- **general-purpose** — anything that edits files, runs commands, or mixes research with action.
- Prefer a more specific agent (e.g. a code-reviewer or code-explorer agent) when one is listed and fits.

## Implementation fan-out

Implementation is delegated the same way exploration is. Turn each plan step into one implementer agent with a specialist role.

**Self-check before you write code yourself:** "Is this more than ~20 lines, or more than one file?" If yes, delegate it.

- **Split by file ownership.** Give each agent a clear list of files it owns. Two agents running in parallel must never edit the same file. If two steps touch the same file, run them one after the other, or give both to one agent.
- **Give each agent a role.** Start the prompt with a role line that fits the step, for example:
  - "You are a database schema and data model specialist."
  - "You are an API layer specialist (validation, serializers, handlers, routes)."
  - "You are a service/business-logic specialist."
  - "You are a test writer."
  - "You are a refactoring specialist. Behaviour must not change."
  Then name any skills it should load and the project conventions to follow (sibling files to copy the style from, the formatter and linter to run, etc.).
- **Order the waves.**
  1. Contracts first: migrations, interfaces, shared types, config — anything other steps depend on.
  2. Then the independent steps in parallel (multiple Agent calls in one message).
  3. Then tests that need the finished code, and a final test-runner or reviewer agent.
  Pass each wave only the facts it needs from the wave before (names, signatures, `path:line`), not full summaries.
- **Tests per agent.** Each implementer runs only the tests for its own files and returns pass/fail counts. A broader run happens once at the end (one agent, or your own targeted re-run during Verify).
- **Fixes.** If verification finds a small problem (≤ ~20 lines), fix it yourself. If it is bigger, send it back to an agent with the exact failing test or `path:line` and the expected behaviour.

## Workflow

1. **Plan.** Restate the goal in one line. Break it into tasks and record them in the task list. Mark which tasks depend on others.
2. **Delegate.** Launch all *independent* tasks together — multiple Agent calls in one message so they run in parallel. Launch *dependent* tasks only after their inputs arrive. This applies to the implementation phase too, not only to exploration: implement through role-briefed agents in waves (see "Implementation fan-out").
3. **Verify.** Spot-check each result before you build on it (see below).
4. **Synthesize.** Combine the results, decide the next step, update the task list, and report to the user.

## Writing a delegation prompt

Subagents see nothing of this conversation. A vague prompt gets a vague (or wrong) result, and fixing it costs another round trip. Every prompt needs:

- **Goal** — what to find or build, and why it matters.
- **Context** — relevant paths, names, decisions already made, conventions to follow.
- **Constraints** — what not to touch, read-only or not, tools/commands to use or avoid.
- **Output format** — exactly what to return, and a size limit.

Always ask for a compact result: a short summary, key findings, and `path:line` references. Say explicitly: no full file contents, no raw logs, no long code blocks unless asked. If a test fails, ask for the failing test name and the few relevant lines of the error, not the whole run.

### Example 1 — exploration (Explore, parallel with others)

```
Goal: Find how task data is validated and stored in this to-do list app,
so I can add a "due_date" field.
Context: Task code is likely under src/api, src/services, and src/models.
Ignore dependencies and tests.
Constraints: Read-only.
Return (max ~25 lines):
- One-paragraph summary of the flow (request → service → model).
- Each relevant file as `path:line` with one line on its role.
- Any existing validation rules for task fields, quoted briefly.
- Open questions or surprises.
```

### Example 2 — implementation fan-out (role-briefed general-purpose, after the plan is agreed)

Task: add a nullable `due_date` (ISO 8601 date) field to tasks in a to-do list app.

**Wave 1** (contract — everything else depends on the column):

```
Role: You are a database schema and data model specialist.
Goal: Add a nullable `due_date` date column to tasks.
Context: The model is src/models/task.ext. Copy the style of the newest
migration in migrations/.
You own ONLY: a new migration file and src/models/task.ext.
Do not edit other files.
Constraints: No new dependencies. Run the project's formatter on your files.
Return (max ~15 lines): files changed as `path:line` + one line each;
the exact column name and type; anything you were unsure about.
```

**Wave 2** (launched together in one message, after wave 1 returns):

```
Role: You are an API layer specialist (validation, serializers).
Goal: Accept and return `due_date` on tasks.
Context: Column `due_date` (nullable date) now exists on tasks. Input is
validated in src/api/task_validator.ext:40-72 and returned by
src/api/task_serializer.ext.
You own ONLY those two files.
Rule: optional; if present, a valid ISO 8601 date (YYYY-MM-DD).
Return (max ~15 lines): files changed as `path:line`; the exact rule added.
```

```
Role: You are a test writer.
Goal: Cover `due_date` on task creation: valid date saved and returned,
missing value allowed, invalid value (e.g. "not-a-date") rejected with 422.
Context: Existing task tests are in tests/api/tasks/. Field name: `due_date`.
You own ONLY new or existing files under tests/api/tasks/.
Run only the task tests, using the project's test command.
Return (max ~20 lines): tests added as `path:line`; pass/fail counts;
for failures, test name + the 5 most relevant error lines.
```

The test writer can run in wave 2 because the field name and rules are already decided. If the tests fail only because wave-2 code is still missing, re-run them after both agents finish.

## Verify before you report

Subagent summaries can be wrong or optimistic. Before telling the user a task is done:

- **Code changes:** look at the actual diff (`git diff --stat`, then a targeted `git diff <file>` for the key parts). Confirm it matches what was claimed.
- **Tests:** re-run the specific affected tests yourself, or check that the reported command and counts are real. A summary saying "all tests pass" is a claim, not evidence.
- **Findings:** open one or two of the cited `path:line` references to confirm they say what the subagent said.

Keep these checks small and targeted — the point is to catch errors, not to redo the work.

## Context hygiene

- Track progress in the task list; update it as tasks finish so the plan survives long sessions.
- Don't re-read files a subagent already summarized. Trust the summary unless you are verifying it or need an exact line to edit.
- Keep summaries you received short when you relay them; pass only what the next step needs into the next delegation prompt.
- When the context grows large (many rounds, long results), suggest `/compact` to the user, or a fresh session with a short handoff note (goal, decisions made, remaining tasks, key `path:line` refs).

## When not to orchestrate

For a quick question, a one-file fix, or a lookup you can answer in one or two small tool calls, just do it. Delegation has a cost (a full prompt, a cold-start agent, a round trip). Use it when it saves more context than it costs.
