---
name: ralphss-loop
description: 구현 계획을 .ralphss/loop/ 파일 구조로 분배. ralphss loop 시 사용.
---

# ralphss-loop

구현 계획 파일을 읽고, claude CLI 자율 루프용 `.ralphss/loop/{task_name}/` 파일 구조로 분배한다.
외부 도구 의존 없이 `claude -p` 만으로 멀티 Phase 반복 실행이 가능하다.

> **이 스킬은 파일 생성만 수행한다.** 코드 수정, 테스트, git 조작 금지.

<!-- NOTE: 알려진 제약
  - `claude -p`는 매 호출마다 새 세션이라 단순 에러에도 세션이 중단되고 컨텍스트를 잃음
  - `--resume <session_id>`로 세션 연속성을 확보하면 자력 복구 가능 (ralphcc 방식)
  - 현재는 OMC 환경에서 사용할 때 OMC의 persistence loop로 극복하는 구조
  - 향후 run.sh에 세션 관리(--resume) 추가 검토 가능
-->

## 워크플로우
```
계획 파일 작성 → ralphss loop → 외부 터미널에서 bash .ralphss/loop/{task_name}/run.sh
```

## 입력
- 구현 계획 파일 (마크다운). 종료 조건, Phase별 작업, 기술 제약이 포함되어야 함.

## task_name 결정

`{YYYY-MM-DD}_{slug}` 형식으로 자동 생성한다.
- 날짜: 오늘 날짜
- slug: 계획 파일명에서 확장자를 제거하고 공백을 `-`로 치환 (예: `di-migration-plan.md` → `2026-03-10_di-migration-plan`)
- 계획 파일명이 generic할 경우 (예: `plan.md`) 계획 내 첫 번째 제목에서 slug 추출

## 출력

### 단일 Phase (기본)
```
.ralphss/loop/{task_name}/
├── run.sh                 # 자율 루프 실행 스크립트
├── PROMPT.md              # 루프 제어 지시문
├── fix_plan.md            # 커밋 단위 태스크 체크리스트
├── AGENT.md               # 빌드/테스트 명령어
└── specs/
    └── requirements.md    # 상세 도메인 스펙
```

### 멀티 Phase (계획에 Phase가 2개 이상일 때)
```
.ralphss/loop/{task_name}/
├── run.sh                   # Phase 자동 전환 + 자율 루프
├── phases/
│   ├── phase-1/
│   │   ├── PROMPT.md        # Phase별 루프 제어
│   │   ├── fix_plan.md      # Phase별 태스크
│   │   └── verify.sh        # Phase별 검증 (exit 0 = 통과)
│   ├── phase-2/
│   │   └── ...
│   └── ...
├── MASTER_PLAN.md           # 전체 계획 (참조용)
├── AGENT.md                 # 공통 빌드/테스트
└── specs/
    └── requirements.md      # 공통 상세 스펙
```

---

## 파일별 생성 규칙

### 1. PROMPT.md — 루프 제어 전용

**원칙**: 도메인 지식은 `specs/requirements.md`로. PROMPT.md는 루프 제어 정보만.

**필수 섹션** (이 순서로):

```markdown
# [프로젝트/태스크명]

## Context
[3~5줄. 프로젝트명, 한줄 설명, 현재 상태]

## Current Objectives
[핵심 목표 4~6개]

## Key Principles
- ONE task per loop — 가장 중요한 한 가지에 집중
- 코드베이스 먼저 검색, 가정하지 않기
- .ralphss/loop/{task_name}/fix_plan.md 업데이트 후 커밋
- 상세 스펙은 .ralphss/loop/{task_name}/specs/requirements.md 참조

## Protected Files (수정 금지)
- .ralphss/ 디렉터리 (단, fix_plan.md는 수정 허용)
- .claude/ 전체 디렉터리

## Testing Guidelines
- 테스트 비중: 전체 작업의 ~20%
- 새 기능 구현분만 테스트

## 종료 조건 (EXIT_SIGNAL: true)
[5개 이내 구체 기준. 모든 조건 AND일 때만 true]

## 금지 행동
[스코프 밖 행동 나열]

## STATUS 보고

응답 끝에 반드시 아래 블록을 포함한다:

---LOOP_STATUS---
STATUS: IN_PROGRESS | COMPLETE | BLOCKED
EXIT_SIGNAL: false | true
SUMMARY: <one line>
---END_LOOP_STATUS---

## Current Task
Follow .ralphss/loop/{task_name}/fix_plan.md and choose the most important item to implement next.
```

### 2. fix_plan.md — 1 태스크 = 1 커밋

**크기 규칙**: 1 태스크 = 1 커밋 = **최대 3~5 파일 변경**

