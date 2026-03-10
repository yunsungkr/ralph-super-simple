# ralph-super-simple

`ralph-super-simple` is a repository of practical patterns for running AI coding agents in repeated loops until the job is finished.
Rather than focusing on a specific model implementation, it focuses on operating Claude Code, Codex CLI, and Gemini CLI around **phase-based planning, autonomous loops, and verification scripts**.

Tools used in this setup:
- [Claude Code](https://claude.com/claude-code)
- [OMC (oh-my-claudecode)](https://github.com/nicobailey-omc/oh-my-claudecode)
- [Codex CLI](https://github.com/openai/codex)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli)

> OMC is a multi-agent orchestration layer for Claude Code. It generates phase-separated plans (`/ralplan`) that feed directly into ralphss-loop, and minimizes failures by ensuring tasks are completed reliably — where a single agent session might stall or miss steps.

## TL;DR

> If you have a well-written step-by-step plan and a structure that can execute it repeatedly, you can greatly reduce the effort spent on session limits, context management, summarizing, clearing, and restarting.

- **Session Constraints**: Once context fills up, you end up repeating summaries, memory updates, clearing, and restarts.
- **Interruptions on Errors**: Even simple errors can break the session and require manual intervention every time.
- **CLI Boundaries**: If you depend on a single tool, that tool's limits become the limits of the work.

This repository addresses those problems with **a phase-based file structure and isolated session loops**.
Each phase runs in its own session, so context does not leak across phases, and the execution CLI can be swapped freely.

## Core Loop

```python
for phase in phases:
    prompt = read(f"{phase}/PROMPT.md")
    while "EXIT_SIGNAL: true" not in run(f"claude -p '{prompt}'"):
        prompt = "Check fix_plan.md and continue with the next task."
    run(f"bash {phase}/verify.sh")
```

The idea is simple. Lock each phase into **plan decomposition → repeated execution → automated verification**, and you can run long-duration tasks reliably — free from session constraints, regardless of the agent.

## Included Skills

| Skill | Purpose | Required Tools |
| --- | --- | --- |
| **ralphss-loop** | Repeatedly executes a plan through independent `claude -p` loops | Claude Code |
| **ralphss-disc** | Automates rotating discussion rounds between Codex, Gemini, and Claude | Claude Code + Codex CLI + Gemini CLI |

### ralphss-loop: Independent `claude -p` Loop

Runs with `claude -p` by default, but you can swap the CLI call in `run.sh` to Codex, Gemini, or any other agent.

```bash
/ralphss-loop my-plan.md                          # creates .ralphss/loop/{task_name}/
cd .ralphss/loop/{task_name} && bash run.sh       # run in a separate terminal
rm -rf .ralphss/loop/{task_name}/                  # cleanup
```

Directory structure:

```text
.ralphss/loop/{task_name}/
├── run.sh                     # Autonomous loop (calls claude -p directly)
├── MASTER_PLAN.md             # Overall plan
├── AGENT.md                   # Build/test commands
├── specs/requirements.md      # Detailed requirements
└── phases/
    ├── phase-1/
    │   ├── PROMPT.md          # Loop control instructions
    │   ├── fix_plan.md        # Commit-level checklist
    │   └── verify.sh          # Phase completion verification
    └── ...
```

### ralphss-disc: Codex/Gemini/Claude Multi-Model Discussion

```bash
/ralphss-disc my-topic.md                          # creates .ralphss/disc/{task_name}/
cd .ralphss/disc/{task_name} && bash run.sh        # rotating Codex → Gemini → Claude rounds
rm -rf .ralphss/disc/{task_name}/                  # cleanup
```

Directory structure:

```text
.ralphss/disc/{task_name}/
├── run.sh                     # Round runner (run from this directory)
├── topic.md                   # Copy of the original topic
├── synthesis.md               # Final-round synthesis (last claude output)
└── rounds/
    ├── round-1-codex/
    │   ├── instruction.md     # Round objective
    │   ├── prompt.md          # Actual prompt (topic + instruction + prev output)
    │   └── output.md          # Round response
    ├── round-2-gemini/
    │   └── ...
    ├── round-3-claude/
    │   └── ...
    └── ...
```

### Unified `.ralphss/` Directory

All skills share a single `.ralphss/` root, organized by type and dated task name:

```text
.ralphss/
├── loop/
│   ├── 2025-01-15_api-refactor/
│   └── 2025-01-20_auth-module/
└── disc/
    ├── 2025-01-15_caching-strategy/
    └── 2025-01-22_database-migration/
```

## Quick Start

### 1) Install the skills

```bash
cp -r skills/* your-project/.claude/skills/
```

### 2) First run example

```bash
/ralphss-loop my-plan.md
cd .ralphss/loop/{task_name} && bash run.sh
```

### 3) Recommended operating rules

- Keep `fix_plan.md` aligned to the `1 task = 1 commit` rule.
- Write `verify.sh` as commands a human can read and verify immediately.
- Make the phase exit condition explicit with `EXIT_SIGNAL`.

## Terminology

| Term | Description |
| --- | --- |
| `PROMPT.md` | Phase loop control instructions |
| `fix_plan.md` | Commit-level task checklist |
| `verify.sh` | Automated verification script for phase completion |
| `EXIT_SIGNAL` | Loop termination signal |
| `MASTER_PLAN.md` | Overall multi-phase plan |
| `AGENT.md` | Build/test command collection |
| `instruction.md` | Discussion round objective instructions |
| `synthesis.md` | Final synthesized result of the discussion |

## Acknowledgments

The autonomous loop structure and patterns in this repository were inspired by [ralph-claude-code](https://github.com/frankbria/ralph-claude-code).

## License

MIT