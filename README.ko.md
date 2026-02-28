# ralph-super-simple

ralph-super-simple은 특정 에이전트 구현이 아니라, `claude -p` 루프 기반 멀티 Phase 작업을 위한 최소 아키텍처 패턴입니다.

실제 실행 스킬 이름은 환경에 맞게 자유롭게 생성할 수 있습니다.

> 장기 실행 Claude Code 루프를 위한 최소 아키텍처 패턴.
>
> 이 저장소는 패턴을 정의합니다.
> 스킬은 패턴을 구현합니다.

---

## 설치

스킬 폴더를 프로젝트의 `.claude/skills/`에 복사한다.

```bash
# 전체 설치 (독립 실행 + ralph-claude-code 연동)
cp -r skills/* your-project/.claude/skills/

# 독립 실행만 (ralph-claude-code 불필요)
cp -r skills/ralphss-import skills/ralphss-clear your-project/.claude/skills/

# ralph-claude-code 연동만
cp -r skills/ralphcc-import skills/ralphcc-clear your-project/.claude/skills/
```

설치 후 Claude Code에서 `/ralphss-import` 또는 `/ralphcc-import`로 호출 가능.

---

## 개념

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

## 배경

Claude Code는 강력한 코딩 에이전트지만, 단일 세션에서 대형 작업을 수행할 때 몇 가지 한계가 있다:

- **에러 시 중단**: 단순한 에러에도 세션이 중단되어 다시 개입해야 하는 경우가 잦다
- **컨텍스트 한계**: 서브 에이전트, 에이전트 팀 등 컨텍스트 제약을 벗어나기 위한 방법들이 계속 나오고 있으나, 계획이 커질수록 제약을 벗어나기가 쉽지 않다. 매번 요약, 메모리, 클리어 등 컨텍스트가 다 찰 즈음에 작업을 정리하고 다음 세션으로 넘어가기 위한 작업들이 우리를 굉장히 피로하게 만든다

이를 보완하기 위해 여러 도구를 조합하여 사용해왔다.

### oh-my-claudecode (OMC)

