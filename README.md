# ralph-super-simple

ralph-super-simple is not a specific agent implementation, but a minimal architecture pattern for multi-phase workflows based on `claude -p` loops.

Skill names can be freely created to fit your environment.

> Minimal architecture pattern for long-running Claude Code loops.
>
> This repository defines the pattern.
> Skills implement the pattern.

---

## Install

Copy the skill folders into your project's `.claude/skills/` directory:

```bash
# Full install (standalone + ralph-claude-code integration)
cp -r skills/* your-project/.claude/skills/

# Standalone only (no ralph-claude-code needed)
cp -r skills/ralphss-import skills/ralphss-clear your-project/.claude/skills/

# ralph-claude-code integration only
cp -r skills/ralphcc-import skills/ralphcc-clear your-project/.claude/skills/
```

After installation, run `/ralphss-import` or `/ralphcc-import` in Claude Code.

---

## Concept

**ralph-claude-code**
```
complex
robust
fragile
```

**openclaw**
```
powerful
heavy
overkill
```

**ralph-super-simple**
```
minimal
stable
evolvable
```

---

## Background

Claude Code is a powerful coding agent, but it has limitations when performing large tasks in a single session:

- **Breaks on errors**: Even simple errors can halt a session, requiring manual intervention
- **Context limits**: Sub-agents, agent teams, and other methods to escape context constraints keep emerging, but as plans grow larger, escaping those constraints becomes difficult. The overhead of summarizing, managing memory, clearing context, and transitioning between sessions becomes burdensome

To address this, I combined several tools.

### oh-my-claudecode (OMC)

