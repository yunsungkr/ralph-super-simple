---
name: ralphss-disc
description: Read a topic file and generate Codex/Gemini/Claude round-robin discussion structure. Use with ralphss disc or discussion import.
---

# ralphss-disc

Reads a topic file and generates a Codex → Gemini → Claude round-robin design discussion under `.ralphss/disc/{task_name}/`.

> **This skill only creates files.** No code modification, testing, or git operations.

## Workflow
```
Write topic file → ralphss disc → run bash .ralphss/disc/{task_name}/run.sh in an external terminal
```

## Input
- Topic file (markdown). Must include design questions, context, constraints, and goals.

## task_name Determination

Auto-generated in `{YYYY-MM-DD}_{slug}` format.
- Date: today's date
- slug: topic filename with extension removed, spaces replaced with `-`

## Output
```
.ralphss/disc/{task_name}/
├── run.sh                      # Round runner (fixed)
├── topic.md                    # Original topic (copied)
└── rounds/
    ├── round-1-codex/
    │   └── instruction.md
    ├── round-2-gemini/
    │   └── instruction.md
    ├── round-3-claude/
    │   └── instruction.md
    └── ...
```

At runtime, run.sh additionally creates:
```
rounds/round-N-{tool}/
    ├── instruction.md          # (created on import)
    ├── prompt.md               # Actual prompt (topic + instruction + previous output)
    └── output.md               # Tool response
.ralphss/disc/{task_name}/synthesis.md  # Copy of last round output
```

---

## What Claude Does on Import

### 1. Analyze Topic → Determine Number of Rounds

Determine the number of rounds based on topic complexity and number of questions:
- **3 rounds**: Simple design questions, clear choices (codex → gemini → claude)
- **6 rounds**: Complex architecture, many trade-offs (codex → gemini → claude × 2)

Tool rotation: `codex → gemini → claude → codex → gemini → claude`
The last round is always claude, responsible for synthesis.

### 2. Generate instruction.md per Round

Assign a **debate role** to each round. Core principles:
- **Take a clear position. No neutrality.**
- Must **rebut** the previous round's argument (except the first round).
- Only the last round (claude) delivers a final verdict.

**3-round example (ping-pong debate):**

| Round | Tool | instruction.md content |
|-------|------|------------------------|
| 1 | codex | Analyze the topic and **take a clear position**. 3 core arguments + specific evidence. No neutrality. |
| 2 | gemini | **Attack the weaknesses** of the previous argument and **argue the opposing position**. Specifically point out why it is wrong + propose alternatives. |
| 3 | claude | **Adjudicate** both arguments. State which side is stronger with evidence and deliver a final conclusion. Summarize unresolved issues. |

**6-round example (deep debate):**

| Round | Tool | instruction.md content |
|-------|------|------------------------|
| 1 | codex | Analyze the topic and **take a clear position**. Core arguments + specific evidence. No neutrality. |
| 2 | gemini | **Attack the weaknesses** of the previous argument and **argue the opposing position**. Specifically point out why it is wrong. |
| 3 | claude | Evaluate both arguments and **support the stronger side**, while suggesting improvements. No neutrality. |
| 4 | codex | **Counter-rebut** the previous round's conclusion. Attack overlooked risks, edge cases, and feasibility. |
| 5 | gemini | **Defend or revise and counter** the previous rebuttal. Clearly finalize your position. |
| 6 | claude | **Adjudicate** the entire debate. Final conclusion + evidence + points of agreement + unresolved issues. |

**instruction.md format:**

```markdown
## Round Objective
[Specifically what must be argued, rebutted, or adjudicated]

## Debate Rules
- Take a clear position. No neutrality. No "both options are possible" fence-sitting.
- Previous round output is provided. Must directly quote and rebut it.
- Arguments must include specific evidence (code examples, numbers, cases).

## Output Format
[What sections to produce]

## Constraints
- No file creation or modification. Text output only.
```

### 3. Generate run.sh

Use the template below as-is. Replace `{task_name}` with the actual value. Include `chmod +x`.