```markdown
# Fix Plan

## High Priority
- [ ] 태스크명 (대상 파일 1~3개 명시)

## Medium Priority
- [ ] 태스크명

## Completed

## Notes
- 참조: specs/requirements.md
```

### 3. specs/requirements.md — 상세 도메인 스펙

PROMPT.md에서 빠진 모든 도메인 정보.

### 4. AGENT.md — 빌드/테스트 명령어

```markdown
# Agent Build Instructions

## 기술 스택
## 빌드/실행
## 테스트
## 핵심 규칙
```

### 5. MASTER_PLAN.md — 전체 계획 (멀티 Phase 전용)

```markdown
# Master Plan

## Phase 목록
| Phase | 디렉터리명 | 목표 | 의존 |
|-------|-----------|------|------|

## 전체 완료 기준
```

### 6. Phase별 verify.sh (멀티 Phase 전용)

```bash
#!/bin/bash
set -e
# 검증 명령어 1~5개
echo "Phase N verify: PASSED"
exit 0
```

### 7. run.sh — 자율 루프 실행 스크립트

**단일 Phase:**

```bash
#!/bin/bash
set -uo pipefail

RLOOP_DIR="$(cd "$(dirname "$0")" && pwd)"
LOG_DIR="$RLOOP_DIR/logs"
MAX_LOOPS="${MAX_LOOPS:-30}"

mkdir -p "$LOG_DIR"

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
  echo ">>> Ctrl+C — 정리 중..."
  if [ -n "${tmpfile:-}" ] && [ -f "${tmpfile:-}" ]; then
    local log="$LOG_DIR/loop-$(printf '%02d' "${loop_count:-0}").log"
    {
      echo "=== INTERRUPTED — $(date '+%Y-%m-%d %H:%M:%S') ==="
      cat "$tmpfile"
    } > "$log"
    echo ">>> 마지막 출력 저장: $log"
    rm -f "$tmpfile"
  fi
  echo "=== 중단: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
  rm -f /tmp/ralphss_output.*
  exit 130
}
trap cleanup SIGINT SIGTERM

prompt=$(cat "$RLOOP_DIR/PROMPT.md")
loop_count=0
exit_count=0

echo "=== ralphss-loop 시작: $(date '+%Y-%m-%d %H:%M:%S') ===" | tee "$LOG_DIR/run.log"

while [ $loop_count -lt $MAX_LOOPS ]; do
  loop_count=$((loop_count + 1))
  echo ""
  echo "=========================================="
  echo "[loop $loop_count/$MAX_LOOPS] $(date '+%H:%M:%S')"
  echo "=========================================="

  tmpfile=$(mktemp /tmp/ralphss_output.XXXXXX)

  # 실시간 스트리밍: stream-json + stdbuf + jq
  set +e
  stdbuf -oL claude -p "$prompt" --dangerously-skip-permissions \
    --output-format stream-json --verbose --include-partial-messages \
    < /dev/null 2>&1 \
    | stdbuf -oL tee "$tmpfile" \
    | stdbuf -oL jq --unbuffered -j "$JQ_FILTER" 2>/dev/null
  pipe_exit=${PIPESTATUS[0]}
  set -e

  if [ $pipe_exit -gt 128 ]; then
    echo ">>> 중단됨 (exit=$pipe_exit)"
    rm -f "$tmpfile"
    exit 130
  fi

  # 루프 로그 저장
  {
    echo "=== loop $loop_count — $(date '+%Y-%m-%d %H:%M:%S') ==="
    cat "$tmpfile"
    echo ""
  } > "$LOG_DIR/loop-$(printf '%02d' $loop_count).log"

  if grep -q "EXIT_SIGNAL: true" "$tmpfile"; then
    exit_count=$((exit_count + 1))
    echo ">>> EXIT_SIGNAL ($exit_count/2)"
    if [ $exit_count -ge 2 ]; then
      echo ">>> 완료"
      echo "=== 완료: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
      rm -f "$tmpfile"
      exit 0
    fi
  else
    exit_count=0
  fi

  rm -f "$tmpfile"
  sleep 1
  prompt="$RLOOP_DIR/fix_plan.md를 확인하고 남은 작업 중 가장 중요한 항목을 수행하라. 완료 시 EXIT_SIGNAL: true를 보고하라."
done

echo "WARNING: MAX_LOOPS 도달"
echo "=== MAX_LOOPS 도달: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
exit 1
```

**멀티 Phase:**

