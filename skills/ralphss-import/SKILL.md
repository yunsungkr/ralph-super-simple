---
name: ralphss-import
description: Distribute implementation plan into .ralphss/ file structure. Use for ralphss import.
---

# ralphss-import

Reads an implementation plan and distributes it into a `.ralphss/` file structure for autonomous `claude -p` loops. No external tools required — runs with `claude -p` only.

> **This skill only creates files.** No code modification, testing, or git operations.

## Workflow
```
Write plan → ralphss import → Run bash .ralphss/run.sh in external terminal
```

## Input
- Implementation plan file (markdown). Must include exit criteria, per-phase tasks, and technical constraints.

## Output

### Single Phase (default)
```
.ralphss/
├── run.sh                 # Autonomous loop script
├── PROMPT.md              # Loop control instructions
├── fix_plan.md            # Commit-sized task checklist
├── AGENT.md               # Build/test commands
└── specs/
    └── requirements.md    # Detailed domain specs
```

### Multi Phase (when plan has 2+ phases)
```
.ralphss/
├── run.sh                   # Phase auto-switch + autonomous loop
├── phases/
│   ├── phase-1/
│   │   ├── PROMPT.md        # Per-phase loop control
│   │   ├── fix_plan.md      # Per-phase tasks
│   │   └── verify.sh        # Per-phase verification (exit 0 = pass)
│   ├── phase-2/
│   │   └── ...
│   └── ...
├── MASTER_PLAN.md           # Full plan (reference only)
├── AGENT.md                 # Shared build/test
└── specs/
    └── requirements.md      # Shared detailed specs
```

---

## File Generation Rules

### 1. PROMPT.md — Loop Control Only

**Principle**: Domain knowledge goes in `specs/requirements.md`. PROMPT.md contains only loop control info.

**Required sections** (in this order):

```markdown
# [Project/Task Name]

## Context
[3-5 lines. Project name, one-line description, current state]

## Current Objectives
[4-6 key objectives]

## Key Principles
- ONE task per loop — focus on the single most important item
- Search codebase first, don't assume
- Update .ralphss/fix_plan.md then commit
- Detailed specs at .ralphss/specs/requirements.md

## Protected Files (do not modify)
- .ralphss/ directory (fix_plan.md modification allowed)
- .claude/ entire directory

## Testing Guidelines
- Test ratio: ~20% of total work
- Test only new implementations

## Exit Criteria (EXIT_SIGNAL: true)
[Up to 5 concrete criteria. All conditions must be AND true]

## Prohibited Actions
[List out-of-scope actions]

## STATUS Report

Always include this block at the end of every response:

---LOOP_STATUS---
STATUS: IN_PROGRESS | COMPLETE | BLOCKED
EXIT_SIGNAL: false | true
SUMMARY: <one line>
---END_LOOP_STATUS---

## Current Task
Follow .ralphss/fix_plan.md and choose the most important item to implement next.
```

### 2. fix_plan.md — 1 Task = 1 Commit

**Size rule**: 1 task = 1 commit = **max 3-5 file changes**

```markdown
# Fix Plan

## High Priority
- [ ] Task name (list 1-3 target files)

## Medium Priority
- [ ] Task name

## Completed

## Notes
- Reference: specs/requirements.md
```

### 3. specs/requirements.md — Detailed Domain Specs

All domain information not in PROMPT.md.

### 4. AGENT.md — Build/Test Commands

```markdown
# Agent Build Instructions

## Tech Stack
## Build/Run
## Test
## Key Rules
```

### 5. MASTER_PLAN.md — Full Plan (Multi Phase only)

```markdown
# Master Plan

## Phase List
| Phase | Directory | Goal | Depends On |
|-------|-----------|------|------------|

## Overall Completion Criteria
```

### 6. Per-Phase verify.sh (Multi Phase only)

```bash
#!/bin/bash
set -e
# 1-5 verification commands
echo "Phase N verify: PASSED"
exit 0
```

### 7. run.sh — Autonomous Loop Script

**Single Phase:**

```bash
#!/bin/bash
set -uo pipefail

RLOOP_DIR=".ralphss"
MAX_LOOPS="${MAX_LOOPS:-30}"

cleanup() {
  echo ""
  echo ">>> Ctrl+C — stopping"
  pkill -P $$ 2>/dev/null
  rm -f /tmp/ralphss_output.*
  exit 130
}
trap cleanup INT TERM

prompt=$(cat "$RLOOP_DIR/PROMPT.md")
loop_count=0
exit_count=0

while [ $loop_count -lt $MAX_LOOPS ]; do
  loop_count=$((loop_count + 1))
  echo ""
  echo "=========================================="
  echo "[loop $loop_count/$MAX_LOOPS] $(date '+%H:%M:%S')"
  echo "=========================================="

  tmpfile=$(mktemp /tmp/ralphss_output.XXXXXX)

  # Background exec + wait pattern: wait responds to signals immediately
  claude -p "$prompt" --dangerously-skip-permissions | tee "$tmpfile" &
  wait $!
  pipe_exit=$?

  # Exit on SIGINT (wait interrupted by signal)
  if [ $pipe_exit -gt 128 ]; then
    echo ">>> Interrupted (exit=$pipe_exit)"
    rm -f "$tmpfile"
    exit 130
  fi

  if grep -q "EXIT_SIGNAL: true" "$tmpfile"; then
    exit_count=$((exit_count + 1))
    echo ">>> EXIT_SIGNAL ($exit_count/2)"
    if [ $exit_count -ge 2 ]; then
      echo ">>> Complete"
      rm -f "$tmpfile"
      exit 0
    fi
  else
    exit_count=0
  fi

  rm -f "$tmpfile"
  prompt="Check .ralphss/fix_plan.md and execute the most important remaining task. Report EXIT_SIGNAL: true when complete."
done

echo "WARNING: MAX_LOOPS reached"
exit 1
```

