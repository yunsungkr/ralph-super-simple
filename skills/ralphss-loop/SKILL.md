---
name: ralphss-loop
description: Distribute an implementation plan into .ralphss/loop/ file structure. Use with ralphss loop.
---

# ralphss-loop

Reads an implementation plan file and distributes it into a `.ralphss/loop/{task_name}/` file structure for autonomous claude CLI loops.
Supports multi-phase repeated execution using only `claude -p`, with no external tool dependencies.

> **This skill only creates files.** Do not modify code, run tests, or manipulate git.

<!-- NOTE: Known constraints
  - `claude -p` starts a new session on every call, so even simple errors can interrupt the session and lose context
  - Session continuity via `--resume <session_id>` enables self-recovery (as in ralphcc)
  - Currently, when used in an OMC environment, OMC's persistence loop compensates for this
  - Future: consider adding session management (--resume) to run.sh
-->

## Workflow
```
Write plan file → ralphss loop → run bash .ralphss/loop/{task_name}/run.sh in an external terminal
```

## Input
- An implementation plan file (markdown). Must include exit conditions, per-phase tasks, and technical constraints.

> **Tip**: Plans should be broken into clearly defined phases before importing. Monolithic plans will be treated as a single phase.

## task_name Determination

Auto-generated in `{YYYY-MM-DD}_{slug}` format.
- Date: today's date
- slug: plan filename with extension removed and spaces replaced by `-` (e.g. `di-migration-plan.md` → `2026-03-10_di-migration-plan`)
- If the plan filename is generic (e.g. `plan.md`), extract the slug from the first heading in the plan

## Output

### Single Phase (default)
```
.ralphss/loop/{task_name}/
├── run.sh                 # Autonomous loop execution script
├── PROMPT.md              # Loop control instructions
├── fix_plan.md            # Commit-unit task checklist
├── AGENT.md               # Build/test commands
└── specs/
    └── requirements.md    # Detailed domain specs
```

### Multi-Phase (when plan has 2 or more phases)
```
.ralphss/loop/{task_name}/
├── run.sh                   # Phase auto-transition + autonomous loop
├── phases/
│   ├── phase-1/
│   │   ├── PROMPT.md        # Per-phase loop control
│   │   ├── fix_plan.md      # Per-phase tasks
│   │   └── verify.sh        # Per-phase verification (exit 0 = pass)
│   ├── phase-2/
│   │   └── ...
│   └── ...
├── MASTER_PLAN.md           # Overall plan (reference)
├── AGENT.md                 # Shared build/test commands
└── specs/
    └── requirements.md      # Shared detailed specs
```

---

## File Generation Rules

### 1. PROMPT.md — Loop Control Only

**Principle**: Domain knowledge goes in `specs/requirements.md`. PROMPT.md contains only loop control information.

**Required sections** (in this order):

```markdown
# [Project/Task Name]

## Context
[3–5 lines. Project name, one-line description, current state]

## Current Objectives
[4–6 core objectives]

## Key Principles
- ONE task per loop — focus on the single most important thing
- Search the codebase first, never assume
- Update .ralphss/loop/{task_name}/fix_plan.md before committing
- Refer to .ralphss/loop/{task_name}/specs/requirements.md for detailed specs

## Protected Files (do not modify)
- .ralphss/ directory (fix_plan.md is the only exception)
- .claude/ entire directory

## Testing Guidelines
- Tests should account for ~20% of total work
- Only test newly implemented functionality

## Exit Conditions (EXIT_SIGNAL: true)
[Up to 5 concrete criteria. Only true when ALL conditions are met]

## Prohibited Actions
[List of out-of-scope actions]

## STATUS Report

Always include the following block at the end of each response:

---LOOP_STATUS---
STATUS: IN_PROGRESS | COMPLETE | BLOCKED
EXIT_SIGNAL: false | true
SUMMARY: <one line>
---END_LOOP_STATUS---

## Current Task
Follow .ralphss/loop/{task_name}/fix_plan.md and choose the most important item to implement next.
```

### 2. fix_plan.md — 1 Task = 1 Commit

**Size rule**: 1 task = 1 commit = **max 3–5 file changes**