```bash
#!/bin/bash
set -uo pipefail

RLOOP_DIR="$(cd "$(dirname "$0")" && pwd)"
PHASES_DIR="$RLOOP_DIR/phases"
LOG_DIR="$RLOOP_DIR/logs"
CURRENT_FILE="$RLOOP_DIR/.current_phase"
MAX_LOOPS="${MAX_LOOPS:-30}"

mkdir -p "$LOG_DIR"

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
  echo ">>> Ctrl+C — 정리 중..."
  if [ -n "${tmpfile:-}" ] && [ -f "${tmpfile:-}" ]; then
    local log="$LOG_DIR/${current_phase:-unknown}-loop-$(printf '%02d' "${loop_count:-0}").log"
    {
      echo "=== INTERRUPTED — $(date '+%Y-%m-%d %H:%M:%S') ==="
      cat "$tmpfile"
    } > "$log"
    echo ">>> 마지막 출력 저장: $log"
    rm -f "$tmpfile"
  fi
  echo ">>> Phase: ${current_phase:-unknown}, 루프: ${loop_count:-0} 에서 중단됨"
  echo "=== 중단: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
  rm -f /tmp/ralphss_output.*
  exit 130
}
trap cleanup SIGINT SIGTERM

phases=($(ls -d "$PHASES_DIR"/*/ 2>/dev/null | sort))
if [ ${#phases[@]} -eq 0 ]; then
  echo "ERROR: No phases found in $PHASES_DIR"
  exit 1
fi

# 재개 지점 결정
current_phase=""
if [ -f "$CURRENT_FILE" ]; then
  current_phase=$(cat "$CURRENT_FILE")
fi
started=false
[ -z "$current_phase" ] && started=true

echo "=== ralphss-loop 멀티 Phase 시작: $(date '+%Y-%m-%d %H:%M:%S') ===" | tee "$LOG_DIR/run.log"
echo "Phase 수: ${#phases[@]}" | tee -a "$LOG_DIR/run.log"

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

  # Phase별 fix_plan.md를 루트에 심볼릭 링크
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

    # 실시간 스트리밍: stream-json + stdbuf + jq
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
      echo "  >>> 중단됨 (exit=$pipe_exit)"
      rm -f "$tmpfile"
      exit 130
    fi

    echo ""

    # 루프 로그 저장
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
        echo "  >>> Phase 루프 완료"
        rm -f "$tmpfile"
        break
      fi
    else
      exit_count=0
    fi

    rm -f "$tmpfile"
    sleep 1
    prompt="$RLOOP_DIR/phases/$current_phase/fix_plan.md를 확인하고 남은 작업 중 가장 중요한 항목을 수행하라. 완료 시 EXIT_SIGNAL: true를 보고하라."
  done

  # Phase 검증
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
echo "=== 모든 Phase 완료: $(date '+%Y-%m-%d %H:%M:%S') ==="
echo "=== 완료: $(date '+%Y-%m-%d %H:%M:%S') ===" >> "$LOG_DIR/run.log"
rm -f "$CURRENT_FILE"
```

---

## 실행 절차

### 공통 (1~4)
1. 계획 파일 읽기
2. 프로젝트 상태 파악
3. task_name 결정 (`YYYY-MM-DD_slug`)
4. 계획에서 Phase 수 판별 → **2개 이상이면 멀티 모드**

### 단일 Phase 모드
5. `mkdir -p .ralphss/loop/{task_name}/specs`
6. PROMPT.md 생성 (경로에 `{task_name}` 치환)
7. fix_plan.md 생성
8. specs/requirements.md 생성
9. AGENT.md 생성
10. run.sh 생성 (단일 Phase 템플릿, `{task_name}` 치환) + `chmod +x`
11. 결과 요약 출력
12. **종료**

### 멀티 Phase 모드
5. `mkdir -p .ralphss/loop/{task_name}/specs .ralphss/loop/{task_name}/phases`
6. MASTER_PLAN.md 생성
7. Phase별 디렉터리 + PROMPT.md + fix_plan.md + verify.sh 생성
8. AGENT.md + specs/requirements.md 생성
9. run.sh 생성 (멀티 Phase 템플릿, `{task_name}` 치환) + `chmod +x`
10. 결과 요약 출력
11. **종료**

## 실행 방법 (외부 터미널)

```bash
cd your-project
bash .ralphss/loop/{task_name}/run.sh
```

환경변수로 루프 횟수 조절 가능:
```bash
MAX_LOOPS=50 bash .ralphss/loop/{task_name}/run.sh
```

정리:
```bash
rm -rf .ralphss/loop/{task_name}/
```

## 금지 행동
- 소스 코드 수정/삭제 금지
- 테스트 실행 금지
- git 조작 금지
- `.ralphss/loop/{task_name}/` 외 파일 생성/수정 금지
