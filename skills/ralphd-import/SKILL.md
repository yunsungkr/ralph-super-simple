---
name: ralphd-import
description: Set up a Codex vs Gemini design discussion. Use for discussion import.
---

# ralphd-import

Reads a design topic and creates a `.discussion/` file structure for alternating Codex ↔ Gemini design discussions. No external tools required — runs with `codex exec` and `gemini -p` only.

> **This skill only creates files.** No code modification, testing, or git operations.

## Workflow
```
Write topic → ralphd import → Run bash .discussion/run.sh in external terminal
```

## Input
- Design topic file (markdown). Should describe the design question, context, constraints, and specific questions to resolve.

## Output
```
.discussion/
├── run.sh              # Automated discussion runner
├── topic.md            # Design topic (from input)
└── rounds/             # Created by run.sh during execution
    ├── round-1-codex/
    │   ├── prompt.md   # Prompt sent to codex
    │   └── output.md   # Codex response
    ├── round-2-gemini/
    │   ├── prompt.md
    │   └── output.md
    ├── round-3-codex/
    │   └── ...
    ├── round-4-gemini/
    │   └── ...
    ├── round-5-codex/  # Synthesis round
    │   └── ...
    └── synthesis.md    # Final design document (copy of round 5)
```

---

## Discussion Rounds

| Round | Tool   | Role        | Goal                                      |
|-------|--------|-------------|-------------------------------------------|
| 1     | codex  | architect   | Analyze topic, propose initial design      |
| 2     | gemini | critic      | Find weaknesses, propose alternatives      |
| 3     | codex  | architect   | Refine design based on critique            |
| 4     | gemini | critic      | Final review, remaining concerns           |
| 5     | codex  | synthesizer | Merge all discussion into final document   |

Each round receives the full accumulated discussion as context.

---

## File Generation Rules

### 1. topic.md — Design Question

Copy the input topic file as-is. Should include:

```markdown
# [Design Topic]

## Question
[Core design question to resolve]

## Context
[Background, constraints, current state]

## Current Proposal (optional)
[If a draft design exists]

## Design Questions
[Specific questions for the discussion to address]
```

### 2. run.sh — Discussion Runner

```bash
#!/bin/bash
set -uo pipefail

DISC_DIR=".discussion"
ROUNDS_DIR="$DISC_DIR/rounds"
TOPIC_FILE="$DISC_DIR/topic.md"
MAX_ROUNDS="${MAX_ROUNDS:-5}"

if [ ! -f "$TOPIC_FILE" ]; then
  echo "ERROR: $TOPIC_FILE not found"
  exit 1
fi

cleanup() {
  echo ""
  echo ">>> Stopping..."
  rm -f /tmp/ralphd_prompt.*
  exit 130
}
trap cleanup INT TERM

get_tool() {
  if [ $(($1 % 2)) -eq 1 ]; then echo "codex"; else echo "gemini"; fi
}

get_role() {
  case $1 in
    1) echo "architect" ;;
    2) echo "critic" ;;
    3) echo "architect" ;;
    4) echo "critic" ;;
    *) echo "synthesizer" ;;
  esac
}

accumulated() {
  local up_to=$1
  for i in $(seq 1 $up_to); do
    local f="$ROUNDS_DIR/round-${i}-$(get_tool $i)/output.md"
    if [ -f "$f" ]; then
      echo ""
      echo "### Round $i ($(get_tool $i) as $(get_role $i)):"
      cat "$f"
      echo ""
    fi
  done
}

build_prompt() {
  local round=$1
  local role=$(get_role $round)
  local tmpfile=$(mktemp /tmp/ralphd_prompt.XXXXXX)

  local instruction=""
  case $role in
    architect)
      if [ $round -eq 1 ]; then
        instruction="You are a software architect. Analyze the design topic below and propose a concrete design.

Output structured markdown: ## Analysis, ## Proposed Design, ## Key Decisions, ## Trade-offs.
Do not create or modify any files. Text output only."
      else
        instruction="You are a software architect. Refine your design based on the critique. Address each point raised.

Output structured markdown: ## Refined Design, ## Addressed Critiques, ## Remaining Trade-offs.
Do not create or modify any files. Text output only."
      fi
      ;;
    critic)
      instruction="You are a design critic. Review the latest proposal. Find weaknesses, missing considerations, and propose alternatives. Be specific and constructive.

Output structured markdown: ## Strengths, ## Weaknesses, ## Missing Considerations, ## Alternative Approaches.
Do not create or modify any files. Text output only."
      ;;
    synthesizer)
      instruction="Synthesize the entire discussion into a final design document. Include decisions with rationale.

Output structured markdown: ## Final Design, ## Key Decisions, ## Consensus Points, ## Open Questions.
Do not create or modify any files. Text output only."
      ;;
  esac

  {
    echo "$instruction"
    echo ""
    echo "---"
    echo "TOPIC:"
    cat "$TOPIC_FILE"

    if [ $round -gt 1 ]; then
      echo ""
      echo "---"
      echo "DISCUSSION SO FAR:"
      accumulated $((round - 1))
    fi
  } > "$tmpfile"

  echo "$tmpfile"
}

echo "=== ralph-discussion start: $(date '+%Y-%m-%d %H:%M:%S') ==="
echo "Topic: $(head -1 "$TOPIC_FILE")"
echo "Rounds: $MAX_ROUNDS"
echo ""

for round in $(seq 1 $MAX_ROUNDS); do
  tool=$(get_tool $round)
  role=$(get_role $round)
  round_dir="$ROUNDS_DIR/round-${round}-${tool}"
  mkdir -p "$round_dir"

  echo "=============================="
  echo "[Round $round/$MAX_ROUNDS] $tool ($role) — $(date '+%H:%M:%S')"
  echo "=============================="

  prompt_file=$(build_prompt $round)
  cp "$prompt_file" "$round_dir/prompt.md"

  if [ "$tool" = "codex" ]; then
    codex exec < "$prompt_file" 2>/dev/null | tee "$round_dir/output.md"
  else
    gemini -p "$(cat "$prompt_file")" --approval-mode yolo 2>/dev/null | tee "$round_dir/output.md"
  fi

  rm -f "$prompt_file"

  echo ""
  echo ">>> Round $round saved: $round_dir/output.md"
  echo ""
  sleep 2
done

final_tool=$(get_tool $MAX_ROUNDS)
cp "$ROUNDS_DIR/round-${MAX_ROUNDS}-${final_tool}/output.md" "$DISC_DIR/synthesis.md" 2>/dev/null

echo "=== Discussion complete: $(date '+%Y-%m-%d %H:%M:%S') ==="
echo "Synthesis: $DISC_DIR/synthesis.md"
echo "All rounds: $ROUNDS_DIR/"
```

---

## Execution Steps

1. Read topic file
2. `mkdir -p .discussion`
3. Copy topic to `.discussion/topic.md`
4. Generate `run.sh` (template above) + `chmod +x`
5. Output summary
6. **Stop**

## Run (External Terminal)

```bash
cd your-project
bash .discussion/run.sh
```

Control round count with environment variable:
```bash
MAX_ROUNDS=7 bash .discussion/run.sh
```

## Prohibited Actions
- No source code modification/deletion
- No test execution
- No git operations
- No file creation/modification outside `.discussion/`