```markdown
# Fix Plan

## High Priority
- [ ] Task name (specify 1–3 target files)

## Medium Priority
- [ ] Task name

## Completed

## Notes
- Reference: specs/requirements.md
```

### 3. specs/requirements.md — Detailed Domain Specs

All domain information that is not in PROMPT.md.

### 4. AGENT.md — Build/Test Commands

```markdown
# Agent Build Instructions

## Tech Stack
## Build / Run
## Tests
## Core Rules
```

### 5. MASTER_PLAN.md — Overall Plan (multi-phase only)

```markdown
# Master Plan

## Phase List
| Phase | Directory | Goal | Dependencies |
|-------|-----------|------|--------------|

## Overall Completion Criteria
```

### 6. Per-Phase verify.sh (multi-phase only)

```bash
#!/bin/bash
set -e
# 1–5 verification commands
echo "Phase N verify: PASSED"
exit 0
```

### 7. run.sh — Autonomous Loop Execution Script

**Single Phase:**

```bash
#!/bin/bash
set -uo pipefail

RLOOP_DIR="$(cd "$(dirname "$0")" && pwd)"
LOG_DIR="$RLOOP_DIR/logs"
MAX_LOOPS="${MAX_LOOPS:-30}"

mkdir -p "$LOG_DIR"

# jq filter to extract text + tool names from stream-json
JQ_FILTER='
  if .type == "stream_event" then
    if .event.type == "content_block_delta" and .event.delta.type == "text_delta" then
      .event.delta.text
    elif .event.type == "content_block_start" and .event.content_block.type == "tool_use" then
      "\n[" + .event.content_block.name + "]\n"
    elif .event.type == "content_block_stop" then
      "\n"
    else
      empty
    end
  else
    empty
  end'

cleanup() {
  echo ""
  echo ">>> Ctrl+C — cleaning up..."
  if [ -n "${tmpfile:-}" ] && [ -f "${tmpfile:-}" ]; then
    local log="$LOG_DIR/loop-$(printf '%02d' "${loop_count:-0}").log"
    {
      echo "=== INTERRUPTED — $(date '+%Y-%m-%d %H:%M:%S') ==="
      cat "$tmpfile"
    } > "$log"
    echo ">>> Last output saved: $log"
    rm -f "$tmpfile"
  fi
  echo "=== Aborted: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
  rm -f /tmp/ralphss_output.*
  exit 130
}
trap cleanup SIGINT SIGTERM

prompt=$(cat "$RLOOP_DIR/PROMPT.md")
loop_count=0
exit_count=0

echo "=== ralphss-loop started: $(date '+%Y-%m-%d %H:%M:%S') ===" | tee "$LOG_DIR/run.log"

while [ $loop_count -lt $MAX_LOOPS ]; do
  loop_count=$((loop_count + 1))
  echo ""
  echo "=========================================="
  echo "[loop $loop_count/$MAX_LOOPS] $(date '+%H:%M:%S')"
  echo "=========================================="

  tmpfile=$(mktemp /tmp/ralphss_output.XXXXXX)

  # Real-time streaming: stream-json + stdbuf + jq
  set +e
  stdbuf -oL claude -p "$prompt" --dangerously-skip-permissions \
    --output-format stream-json --verbose --include-partial-messages \
    < /dev/null 2>&1 \
    | stdbuf -oL tee "$tmpfile" \
    | stdbuf -oL jq --unbuffered -j "$JQ_FILTER" 2>/dev/null
  pipe_exit=${PIPESTATUS[0]}
  set -e

  if [ $pipe_exit -gt 128 ]; then
    echo ">>> Interrupted (exit=$pipe_exit)"
    rm -f "$tmpfile"
    exit 130
  fi

  # Save loop log
  {
    echo "=== loop $loop_count — $(date '+%Y-%m-%d %H:%M:%S') ==="
    cat "$tmpfile"
    echo ""
  } > "$LOG_DIR/loop-$(printf '%02d' $loop_count).log"

  if grep -q "EXIT_SIGNAL: true" "$tmpfile"; then
    exit_count=$((exit_count + 1))
    echo ">>> EXIT_SIGNAL ($exit_count/2)"
    if [ $exit_count -ge 2 ]; then
      echo ">>> Done"
      echo "=== Completed: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
      rm -f "$tmpfile"
      exit 0
    fi
  else
    exit_count=0
  fi

  rm -f "$tmpfile"
  sleep 1
  prompt="Check $RLOOP_DIR/fix_plan.md and perform the most important remaining task. Report EXIT_SIGNAL: true when done."
done

echo "WARNING: MAX_LOOPS reached"
echo "=== MAX_LOOPS reached: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
exit 1
```

