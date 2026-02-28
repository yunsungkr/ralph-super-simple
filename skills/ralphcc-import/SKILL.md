---
name: ralphcc-import
description: Distribute implementation plan into .ralph/ file structure. Use for ralph import.
---

# ralphcc-import

Reads an implementation plan file and distributes it into `.ralph/` file structure for Ralph for Claude Code autonomous loop.

> **This skill only creates files.** No code modification, testing, or git operations.

## Workflow
```
Write plan → ralphcc import → Run ralph --monitor externally
```

## Input
- Implementation plan file (markdown). Must include exit criteria, per-phase tasks, and technical constraints.

## Output

### Single Phase (default)
```
.ralph/
├── PROMPT.md              # Loop control instructions (concise)
├── fix_plan.md            # Commit-sized task checklist
├── AGENT.md               # Build/test commands
└── specs/
    └── requirements.md    # Detailed domain specs
.ralphrc                   # Project settings (review ALLOWED_TOOLS if file exists)
```

### Multi Phase (when plan has 2+ phases)
```
.ralph/
├── phases/
│   ├── phase-1/
│   │   ├── PROMPT.md        # Per-phase loop control
│   │   ├── fix_plan.md      # Per-phase tasks
│   │   └── verify.sh        # Per-phase verification (exit 0 = pass)
│   ├── phase-2a/
│   │   ├── PROMPT.md
│   │   ├── fix_plan.md
│   │   └── verify.sh
│   └── ...
├── run.sh                   # Phase auto-switch script
├── MASTER_PLAN.md           # Full plan (reference only, ralph must not modify)
├── AGENT.md                 # Shared build/test (all phases)
└── specs/
    └── requirements.md      # Shared detailed specs (all phases)
.ralphrc                     # Project settings
```

---

## File Generation Rules

### 1. PROMPT.md — Loop Control Only, Minimize Domain

**Principle**: Domain knowledge goes in `specs/requirements.md`. PROMPT.md contains only what ralph needs to run the loop correctly.

**Required sections** (in this order):

```markdown
# Ralph — [Project/Task Name]

## Context
[3-5 lines. Project name, one-line description, current state]

## Current Objectives
[4-6 key objectives extracted from plan]

## Startup Check (run at loop start)
cat .ralphrc | grep -E "ALLOWED_TOOLS|CLAUDE_TIMEOUT_MINUTES|MAX_CALLS_PER_HOUR"
Check permissions, timeout, and call limits from the above command output.

## Key Principles
- ONE task per loop — focus on the single most important item
- Search codebase first, don't assume
- Use sub-agents (file search, analysis)
- Update fix_plan.md then commit
- Detailed specs at .ralph/specs/requirements.md

## Protected Files (do not modify)
- Entire .ralph/ directory, .ralphrc
- [Additional protected files derived from plan]

## Testing Guidelines
- Test ratio: ~20% of total work
- Test only new implementations, do not refactor existing tests

## Architecture Rules Reference
- [List relevant .claude/rules/ file links only]

## Exit Criteria (EXIT_SIGNAL: true)
[Up to 5 concrete criteria from plan. All conditions must be AND true]
1. ...
2. ...

## Prohibited Actions
[List out-of-scope actions]

## RALPH_STATUS Report

Always include this block at the end of every response:

---RALPH_STATUS---
STATUS: IN_PROGRESS | COMPLETE | BLOCKED
TASKS_COMPLETED_THIS_LOOP: <number>
FILES_MODIFIED: <number>
TESTS_STATUS: PASSING | FAILING | NOT_RUN
WORK_TYPE: IMPLEMENTATION | TESTING | DOCUMENTATION | REFACTORING
EXIT_SIGNAL: false | true
RECOMMENDATION: <one line summary>
---END_RALPH_STATUS---

### Example — In Progress
STATUS: IN_PROGRESS / TASKS: 2 / FILES: 5 / TESTS: PASSING
WORK_TYPE: IMPLEMENTATION / EXIT_SIGNAL: false
RECOMMENDATION: Continue with next task

### Example — Complete
STATUS: COMPLETE / TASKS: 1 / FILES: 1 / TESTS: PASSING
WORK_TYPE: DOCUMENTATION / EXIT_SIGNAL: true
RECOMMENDATION: All requirements met

### Example — Blocked
STATUS: BLOCKED / TASKS: 0 / FILES: 0 / TESTS: FAILING
WORK_TYPE: DEBUGGING / EXIT_SIGNAL: false
RECOMMENDATION: Stuck on [error] - human help needed

## Current Task
Follow .ralph/fix_plan.md and choose the most important item to implement next.
```