```bash
#!/bin/bash
set -uo pipefail

DISC_DIR="$(cd "$(dirname "$0")" && pwd)"
ROUNDS_DIR="$DISC_DIR/rounds"
TOPIC_FILE="$DISC_DIR/topic.md"

if [ ! -f "$TOPIC_FILE" ]; then
  echo "ERROR: $TOPIC_FILE not found"
  exit 1
fi

# jq filter to extract text + tool name from stream-json
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
  echo ">>> Stopping..."
  rm -f /tmp/ralphd_prompt.* /tmp/ralphd_claude.*
  exit 130
}
trap cleanup INT TERM

# Auto-detect round directories
round_dirs=($(ls -d "$ROUNDS_DIR"/round-*/ 2>/dev/null | sort))
total=${#round_dirs[@]}

if [ $total -eq 0 ]; then
  echo "ERROR: No round directories found in $ROUNDS_DIR"
  exit 1
fi

START_TIME=$(date +%s)

echo "╔══════════════════════════════════════════╗"
echo "║       ralphss-disc                       ║"
echo "╚══════════════════════════════════════════╝"
echo ""
echo "  Started: $(date '+%Y-%m-%d %H:%M:%S')"
echo "  Topic:   $(head -1 "$TOPIC_FILE")"
echo "  Rounds:  $total"
echo ""

# Round preview
for round_dir in "${round_dirs[@]}"; do
  dn=$(basename "$round_dir")
  r=$(echo "$dn" | sed 's/round-\([0-9]*\)-.*/\1/')
  t=$(echo "$dn" | sed 's/round-[0-9]*-//')
  goal=$(head -3 "$round_dir/instruction.md" | grep -v '^#' | grep -v '^$' | head -1)
  echo "  [$r] $t — $goal"
done
echo ""
echo "──────────────────────────────────────────"
echo ""

prev_output=""

for round_dir in "${round_dirs[@]}"; do
  dir_name=$(basename "$round_dir")
  round=$(echo "$dir_name" | sed 's/round-\([0-9]*\)-.*/\1/')
  tool=$(echo "$dir_name" | sed 's/round-[0-9]*-//')

  instruction_file="$round_dir/instruction.md"
  if [ ! -f "$instruction_file" ]; then
    echo "ERROR: $instruction_file not found"
    exit 1
  fi

  round_start=$(date +%s)
  goal=$(head -3 "$instruction_file" | grep -v '^#' | grep -v '^$' | head -1)

  echo "▶ [Round $round/$total] $tool"
  echo "  Objective: $goal"
  echo "  Started:   $(date '+%H:%M:%S')"

  # Compose prompt: instruction + topic + previous output
  prompt_file=$(mktemp /tmp/ralphd_prompt.XXXXXX)
  {
    cat "$instruction_file"
    echo ""
    echo "---"
    echo "TOPIC:"
    cat "$TOPIC_FILE"

    if [ -n "$prev_output" ] && [ -f "$prev_output" ]; then
      echo ""
      echo "---"
      echo "PREVIOUS ROUND OUTPUT:"
      cat "$prev_output"
    fi
  } > "$prompt_file"

  cp "$prompt_file" "$round_dir/prompt.md"

  # Execute — branch by tool
  if [ "$tool" = "codex" ]; then
    echo "  [Running codex...]"
    codex exec -o "$round_dir/output.md" \
      -c 'sandbox_permissions=[]' \
      "$(cat "$prompt_file")"

  elif [ "$tool" = "gemini" ]; then
    echo "  [Running gemini...]"
    gemini -p "$(cat "$prompt_file")" --approval-mode yolo 2>/dev/null \
      | grep -v 'YOLO\|credentials' \
      | tee "$round_dir/output.md"

  elif [ "$tool" = "claude" ]; then
    echo "  [Running claude...]"
    tmpfile=$(mktemp /tmp/ralphd_claude.XXXXXX)
    set +e
    DISABLE_OMC=1 stdbuf -oL claude -p "$(cat "$prompt_file")" \
      --dangerously-skip-permissions \
      --output-format stream-json --verbose --include-partial-messages \
      < /dev/null 2>&1 \
      | stdbuf -oL tee "$tmpfile" \
      | stdbuf -oL jq --unbuffered -j "$JQ_FILTER" 2>/dev/null
    set -e
    # Extract final text only from stream-json and save to output.md
    jq -j '
      if .type == "stream_event" then
        if .event.type == "content_block_delta" and .event.delta.type == "text_delta" then
          .event.delta.text
        else
          empty
        end
      else
        empty
      end' "$tmpfile" 2>/dev/null | tr -d '\0' > "$round_dir/output.md"
    rm -f "$tmpfile"

  else
    echo "  ERROR: Unknown tool '$tool'"
    rm -f "$prompt_file"
    exit 1
  fi

  rm -f "$prompt_file"
  prev_output="$round_dir/output.md"

  round_end=$(date +%s)
  round_elapsed=$(( round_end - round_start ))
  output_lines=$(wc -l < "$round_dir/output.md" 2>/dev/null || echo 0)

  echo "  Done:   $(date '+%H:%M:%S') (${round_elapsed}s)"
  echo "  Output: $round_dir/output.md (${output_lines} lines)"
  echo ""
  echo "──────────────────────────────────────────"
  echo ""
  sleep 2
done

# Copy last round output as synthesis
last_dir="${round_dirs[$((total-1))]}"
cp "$last_dir/output.md" "$DISC_DIR/synthesis.md" 2>/dev/null

END_TIME=$(date +%s)
TOTAL_ELAPSED=$(( END_TIME - START_TIME ))
TOTAL_MIN=$(( TOTAL_ELAPSED / 60 ))
TOTAL_SEC=$(( TOTAL_ELAPSED % 60 ))

echo "╔══════════════════════════════════════════╗"
echo "║       Discussion Complete                ║"
echo "╚══════════════════════════════════════════╝"
echo ""
echo "  Finished: $(date '+%Y-%m-%d %H:%M:%S')"
echo "  Duration: ${TOTAL_MIN}m ${TOTAL_SEC}s"
echo "  Result:   $DISC_DIR/synthesis.md"
echo ""
```