**Multi Phase:**

```bash
#!/bin/bash
set -uo pipefail

RLOOP_DIR=".ralphss"
PHASES_DIR="$RLOOP_DIR/phases"
CURRENT_FILE="$RLOOP_DIR/.current_phase"
MAX_LOOPS="${MAX_LOOPS:-30}"

cleanup() {
  echo ""
  echo ">>> Ctrl+C — stopping"
  pkill -P $$ 2>/dev/null
  rm -f /tmp/ralphss_output.*
  exit 130
}
trap cleanup INT TERM

phases=($(ls -d "$PHASES_DIR"/*/ 2>/dev/null | sort))
if [ ${#phases[@]} -eq 0 ]; then
  echo "ERROR: No phases found in $PHASES_DIR"
  exit 1
fi

# Determine resume point (current_phase = completed phase → start from next)
current_phase=""
if [ -f "$CURRENT_FILE" ]; then
  current_phase=$(cat "$CURRENT_FILE")
fi
started=false
[ -z "$current_phase" ] && started=true

for phase_dir in "${phases[@]}"; do
  phase_name=$(basename "$phase_dir")

  if [ "$started" = false ]; then
    if [ "$phase_name" = "$current_phase" ]; then
      started=true
    fi
    continue
  fi

  echo ""
  echo "##############################"
  echo "### Phase: $phase_name"
  echo "##############################"

  prompt=$(cat "$phase_dir/PROMPT.md")
  loop_count=0
  exit_count=0

  while [ $loop_count -lt $MAX_LOOPS ]; do
    loop_count=$((loop_count + 1))
    echo ""
    echo "  [loop $loop_count/$MAX_LOOPS] $(date '+%H:%M:%S')"

    tmpfile=$(mktemp /tmp/ralphss_output.XXXXXX)

    claude -p "$prompt" --dangerously-skip-permissions | tee "$tmpfile" &
    wait $!
    pipe_exit=$?

    if [ $pipe_exit -gt 128 ]; then
      echo "  >>> Interrupted (exit=$pipe_exit)"
      rm -f "$tmpfile"
      exit 130
    fi

    if grep -q "EXIT_SIGNAL: true" "$tmpfile"; then
      exit_count=$((exit_count + 1))
      echo "  >>> EXIT_SIGNAL ($exit_count/2)"
      if [ $exit_count -ge 2 ]; then
        echo "  >>> Phase complete"
        rm -f "$tmpfile"
        break
      fi
    else
      exit_count=0
    fi

    rm -f "$tmpfile"
    prompt="Check .ralphss/fix_plan.md and execute the most important remaining task. Report EXIT_SIGNAL: true when complete."
  done

  # Phase verification
  if [ -f "$phase_dir/verify.sh" ]; then
    echo "--- Verifying $phase_name ---"
    if bash "$phase_dir/verify.sh"; then
      echo "✓ $phase_name PASSED"
      echo "$phase_name" > "$CURRENT_FILE"
    else
      echo "✗ $phase_name FAILED"
      exit 1
    fi
  else
    echo "$phase_name" > "$CURRENT_FILE"
  fi
done

echo ""
echo "=== All phases complete ==="
rm -f "$CURRENT_FILE"
```

---

## Execution Steps

### Common (1-3)
1. Read plan file
2. Understand project state
3. Determine phase count from plan → **2+ phases = multi mode**

### Single Phase Mode
4. `mkdir -p .ralphss/specs`
5. Generate PROMPT.md
6. Generate fix_plan.md
7. Generate specs/requirements.md
8. Generate AGENT.md
9. Generate run.sh (single phase template) + `chmod +x`
10. Output summary
11. **Stop**

### Multi Phase Mode
4. `mkdir -p .ralphss/specs .ralphss/phases`
5. Generate MASTER_PLAN.md
6. Generate per-phase directories + PROMPT.md + fix_plan.md + verify.sh
7. Generate AGENT.md + specs/requirements.md
8. Generate run.sh (multi phase template) + `chmod +x`
9. Output summary
10. **Stop**

## Run (External Terminal)

```bash
cd your-project
bash .ralphss/run.sh
```

Control loop count with environment variable:
```bash
MAX_LOOPS=50 bash .ralphss/run.sh
```

## Prohibited Actions
- No source code modification/deletion
- No test execution
- No git operations
- No file creation/modification outside `.ralphss/`