> **Note**: RALPH_STATUS block must be written in **English** (parsed by bash script).

### 2. fix_plan.md — 1 Task = 1 Commit

**Size rule**: 1 task = 1 commit = **max 3-5 file changes**

```markdown
# Fix Plan

## High Priority
- [ ] Task name (list 1-3 target files)
- [ ] Task name

## Medium Priority
- [ ] Task name

## [BLOCKED — reason]
- [ ] Task name (prerequisite not met)

## Completed
- [x] Project initialization

## Notes
- Reference: .ralph/specs/requirements.md
```

- Use `## Phase 1: High Priority` format if phase separation is needed
- Always split large tasks (e.g., "migrate 8 methods" → individual per-file tasks)

### 3. specs/requirements.md — Detailed Domain Specs

All domain information not in PROMPT.md:
- Per-phase details (method lists, migration targets, etc.)
- Architecture constraints in detail
- Data models, DTO definitions
- Verification methods and commands

### 4. AGENT.md — Build/Test Commands

```markdown
# Agent Build Instructions

## Tech Stack
[Languages, frameworks, DB, etc.]

## Build/Run
[Commands with absolute paths]

## Test
[Backend + frontend commands]

## Key Rules
[DB access patterns, FK rules, etc. summary]

## Key Learnings
[Ralph updates during loop]
```

### 5. MASTER_PLAN.md — Full Plan (Multi Phase only)

Generated only for multi phase. Summarizes the full phase structure based on input plan.

```markdown
# Master Plan — [Project/Task Name]

## Phase List
| Phase | Directory | Goal | Depends On |
|-------|-----------|------|------------|
| 1     | phase-1   | ...  | None       |
| 2A    | phase-2a  | ...  | 1          |
| 2B    | phase-2b  | ...  | 1          |

## Overall Completion Criteria
1. All Phase verify.sh exit 0
2. [Additional criteria]
```

> **Note**: Include MASTER_PLAN.md in Protected Files so ralph doesn't modify it.

### 6. Per-Phase PROMPT.md (Multi Phase only)

Contains only this phase's goals and exit criteria. Common info replaced with reference paths:

```markdown
# Ralph — [Phase Name]

## Context
Phase N of M. [Phase goal in 1-2 lines]

## Common References
- Build/test: .ralph/AGENT.md
- Detailed specs: .ralph/specs/requirements.md
- Full plan: .ralph/MASTER_PLAN.md

## Current Objectives
[3-5 key objectives for this phase]

## Startup Check (run at loop start)
cat .ralphrc | grep -E "ALLOWED_TOOLS|CLAUDE_TIMEOUT_MINUTES|MAX_CALLS_PER_HOUR"

## Key Principles
[Same as single phase PROMPT.md]

## Protected Files (do not modify)
- Entire .ralph/ directory, .ralphrc

## Testing Guidelines
- Test ratio: ~20% of total work
- Test only new implementations

## Exit Criteria (EXIT_SIGNAL: true)
[3-5 exit criteria for this phase only]

## RALPH_STATUS Report
[Same format as single phase PROMPT.md]

## Current Task
Follow .ralph/fix_plan.md and choose the most important item to implement next.
```

### 7. Per-Phase verify.sh (Multi Phase only)

Bash script for automated phase completion verification. Called by run.sh.

**Rules:**
- 1-5 verification commands (grep, pytest, curl, etc.)
- All commands pass → `exit 0`, any failure → `exit 1`
- Must have executable permission (`chmod +x`)

```bash
#!/bin/bash
set -e

# Example: Phase 1 — Container module exists + importable
grep -q "class AppContainer" api/container.py
python -c "from api.container import AppContainer; print('OK')"

echo "Phase 1 verify: PASSED"
exit 0
```

### 8. run.sh (Multi Phase only)

Auto-switch script that runs phases in order. Invoked externally with `bash .ralph/run.sh`.