---

## Execution Steps

1. Read the topic file
2. Analyze topic complexity → determine number of rounds (3 or 6)
3. Determine task_name (`YYYY-MM-DD_slug`)
4. `mkdir -p .ralphss/disc/{task_name}/rounds/round-{N}-{codex|gemini|claude}`
5. Copy `topic.md`
6. Generate `instruction.md` per round (assign progressive objectives)
7. Generate `run.sh` (template above, replace `{task_name}`) + `chmod +x`
8. Print result summary (format below)
9. **Stop**

### Result Summary Format

After import, always output in the following format:

```
Discussion import complete.

**Topic**: [First line of topic.md (# title)]
**Rounds**: [N] rounds (one-line rationale for complexity decision)

| Round | Tool | Objective |
|-------|------|-----------|
| 1 | codex | [one-line summary of instruction] |
| 2 | gemini | [one-line summary of instruction] |
| 3 | claude | [one-line summary of instruction] |
| ... | ... | ... |

**Generated files**:
.ralphss/disc/{task_name}/
├── run.sh (chmod +x)
├── topic.md
└── rounds/
    ├── round-1-codex/instruction.md
    ├── round-2-gemini/instruction.md
    ├── round-3-claude/instruction.md
    └── ...

**Run**: `bash .ralphss/disc/{task_name}/run.sh`
```

## How to Run (External Terminal)

```bash
cd your-project
bash .ralphss/disc/{task_name}/run.sh
```

Cleanup:
```bash
rm -rf .ralphss/disc/{task_name}/
```

## Tool Execution Environment

| Tool | How to run | Notes |
|------|------------|-------|
| codex | `codex exec -o output.md -c 'sandbox_permissions=[]'` | File exploration blocked |
| gemini | `gemini -p --approval-mode yolo` | Filters YOLO/credentials noise |
| claude | `DISABLE_OMC=1 claude -p --dangerously-skip-permissions --output-format text` | Must disable OMC hooks |

> **Note when running claude**: The `DISABLE_OMC=1` environment variable must be set to disable OMC hooks.
> This prevents project terms in the prompt from conflicting with OMC keywords and triggering false skill invocations.

## Prohibited Actions
- No source code modification or deletion
- No test execution
- No git operations
- No file creation or modification outside `.ralphss/disc/{task_name}/`