[oh-my-claudecode](https://github.com/nicobailey-omc/oh-my-claudecode)는 매우 훌륭한 도구다. 멀티 에이전트 오케스트레이션, 반복 루프, 병렬 실행 등을 제공하며 개발 퍼포먼스를 비약적으로 향상시켜준다. 기본적으로 설치하여 사용하는 것을 추천한다.

### ralph-claude-code

[ralph-claude-code](https://github.com/frankbria/ralph-claude-code)는 "끝날 때까지 멈추지 않는" 자율 루프에 대한 진정한 해결과 고민에 영감을 준 도구다. `claude -p`를 반복 호출하며 EXIT_SIGNAL 기반으로 완료를 감지하는 접근 방식은 단순하면서도 효과적이다.

다만 실사용에서 몇 가지 어려움이 있었다:

- **타임아웃 오판**: 정상 완료인데 timeout으로 처리됨 ([#198](https://github.com/frankbria/ralph-claude-code/issues/198))
- **프롬프트 크기 관리**: 세션이 이어질수록 컨텍스트가 누적되어 프롬프트 초과
- **exit code 처리**: `set -e`로 인한 무조건 중단 ([#200](https://github.com/frankbria/ralph-claude-code/issues/200), [#175](https://github.com/frankbria/ralph-claude-code/issues/175))
- **조기 종료**: 작업 미완료 상태에서 루프 종료 ([#194](https://github.com/frankbria/ralph-claude-code/issues/194))

이 도구의 사용 경험이 없었다면 이 스킬들에 대한 영감도 받지 못했을 것이다. 한번 사용해보기를 추천한다.

### 결론

결국 **단계별 계획이 잘 작성되고 이를 반복 수행할 수 있는 구조**만 있으면, 세션 제한 · 컨텍스트 관리 · 요약 · clear · 다시 시작에 대한 수고를 크게 줄일 수 있었다. 특정 도구에 의존하지 않더라도 `claude -p "프롬프트"` → 출력 확인 → 반복, 이것만으로 충분하다.

이 경험을 바탕으로 두 가지 스킬을 공유한다. 누군가에게 도움이 되고 영감을 주는 계기가 된다면 매우 기쁠 것이다.

---

## 핵심 원리

```python
# 이것이 전부다
for phase in phases:
    prompt = read(f"{phase}/PROMPT.md")
    while "EXIT_SIGNAL: true" not in run(f"claude -p '{prompt}'"):
        prompt = "fix_plan.md 확인하고 다음 작업 수행하라."
    run(f"bash {phase}/verify.sh")
```

---

## 제공 스킬

### 1. `ralphcc-import` — ralph-claude-code 연동 스킬

구현 계획을 `.ralph/` 파일 구조로 분배. ralph-claude-code가 바라보는 구성 파일(PROMPT.md, fix_plan.md)을 단계별로 복사하여 실행하고, 종료 후 다음 단계를 복사하여 반복 실행하는 멀티 Phase 자동 전환 기능이다.

**요구사항**: [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) 설치 필요

**사용법:**
```
/ralphcc-import   # 계획 파일을 .ralph/ 구조로 분배
/ralphcc-clear    # .ralph/ 정리 (다음 import 준비)
```

**실행:**
```bash
cd your-project
bash .ralph/run.sh            # 외부 터미널에서 실행
# 또는
ralph --live --verbose        # ralph CLI 직접 실행
```

### 2. `ralphss-import` — 독립 실행 스킬

구현 계획을 `.ralphss/` 파일 구조로 분배. 외부 도구 없이 `claude -p`만으로 실행.

**요구사항**: `claude` CLI만 있으면 됨 (추가 설치 없음)

**사용법:**
```
/ralphss-import 계획파일.md   # 계획을 .ralphss/ 구조로 분배
/ralphss-clear                # .ralphss/ 정리 (다음 import 준비)
```

**실행:**
```bash
cd your-project
bash .ralphss/run.sh   # 외부 터미널에서 실행
```

\* 계획은 [oh-my-claudecode](https://github.com/nicobailey-omc/oh-my-claudecode)의 `/ralplan` 스킬로 생성하면 Phase별 구조, 종료 조건, 검증 기준이 포함되어 import 입력으로 잘 맞습니다.

---

## 구조

```
ralph-super-simple/
├── README.md
├── README.ko.md
├── LICENSE
└── skills/
    ├── ralphss-import/SKILL.md    # 독립 실행 (claude -p만 사용)
    ├── ralphss-clear/SKILL.md     # .ralphss/ 정리
    ├── ralphcc-import/SKILL.md    # ralph-claude-code 연동
    └── ralphcc-clear/SKILL.md     # .ralph/ 정리
```

### 생성 결과물 비교

**ralphss-import** (`.ralphss/`):
```
.ralphss/
├── run.sh                     # 자율 루프 (claude -p 직접 호출)
├── MASTER_PLAN.md             # 전체 계획
├── AGENT.md                   # 빌드/테스트
├── specs/requirements.md      # 상세 스펙
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
├── run.sh                     # Phase 전환 + ralph 호출
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

## 용어 출처

| 용어/개념 | 출처 | 설명 |
|-----------|------|------|
| `EXIT_SIGNAL` | [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) | 루프 종료 신호. 2회 연속 감지 시 완료 |
| `RALPH_STATUS` 블록 | ralph-claude-code | ralph가 파싱하는 상태 보고 형식 |
| `.ralphrc` | ralph-claude-code | 프로젝트 설정 (ALLOWED_TOOLS, 타임아웃) |
| `.ralph/` 디렉터리 | ralph-claude-code | ralph 실행 환경 디렉터리 |
| `LOOP_STATUS` 블록 | **ralph-super-simple** | EXIT_SIGNAL 기반 경량 상태 보고 형식 |
| `.ralphss/` 디렉터리 | **ralph-super-simple** | 독립 실행 환경 디렉터리 |
| `run.sh` | **ralph-super-simple** | 자율 루프 + Phase 전환 통합 스크립트 |
| `fix_plan.md` | 공통 | 커밋 단위 태스크 체크리스트 |
| `PROMPT.md` | 공통 | 루프 제어 지시문 |
| `verify.sh` | 공통 | Phase별 자동 검증 스크립트 |
| `MASTER_PLAN.md` | 공통 | 멀티 Phase 전체 계획 |
| `AGENT.md` | 공통 | 빌드/테스트 명령어 |
| `specs/requirements.md` | 공통 | 상세 도메인 스펙 |

---

## 핵심 설계 원칙

### 1. Phase = 독립된 세션

각 Phase는 새 세션에서 시작한다. 이전 Phase의 컨텍스트를 이어받지 않아 컨텍스트 오염이 없다.

### 2. PROMPT.md = 루프 제어, specs/ = 도메인 지식

PROMPT.md는 가볍게 유지하고 상세 스펙은 별도 파일로 분리한다. 프롬프트 크기 문제를 방지한다.

### 3. verify.sh = 자동 품질 게이트

Phase가 끝났다고 주장할 때 실제로 검증한다. `grep`, `pytest`, `curl` 등 구체적 명령으로 확인.

### 4. fix_plan.md = 1 태스크 = 1 커밋

작업 단위를 작게 유지하여 각 루프가 명확한 목표를 갖게 한다. 최대 3~5 파일 변경.

### 5. Ctrl+C 안전 종료

`run.sh`/`phase_runner.sh`는 trap으로 인터럽트를 잡아 상태 파일을 정리한다.

---

## 호환 도구

| 도구 | 역할 | 필수 여부 |
|------|------|-----------|
| [Claude Code](https://claude.com/claude-code) | CLI 에이전트 | 필수 |
| [oh-my-claudecode](https://github.com/nicobailey-omc/oh-my-claudecode) | 계획 수립, 에이전트 오케스트레이션 | 권장 |
| [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) | 자율 루프 엔진 | 권장 (`ralphcc-import` 사용 시) |

---

## 알려진 개선 영역

아래 항목들은 의도적으로 구현하지 않았다. 현재 구조로 충분히 동작하며, 실제로 문제가 발생했을 때 필요에 따라 추가하면 된다.

| 영역 | 설명 | 현재 상태 |
|------|------|-----------|
| **EXIT_SIGNAL 단독 종료** | Claude가 EXIT_SIGNAL을 누락하거나 포맷이 깨지면 무한루프 가능성 | verify.sh가 보조 게이트 역할. 필요 시 verify 기반 종료로 전환 가능 |
| **상태 저장 (state.json)** | 중단 후 재시작 시 이전 Phase/루프 위치 복원 | Ctrl+C trap으로 안전 종료. 재시작 시 Phase 처음부터 |
| **verify 실패 retry** | verify.sh 실패 시 자동 재시도 루프 | 실패 시 수동 확인 후 재실행 |
| **프롬프트 파일 명시적 로드** | 컨텍스트 없는 세션에서 fix_plan.md를 못 읽을 가능성 | PROMPT.md에 파일 경로가 명시되어 있어 실사용에서 문제 없었음 |

이 스킬의 철학은 **최소 구현**이다. 실패 처리 코드를 미리 작성하는 것보다, 실제 실패가 발생했을 때 그 맥락에 맞게 대응하는 것이 더 효과적이다.

---

## 노트

내가 사용하는 스킬은 `ralphcc-import`이다. ralphcc-import을 먼저 만들어 보고, 단순한 관점으로 — 이런 에이전트를 짧은 단계 수행만 반복하는 형태로 내 환경에 맞는 스킬을 만들 수 있지 않을까? — 하는 생각으로 `ralphss-import`을 만들어 보았다. 단순히 영감을 증명해본 스킬일 뿐이다.

---

## License

MIT