**Behavior:**
1. Track current phase via `.ralph/.current_phase` file
2. Scan subdirectories in `.ralph/phases/` alphabetically
3. Copy current phase's `PROMPT.md` and `fix_plan.md` to `.ralph/` root
4. Call `ralph_loop.sh` (existing ralph loop)
5. Run phase's `verify.sh`
6. Pass → next phase, fail → stop

**Template:**

```bash
#!/bin/bash
set -euo pipefail

RALPH_DIR=".ralph"
PHASES_DIR="$RALPH_DIR/phases"
CURRENT_FILE="$RALPH_DIR/.current_phase"

# Phase list (alphabetical)
phases=($(ls -d "$PHASES_DIR"/*/ 2>/dev/null | sort))
if [ ${#phases[@]} -eq 0 ]; then
  echo "ERROR: No phases found in $PHASES_DIR"
  exit 1
fi

# Determine current phase
current_phase=""
if [ -f "$CURRENT_FILE" ]; then
  current_phase=$(cat "$CURRENT_FILE")
fi

started=false
[ -z "$current_phase" ] && started=true

for phase_dir in "${phases[@]}"; do
  phase_name=$(basename "$phase_dir")

  # Skip previous phases
  if [ "$started" = false ]; then
    if [ "$phase_name" = "$current_phase" ]; then
      started=true
    fi
    continue
  fi

  echo "=== Phase: $phase_name ==="
  echo "$phase_name" > "$CURRENT_FILE"

  # Copy phase files to root
  cp "$phase_dir/PROMPT.md" "$RALPH_DIR/PROMPT.md"
  cp "$phase_dir/fix_plan.md" "$RALPH_DIR/fix_plan.md"

  # Run ralph loop (user environment's ralph_loop.sh path)
  if command -v ralph_loop.sh &>/dev/null; then
    ralph_loop.sh
  else
    echo "WARNING: ralph_loop.sh not found in PATH. Manual execution required."
    echo "Phase $phase_name PROMPT.md/fix_plan.md copied to .ralph/."
    echo "Press Enter after ralph completes to run verify..."
    read -r
  fi

  # Phase verification
  if [ -f "$phase_dir/verify.sh" ]; then
    echo "--- Verifying $phase_name ---"
    if bash "$phase_dir/verify.sh"; then
      echo "✓ $phase_name PASSED"
    else
      echo "✗ $phase_name FAILED — stopping"
      exit 1
    fi
  fi
done

echo "=== All phases complete ==="
rm -f "$CURRENT_FILE"
```

### 9. .ralphrc — Project Settings

If file exists, review `ALLOWED_TOOLS` only. If not, create:
- Add project tools to `ALLOWED_TOOLS` (pytest, playwright, uvicorn, etc.)
- `CLAUDE_TIMEOUT_MINUTES=15`
- `MAX_CALLS_PER_HOUR=100`

---

## Execution Steps

### Common (1-3)
1. Read plan file
2. Read `docs/PROJECT_STATE.md` to understand current state
3. Determine phase count from plan → **2+ phases = multi mode**, otherwise single mode

### Single Phase Mode (legacy compatible)
4. `mkdir -p .ralph/specs`
5. Generate PROMPT.md (loop control focused)
6. Generate fix_plan.md (decomposed into commit-sized tasks)
7. Generate specs/requirements.md (detailed specs)
8. Generate AGENT.md (build commands)
9. Review/update .ralphrc
10. Output summary
11. **Stop** — no code modification, testing, or git

### Multi Phase Mode
4. `mkdir -p .ralph/specs .ralph/phases`
5. Generate MASTER_PLAN.md (full phase list + dependencies)
6. Create per-phase directories (`mkdir -p .ralph/phases/phase-{N}`)
7. Generate per-phase PROMPT.md (phase goals + exit criteria)
8. Generate per-phase fix_plan.md (commit-sized decomposition)
9. Generate per-phase verify.sh + `chmod +x`
10. Generate AGENT.md (shared build commands, all phases)
11. Generate specs/requirements.md (shared detailed specs)
12. Generate run.sh + `chmod +x`
13. Copy first phase's PROMPT.md/fix_plan.md to `.ralph/` root (ready to run)
14. Review/update .ralphrc
15. Output summary (phase count, directory list, run instructions)
16. **Stop** — no code modification, testing, or git

## Prohibited Actions
- No source code modification/deletion
- No test execution
- No git operations
- No file creation/modification outside `.ralph/` and `.ralphrc`
