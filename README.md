# orchestrator-mode

> A Claude Code skill that runs the main session as a lightweight orchestrator. It plans, delegates to subagents, verifies, and synthesizes, so the main context stays small and clean.

![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blue)
![License](https://img.shields.io/badge/License-MIT-green)

## Why

The main session's context is the scarcest resource in a long task. Every file dump, log, or search result stays there and crowds out the plan. A subagent can read 40 files and hand back 15 lines: you keep the conclusion, not the noise.

## What It Does

When a task is multi-step or would pull a lot of text into context, the skill guides Claude to:

1. **Plan** the work and record tasks, marking which depend on others
2. **Delegate** independent tasks in parallel, with self-contained prompts and a size limit on results
3. **Implement through role-briefed agents**, in waves, with one owner per file
4. **Verify** subagent claims against the real diff and test output
5. **Synthesize** the results and report back

Small jobs stay in the main session. A quick lookup or an edit of about 20 lines in a known file does not need a subagent.

## Installation

```bash
git clone https://github.com/AbdElrahmaN31/orchestrator-mode.git ~/.claude/skills/orchestrator-mode
```

Or copy only the skill file:

```bash
mkdir -p ~/.claude/skills/orchestrator-mode
curl -fsSL https://raw.githubusercontent.com/AbdElrahmaN31/orchestrator-mode/main/SKILL.md \
  -o ~/.claude/skills/orchestrator-mode/SKILL.md
```

## Usage

Claude can load the skill by itself on large tasks. You can also ask for it:

```
/orchestrator-mode Add a nullable due_date field to tasks, with tests.
```

```
Map how user signup works in this repo. Keep my context clean, use subagents.
```

The examples in `SKILL.md` use a small to-do list app and are not tied to one stack. Swap the paths, role lines, and commands for your own.

## Project Structure

```
orchestrator-mode/
├── SKILL.md          # The skill definition
└── evals/evals.json  # Test prompts and expected behaviour, including a negative control
```

## License

[MIT](LICENSE)