[oh-my-claudecode](https://github.com/nicobailey-omc/oh-my-claudecode) is an excellent tool. It provides multi-agent orchestration, iterative loops, parallel execution, and dramatically improves development performance. Recommended as a default install.

### ralph-claude-code

[ralph-claude-code](https://github.com/frankbria/ralph-claude-code) inspired this project with its genuine approach to "never-stopping" autonomous loops. Its approach of repeatedly calling `claude -p` and detecting completion via EXIT_SIGNAL is simple yet effective.

However, there were some challenges in practice:

- **Timeout misjudgment**: Normal completion treated as timeout ([#198](https://github.com/frankbria/ralph-claude-code/issues/198))
- **Prompt size management**: Context accumulates across sessions, exceeding prompt limits
- **Exit code handling**: Unconditional termination from `set -e` ([#200](https://github.com/frankbria/ralph-claude-code/issues/200), [#175](https://github.com/frankbria/ralph-claude-code/issues/175))
- **Premature termination**: Loop exits before work is complete ([#194](https://github.com/frankbria/ralph-claude-code/issues/194))

These skills were inspired by real-world experience with this tool. Give it a try.

### Conclusion

Ultimately, **a well-structured phase plan combined with a repeatable execution structure** is all you need to drastically reduce the overhead of session limits, context management, summarization, clearing, and restarting. Even without relying on specific tools, `claude -p "prompt"` → check output → repeat is sufficient.

Based on this experience, I share two skills. I would be delighted if they help or inspire someone.

---

## Core Principle

```python
# This is the whole idea
for phase in phases:
    prompt = read(f"{phase}/PROMPT.md")
    while "EXIT_SIGNAL: true" not in run(f"claude -p '{prompt}'"):
        prompt = "Check fix_plan.md and continue next task."
    run(f"bash {phase}/verify.sh")
```

---

## Skills

### 1. `ralphcc-import` — ralph-claude-code integration

Distributes an implementation plan into `.ralph/` file structure. Copies configuration files (PROMPT.md, fix_plan.md) phase by phase for ralph-claude-code, with automatic multi-phase switching.

**Requires**: [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) installation

**Usage:**
```
/ralphcc-import   # Distribute plan to .ralph/ structure
/ralphcc-clear    # Clean up .ralph/ (prepare for next import)
```

**Run:**
```bash
cd your-project
bash .ralph/run.sh            # Run in external terminal
# or
ralph --live --verbose        # Run ralph CLI directly
```

### 2. `ralphss-import` — standalone execution

Distributes an implementation plan into `.ralphss/` file structure. Runs with `claude -p` only — no external tools required.

**Requires**: `claude` CLI only (no additional installation)

**Usage:**
```
/ralphss-import plan.md       # Distribute plan to .ralphss/ structure
/ralphss-clear                # Clean up .ralphss/ (prepare for next import)
```

**Run:**
```bash
cd your-project
bash .ralphss/run.sh   # Run in external terminal
```

\* Plans generated with [oh-my-claudecode](https://github.com/nicobailey-omc/oh-my-claudecode)'s `/ralplan` skill include phase structure, exit criteria, and verification standards — a great fit as import input.

---

## Structure

```
ralph-super-simple/
├── README.md
├── README.ko.md
├── LICENSE
└── skills/
    ├── ralphss-import/SKILL.md    # Standalone (claude -p only)
    ├── ralphss-clear/SKILL.md     # Clean .ralphss/
    ├── ralphcc-import/SKILL.md    # ralph-claude-code integration
    └── ralphcc-clear/SKILL.md     # Clean .ralph/
```

### Generated Output Comparison

**ralphss-import** (`.ralphss/`):
```
.ralphss/
├── run.sh                     # Autonomous loop (direct claude -p calls)
├── MASTER_PLAN.md             # Full plan
├── AGENT.md                   # Build/test
├── specs/requirements.md      # Detailed specs
└── phases/
    ├── phase-1/
    │   ├── PROMPT.md
    │   ├── fix_plan.md
    │   └── verify.sh
    └── ...
```

**ralphcc-import** (`.ralph/`):
```
.ralph/
├── run.sh                     # Phase switching + ralph invocation
├── MASTER_PLAN.md
├── AGENT.md
├── specs/requirements.md
└── phases/
    ├── phase-1/
    │   ├── PROMPT.md
    │   ├── fix_plan.md
    │   └── verify.sh
    └── ...
```

---

## Terminology Reference

| Term/Concept | Origin | Description |
|-------------|--------|-------------|
| `EXIT_SIGNAL` | [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) | Loop termination signal. Complete after 2 consecutive detections |
| `RALPH_STATUS` block | ralph-claude-code | Status report format parsed by ralph |
| `.ralphrc` | ralph-claude-code | Project settings (ALLOWED_TOOLS, timeout) |
| `.ralph/` directory | ralph-claude-code | ralph execution environment |
| `LOOP_STATUS` block | **ralph-super-simple** | Lightweight EXIT_SIGNAL-based status report format |
| `.ralphss/` directory | **ralph-super-simple** | Standalone execution environment |
| `run.sh` | **ralph-super-simple** | Autonomous loop + phase switching integrated script |
| `fix_plan.md` | Shared | Commit-sized task checklist |
| `PROMPT.md` | Shared | Loop control instructions |
| `verify.sh` | Shared | Per-phase automated verification script |
| `MASTER_PLAN.md` | Shared | Multi-phase full plan |
| `AGENT.md` | Shared | Build/test commands |
| `specs/requirements.md` | Shared | Detailed domain specs |

---

## Design Principles

### 1. Phase = Isolated Session

Each phase starts in a fresh session. No context carried over from previous phases — no context contamination.

### 2. PROMPT.md = Loop Control, specs/ = Domain Knowledge

Keep PROMPT.md light. Detailed specs live in separate files. Prevents prompt size issues.

### 3. verify.sh = Automated Quality Gate

Don't trust self-reports. Verify with concrete commands: `grep`, `pytest`, `curl`.

### 4. fix_plan.md = 1 Task = 1 Commit

Keep work units small so each loop has a clear goal. Max 3-5 file changes.

### 5. Ctrl+C Safe Shutdown

`run.sh`/`phase_runner.sh` traps interrupts and cleans up state files.

---

## Compatible Tools

| Tool | Role | Required |
|------|------|----------|
| [Claude Code](https://claude.com/claude-code) | CLI agent | Required |
| [oh-my-claudecode](https://github.com/nicobailey-omc/oh-my-claudecode) | Planning, agent orchestration | Recommended |
| [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) | Autonomous loop engine | Recommended (when using `ralphcc-import`) |

---

## Known Limitations

The items below are intentionally not implemented. The current structure works well enough, and these can be added as needed when issues actually arise.

| Area | Description | Current State |
|------|-------------|---------------|
| **EXIT_SIGNAL-only termination** | Possible infinite loop if Claude omits EXIT_SIGNAL or formatting breaks | verify.sh serves as secondary gate. Can switch to verify-based termination if needed |
| **State persistence (state.json)** | Cannot restore previous phase/loop position after interruption | Safe shutdown via Ctrl+C trap. Restarts from phase beginning |
| **Verify failure retry** | No automatic retry loop on verify.sh failure | Manual inspection and re-run on failure |
| **Explicit prompt file loading** | Possibility of missing fix_plan.md in context-free session | File paths are specified in PROMPT.md; no issues in practice |

The philosophy of this skill is **minimal implementation**. Responding to actual failures in context is more effective than writing failure-handling code in advance.

---

## Notes

`ralphcc-import` came first — that's the skill I actually use. Later, I explored a simpler idea: a skill that just repeats short execution steps tailored to my environment. That became `ralphss-import`. It simply proves the inspiration.

---

## License

MIT
