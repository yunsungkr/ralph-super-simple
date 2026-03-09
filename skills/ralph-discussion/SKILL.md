---
name: ralph-discussion
description: Reads a topic file and generates a Codex ↔ Gemini alternating discussion structure. Use with "discussion import".
---

# ralph-discussion

Reads a topic file and generates a `.discussion/` structure for Codex ↔ Gemini alternating design discussions.

> **This skill only creates files.** No code modification, testing, or git operations.

## Workflow
```
Write topic file → invoke ralph-discussion → run `bash .discussion/run.sh` in external terminal
```

## Input
- A topic file (markdown). Should contain design questions, context, constraints, and desired outcomes.

## Output
```
.discussion/
├── run.sh                      # Runner (fixed — iterates rounds)
├── topic.md                    # Original topic (copied)
└── rounds/
    ├── round-1-codex/
    │   └── instruction.md      # Round instruction (generated on import)
    ├── round-2-gemini/
    │   └── instruction.md
    ├── round-3-codex/
    │   └── instruction.md
    └── ...
```

At runtime, run.sh additionally creates:
```
rounds/round-N-{tool}/
    ├── instruction.md          # (generated on import)
    ├── prompt.md               # Actual prompt sent (topic + instruction + previous output)
    └── output.md               # Tool response
.discussion/synthesis.md        # Copy of last round's output
```

---

## What Claude does on import

### 1. Analyze topic → Determine round count

Determine the number of rounds based on topic complexity and number of questions:
- **3 rounds**: Simple design questions, clear options
- **5 rounds**: Complex architecture, many tradeoffs

Odd rounds = codex, even rounds = gemini. The last round is always synthesis.

### 2. Generate instruction.md for each round

Assign a **specific objective** to each round. Not just "critique" or "revise" — a structure that deepens progressively.

**3-round example:**

| Round | Tool | instruction.md content |
|-------|------|----------------------|
| 1 | codex | Problem analysis + structural proposal + identify 3 key decisions |
| 2 | gemini | Risk analysis for each decision + propose alternatives + comparison table |
| 3 | codex | Discussion synthesis — final decisions + rationale + unresolved items |

**5-round example:**

| Round | Tool | instruction.md content |
|-------|------|----------------------|
| 1 | codex | Problem analysis + structural proposal + identify key decisions |
| 2 | gemini | Risk analysis for each decision + propose alternatives |
| 3 | codex | Build comparison table + document selection rationale |
| 4 | gemini | Final review — missed edge cases, scalability, operational concerns |
| 5 | codex | Discussion synthesis — final design + consensus + unresolved items |

**instruction.md format:**

```markdown
## Round Objective
[Specific description of what to do]

## Output Format
[Which sections to produce]

## Constraints
- No file creation/modification. Text output only.
- Previous round output is provided. You must reference it.
```

### 3. Generate run.sh

Use the template below as-is. Include `chmod +x`.

```bash
#!/bin/bash
set -uo pipefail

DISC_DIR=".discussion"
ROUNDS_DIR="$DISC_DIR/rounds"
TOPIC_FILE="$DISC_DIR/topic.md"

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

# Auto-detect round directories
round_dirs=($(ls -d "$ROUNDS_DIR"/round-*/ 2>/dev/null | sort))
total=${#round_dirs[@]}

if [ $total -eq 0 ]; then
  echo "ERROR: No round directories found in $ROUNDS_DIR"
  exit 1
fi

START_TIME=$(date +%s)

echo "╔══════════════════════════════════════════╗"
echo "║       ralph-discussion                   ║"
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

  # Execute
  if [ "$tool" = "codex" ]; then
    echo "  [Running codex...]"
    codex exec -o "$round_dir/output.md" \
      -c 'sandbox_permissions=[]' \
      "$(cat "$prompt_file")"
  else
    echo "  [Running gemini...]"
    gemini -p "$(cat "$prompt_file")" --approval-mode yolo 2>/dev/null | grep -v 'YOLO\|credentials' | tee "$round_dir/output.md"
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
2. Analyze topic complexity → determine discussion rounds (3 or 5)
3. `mkdir -p .discussion/rounds/round-{N}-{codex|gemini}`
4. Copy `topic.md`
5. Generate `instruction.md` per round (progressive specific objectives)
6. Generate `run.sh` (template above) + `chmod +x`
7. Print result summary (format below)
8. **Done**

### Result Summary Format

After import, always output in this format:

```
Discussion import complete.

**Topic**: [First line of topic.md (# title)]
**Rounds**: [N] rounds (1-line complexity rationale)

| Round | Tool | Objective |
|-------|------|-----------|
| 1 | codex | [1-line instruction summary] |
| 2 | gemini | [1-line instruction summary] |
| ... | ... | ... |

**Generated files**:
.discussion/
├── run.sh (chmod +x)
├── topic.md
└── rounds/
    ├── round-1-codex/instruction.md
    ├── round-2-gemini/instruction.md
    └── ...

**Run**: `bash .discussion/run.sh`
```

## How to Run (external terminal)

```bash
cd your-project
bash .discussion/run.sh
```

## Prohibited Actions
- No source code modification/deletion
- No test execution
- No git operations
- No file creation/modification outside `.discussion/`