**Multi-Phase:**

```bash
#!/bin/bash
set -uo pipefail

RLOOP_DIR="$(cd "$(dirname "$0")" && pwd)"
PHASES_DIR="$RLOOP_DIR/phases"
LOG_DIR="$RLOOP_DIR/logs"
CURRENT_FILE="$RLOOP_DIR/.current_phase"
MAX_LOOPS="${MAX_LOOPS:-30}"

mkdir -p "$LOG_DIR"

# jq filter to extract text + tool names from stream-json
JQ_FILTER='
  if .type == "stream_event" then
    if .event.type == "content_block_delta" and .event.delta.type == "text_delta" then
      .event.delta.text
    elif .event.type == "content_block_start" and .event.content_block.type == "tool_use" then
      "\n[" + .event.content_block.name + "]\n"
    elif .event.type == "content_block_stop" then
      "\n"
    else
      empty
    end
  else
    empty
  end'

cleanup() {
  echo ""
  echo ">>> Ctrl+C — cleaning up..."
  if [ -n "${tmpfile:-}" ] && [ -f "${tmpfile:-}" ]; then
    local log="$LOG_DIR/${current_phase:-unknown}-loop-$(printf '%02d' "${loop_count:-0}").log"
    {
      echo "=== INTERRUPTED — $(date '+%Y-%m-%d %H:%M:%S') ==="
      cat "$tmpfile"
    } > "$log"
    echo ">>> Last output saved: $log"
    rm -f "$tmpfile"
  fi
  echo ">>> Aborted at phase: ${current_phase:-unknown}, loop: ${loop_count:-0}"
  echo "=== Aborted: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
  rm -f /tmp/ralphss_output.*
  exit 130
}
trap cleanup SIGINT SIGTERM

phases=($(ls -d "$PHASES_DIR"/*/ 2>/dev/null | sort))
if [ ${#phases[@]} -eq 0 ]; then
  echo "ERROR: No phases found in $PHASES_DIR"
  exit 1
fi

# Determine resume point
current_phase=""
if [ -f "$CURRENT_FILE" ]; then
  current_phase=$(cat "$CURRENT_FILE")
fi
started=false
[ -z "$current_phase" ] && started=true

echo "=== ralphss-loop multi-phase started: $(date '+%Y-%m-%d %H:%M:%S') ===" | tee "$LOG_DIR/run.log"
echo "Phases: ${#phases[@]}" | tee -a "$LOG_DIR/run.log"

for phase_dir in "${phases[@]}"; do
  current_phase=$(basename "$phase_dir")

  if [ "$started" = false ]; then
    if [ "$current_phase" = "$(cat "$CURRENT_FILE")" ]; then
      started=true
    fi
    continue
  fi

  echo ""
  echo "##############################"
  echo "### Phase: $current_phase"
  echo "##############################"
  echo "[Phase: $current_phase] $(date '+%H:%M:%S')" >> "$LOG_DIR/run.log"

  # Symlink per-phase fix_plan.md to root
  ln -sf "phases/$current_phase/fix_plan.md" "$RLOOP_DIR/fix_plan.md"

  prompt=$(cat "$phase_dir/PROMPT.md")
  loop_count=0
  exit_count=0

  while [ $loop_count -lt $MAX_LOOPS ]; do
    loop_count=$((loop_count + 1))
    echo ""
    echo "  [$current_phase | loop $loop_count/$MAX_LOOPS] $(date '+%H:%M:%S')"

    tmpfile=$(mktemp /tmp/ralphss_output.XXXXXX)
    phase_log="$LOG_DIR/${current_phase}-loop-$(printf '%02d' $loop_count).log"

    # Real-time streaming: stream-json + stdbuf + jq
    set +e
    stdbuf -oL claude -p "$prompt" --dangerously-skip-permissions \
      --output-format stream-json --verbose --include-partial-messages \
      < /dev/null 2>&1 \
      | stdbuf -oL tee "$tmpfile" \
      | stdbuf -oL jq --unbuffered -j "$JQ_FILTER" 2>/dev/null
    pipe_exit=${PIPESTATUS[0]}
    set -e

    if [ $pipe_exit -gt 128 ]; then
      echo ""
      echo "  >>> Interrupted (exit=$pipe_exit)"
      rm -f "$tmpfile"
      exit 130
    fi

    echo ""

    # Save loop log
    {
      echo "=== $current_phase loop $loop_count — $(date '+%Y-%m-%d %H:%M:%S') ==="
      cat "$tmpfile"
      echo ""
    } > "$phase_log"

    if grep -q "EXIT_SIGNAL: true" "$tmpfile"; then
      exit_count=$((exit_count + 1))
      echo "  >>> EXIT_SIGNAL ($exit_count/2)"
      echo "  [$current_phase loop $loop_count] EXIT_SIGNAL ($exit_count/2)" >> "$LOG_DIR/run.log"
      if [ $exit_count -ge 2 ]; then
        echo "  >>> Phase loop complete"
        rm -f "$tmpfile"
        break
      fi
    else
      exit_count=0
    fi

    rm -f "$tmpfile"
    sleep 1
    prompt="Check $RLOOP_DIR/phases/$current_phase/fix_plan.md and perform the most important remaining task. Report EXIT_SIGNAL: true when done."
  done

  # Phase verification
  if [ -f "$phase_dir/verify.sh" ]; then
    echo ""
    echo "--- Verifying $current_phase ---"
    if bash "$phase_dir/verify.sh"; then
      echo "  $current_phase PASSED"
      echo "$current_phase" > "$CURRENT_FILE"
      echo "  [$current_phase] VERIFIED" >> "$LOG_DIR/run.log"
    else
      echo "  $current_phase FAILED"
      echo "  [$current_phase] VERIFY FAILED" >> "$LOG_DIR/run.log"
      exit 1
    fi
  else
    echo "$current_phase" > "$CURRENT_FILE"
  fi
done

echo ""
echo "=== All phases complete: $(date '+%Y-%m-%d %H:%M:%S') ==="
echo "=== Completed: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
rm -f "$CURRENT_FILE"
```

