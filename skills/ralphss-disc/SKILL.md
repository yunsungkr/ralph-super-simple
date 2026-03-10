---
name: ralphss-disc
description: 토픽 파일을 읽고 Codex/Gemini/Claude 순환 토론 구조를 생성. ralphss disc, discussion import 시 사용.
---

# ralphss-disc

토픽 파일을 읽고 `.ralphss/disc/{task_name}/` 구조로 Codex → Gemini → Claude 순환 설계 토론을 생성한다.

> **이 스킬은 파일 생성만 수행한다.** 코드 수정, 테스트, git 조작 금지.

## 워크플로우
```
토픽 파일 작성 → ralphss disc → 외부 터미널에서 bash .ralphss/disc/{task_name}/run.sh
```

## 입력
- 토픽 파일 (마크다운). 설계 질문, 컨텍스트, 제약, 목표가 포함되어야 함.

## task_name 결정

`{YYYY-MM-DD}_{slug}` 형식으로 자동 생성한다.
- 날짜: 오늘 날짜
- slug: 토픽 파일명에서 확장자를 제거하고 공백을 `-`로 치환

## 출력
```
.ralphss/disc/{task_name}/
├── run.sh                      # 라운드 실행기 (고정)
├── topic.md                    # 원본 토픽 (복사)
└── rounds/
    ├── round-1-codex/
    │   └── instruction.md
    ├── round-2-gemini/
    │   └── instruction.md
    ├── round-3-claude/
    │   └── instruction.md
    └── ...
```

런타임에 run.sh가 추가 생성:
```
rounds/round-N-{tool}/
    ├── instruction.md          # (import 시 생성)
    ├── prompt.md               # 실제 프롬프트 (토픽 + instruction + 이전 output)
    └── output.md               # 도구 응답
.ralphss/disc/{task_name}/synthesis.md  # 마지막 라운드 output 복사본
```

---

## import 시 Claude가 수행하는 작업

### 1. 토픽 분석 → 라운드 수 결정

토픽 복잡도와 질문 수에 따라 라운드 수를 결정:
- **3 라운드**: 단순한 설계 질문, 명확한 선택지 (codex → gemini → claude)
- **6 라운드**: 복잡한 아키텍처, 다수의 트레이드오프 (codex → gemini → claude × 2회)

도구 순환: `codex → gemini → claude → codex → gemini → claude`
마지막 라운드는 항상 claude가 synthesis를 담당한다.

### 2. 라운드별 instruction.md 생성

각 라운드에 **구체적 목표**를 부여한다. 점진적으로 깊어지는 구조.

**3라운드 예시:**

| 라운드 | 도구 | instruction.md 내용 |
|--------|------|---------------------|
| 1 | codex | 문제 분석 + 구조 제안 + 핵심 결정사항 3개 도출 |
| 2 | gemini | 각 결정에 대한 리스크 분석 + 대안 제시 + 비교표 |
| 3 | claude | 토론 종합 — 최종 결정 + 근거 + 미해결 사항 |

**6라운드 예시:**

| 라운드 | 도구 | instruction.md 내용 |
|--------|------|---------------------|
| 1 | codex | 문제 분석 + 구조 제안 + 핵심 결정사항 도출 |
| 2 | gemini | 각 결정에 대한 리스크 분석 + 대안 제시 |
| 3 | claude | 비교표 작성 + 선택 근거 문서화 |
| 4 | codex | 추천안의 구체적 구현 설계 + 마이그레이션 경로 |
| 5 | gemini | 최종 검토 — 엣지 케이스, 확장성, 운영 이슈 |
| 6 | claude | 토론 종합 — 최종 설계 + ADR + 마이그레이션 로드맵 |

**instruction.md 형식:**

```markdown
## Round Objective
[구체적으로 무엇을 해야 하는지]

## Output Format
[어떤 섹션을 산출해야 하는지]

## Constraints
- 파일 생성/수정 금지. 텍스트 출력만.
- 이전 라운드 출력이 제공됨. 반드시 참조할 것.
```

### 3. run.sh 생성

아래 템플릿을 그대로 사용. `{task_name}`을 실제 값으로 치환. `chmod +x` 포함.

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

# stream-json에서 텍스트 + 도구명 추출하는 jq 필터
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

  # Execute — tool별 분기
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
    # stream-json에서 최종 텍스트만 추출하여 output.md 저장
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

## 실행 절차

1. 토픽 파일 읽기
2. 토픽 복잡도 분석 → 라운드 수 결정 (3 또는 6)
3. task_name 결정 (`YYYY-MM-DD_slug`)
4. `mkdir -p .ralphss/disc/{task_name}/rounds/round-{N}-{codex|gemini|claude}`
5. `topic.md` 복사
6. 라운드별 `instruction.md` 생성 (점진적 목표 부여)
7. `run.sh` 생성 (위 템플릿, `{task_name}` 치환) + `chmod +x`
8. 결과 요약 출력 (아래 형식)
9. **종료**

### 결과 요약 형식

import 후 반드시 아래 형식으로 출력:

```
Discussion import 완료.

**토픽**: [topic.md 첫 줄 (# 제목)]
**라운드**: [N] 라운드 (복잡도 판단 1줄 근거)

| 라운드 | 도구 | 목표 |
|--------|------|------|
| 1 | codex | [instruction 1줄 요약] |
| 2 | gemini | [instruction 1줄 요약] |
| 3 | claude | [instruction 1줄 요약] |
| ... | ... | ... |

**생성 파일**:
.ralphss/disc/{task_name}/
├── run.sh (chmod +x)
├── topic.md
└── rounds/
    ├── round-1-codex/instruction.md
    ├── round-2-gemini/instruction.md
    ├── round-3-claude/instruction.md
    └── ...

**실행**: `bash .ralphss/disc/{task_name}/run.sh`
```

## 실행 방법 (외부 터미널)

```bash
cd your-project
bash .ralphss/disc/{task_name}/run.sh
```

정리:
```bash
rm -rf .ralphss/disc/{task_name}/
```

## 도구별 실행 환경

| 도구 | 실행 방법 | 비고 |
|------|-----------|------|
| codex | `codex exec -o output.md -c 'sandbox_permissions=[]'` | 파일 탐색 차단 |
| gemini | `gemini -p --approval-mode yolo` | YOLO/credentials 노이즈 필터링 |
| claude | `DISABLE_OMC=1 claude -p --dangerously-skip-permissions --output-format text` | OMC 훅 비활성화 필수 |

> **claude 실행 시 주의**: `DISABLE_OMC=1` 환경변수로 OMC 훅을 비활성화해야 한다.
> 프롬프트 내 프로젝트 용어가 OMC 키워드와 충돌하여 스킬이 오탐 호출되는 것을 방지한다.

## 금지 행동
- 소스 코드 수정/삭제 금지
- 테스트 실행 금지
- git 조작 금지
- `.ralphss/disc/{task_name}/` 외 파일 생성/수정 금지