---

## Execution Procedure

### Common Steps (1–4)
1. Read the plan file
2. Understand the current project state
3. Determine task_name (`YYYY-MM-DD_slug`)
4. Count phases in the plan → **2 or more means multi-phase mode**

### Single Phase Mode
5. `mkdir -p .ralphss/loop/{task_name}/specs`
6. Create PROMPT.md (substitute `{task_name}` in paths)
7. Create fix_plan.md
8. Create specs/requirements.md
9. Create AGENT.md
10. Create run.sh (single phase template, substitute `{task_name}`) + `chmod +x`
11. Print result summary
12. **Done**

### Multi-Phase Mode
5. `mkdir -p .ralphss/loop/{task_name}/specs .ralphss/loop/{task_name}/phases`
6. Create MASTER_PLAN.md
7. Create per-phase directories + PROMPT.md + fix_plan.md + verify.sh
8. Create AGENT.md + specs/requirements.md
9. Create run.sh (multi-phase template, substitute `{task_name}`) + `chmod +x`
10. Print result summary
11. **Done**

## How to Run (external terminal)

```bash
cd your-project
bash .ralphss/loop/{task_name}/run.sh
```

Control loop count with an environment variable:
```bash
MAX_LOOPS=50 bash .ralphss/loop/{task_name}/run.sh
```

Clean up:
```bash
rm -rf .ralphss/loop/{task_name}/
```

## Prohibited Actions
- Do not modify or delete source code
- Do not run tests
- Do not manipulate git
- Do not create or modify files outside `.ralphss/loop/{task_name}/`
