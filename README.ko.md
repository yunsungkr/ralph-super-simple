# ralph-super-simple

`ralph-super-simple`은 AI 코딩 에이전트를 "끝날 때까지 반복 실행"하기 위한 실전 패턴을 정리한 저장소입니다.
특정 모델 구현체가 아니라, **phase 기반 계획 + autonomous loop + 검증 스크립트**를 중심으로 Claude Code, Codex CLI, Gemini CLI를 조합해 운영하는 방법에 집중합니다.

구성에 사용하는 도구:
- [Claude Code](https://claude.com/claude-code)
- [OMC (oh-my-claudecode)](https://github.com/nicobailey-omc/oh-my-claudecode)
- [Codex CLI](https://github.com/openai/codex)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli)

## TL;DR

> 단계별 계획이 잘 작성되고 이를 반복 수행할 수 있는 구조만 있으면,
> 세션 제한 · 컨텍스트 관리 · 요약 · clear · 다시 시작에 대한 수고를 크게 줄일 수 있다.

- **세션 제약**: 컨텍스트가 차면 요약 · 메모리 · clear · 다시 시작의 반복이 발생한다
- **에러 시 중단**: 단순한 에러에도 세션이 끊겨 매번 개입해야 한다
- **CLI 경계**: 단일 도구에 종속되면 그 도구의 한계가 곧 작업의 한계가 된다

이 저장소는 **phase 단위 파일 구조 + 독립 세션 반복**으로 해결합니다.
각 phase가 독립 세션이므로 컨텍스트 오염이 없고, 실행 CLI를 자유롭게 교체할 수 있습니다.

## 핵심 패턴 (Core Loop)

```python
for phase in phases:
    prompt = read(f"{phase}/PROMPT.md")
    while "EXIT_SIGNAL: true" not in run(f"claude -p '{prompt}'"):
        prompt = "fix_plan.md 확인하고 다음 작업 수행하라."
    run(f"bash {phase}/verify.sh")
```

핵심은 단순합니다.
`계획 분해 → 반복 실행 → 자동 검증`을 phase별로 고정하면, 에이전트와 관계없이 세션에 제약을 받지 않고 phase 완료에 집중하여 장기간 실행도 안정적으로 수행합니다.

## 포함된 스킬

| 스킬 | 용도 | 필요 도구 |
| --- | --- | --- |
| **ralphss-loop** | 계획을 독립 `claude -p` 루프로 반복 실행 | Claude Code |
| **ralphss-disc** | Codex/Gemini/Claude 순환 라운드 토론 자동화 | Claude Code + Codex CLI + Gemini CLI |

### ralphss-loop: 독립 `claude -p` 반복 루프

기본은 `claude -p`로 동작하지만, `run.sh` 내 CLI 호출을 Codex, Gemini 등으로 교체하면 그대로 사용할 수 있습니다.

```bash
/ralphss-loop my-plan.md                          # .ralphss/loop/{task_name}/ 생성
cd .ralphss/loop/{task_name} && bash run.sh       # 별도 터미널에서 실행
rm -rf .ralphss/loop/{task_name}/                  # 정리
```

생성 구조:

```text
.ralphss/loop/{task_name}/
├── run.sh                     # 자율 루프 (claude -p 직접 호출)
├── MASTER_PLAN.md             # 전체 계획
├── AGENT.md                   # 빌드/테스트 명령
├── specs/requirements.md      # 상세 요구사항
└── phases/
    ├── phase-1/
    │   ├── PROMPT.md          # 루프 제어 지시문
    │   ├── fix_plan.md        # 커밋 단위 체크리스트
    │   └── verify.sh          # phase 완료 검증
    └── ...
```

### ralphss-disc: Codex/Gemini/Claude 멀티 모델 토론

```bash
/ralphss-disc my-topic.md                          # .ralphss/disc/{task_name}/ 생성
cd .ralphss/disc/{task_name} && bash run.sh        # Codex → Gemini → Claude 순환 실행
rm -rf .ralphss/disc/{task_name}/                  # 정리
```

생성 구조:

```text
.ralphss/disc/{task_name}/
├── run.sh                     # 라운드 실행기 (이 폴더에서 바로 실행)
├── topic.md                   # 원본 토픽 복사본
├── synthesis.md               # 마지막 claude 라운드 종합본
└── rounds/
    ├── round-1-codex/
    │   ├── instruction.md     # 라운드 지시
    │   ├── prompt.md          # 실제 프롬프트 (토픽 + 지시 + 이전 output)
    │   └── output.md          # 라운드 응답
    ├── round-2-gemini/
    │   └── ...
    ├── round-3-claude/
    │   └── ...
    └── ...
```

### 통합 `.ralphss/` 디렉터리

모든 스킬이 하나의 `.ralphss/` 루트를 공유하며, 유형과 날짜별 task name으로 구분됩니다:

```text
.ralphss/
├── loop/
│   ├── 2025-01-15_api-refactor/
│   └── 2025-01-20_auth-module/
└── disc/
    ├── 2025-01-15_caching-strategy/
    └── 2025-01-22_database-migration/
```

`.gitignore`에 `.ralphss/` 한 줄이면 전부 커버됩니다. 날짜 폴더로 히스토리가 보존되고, 불필요한 건 수동 삭제하면 됩니다.

## 빠른 시작 (Quick Start)

### 1) 스킬 설치

```bash
cp -r skills/* your-project/.claude/skills/
```

### 2) 첫 실행 예시

```bash
/ralphss-loop my-plan.md
cd .ralphss/loop/{task_name} && bash run.sh
```

### 3) 추천 운영 규칙

- `1 task = 1 commit` 규칙으로 `fix_plan.md`를 유지
- `verify.sh`는 사람이 읽어도 바로 검증 가능한 명령으로 작성
- phase 종료 조건은 반드시 `EXIT_SIGNAL`로 명시

## 용어

| 용어 | 설명 |
| --- | --- |
| `PROMPT.md` | phase 루프 제어 지시문 |
| `fix_plan.md` | 커밋 단위 태스크 체크리스트 |
| `verify.sh` | phase 완료를 확인하는 자동 검증 스크립트 |
| `EXIT_SIGNAL` | 루프 종료 신호 |
| `MASTER_PLAN.md` | 멀티 phase 전체 계획 |
| `AGENT.md` | 빌드/테스트 명령 모음 |
| `instruction.md` | discussion 라운드 목표 지시 |
| `synthesis.md` | discussion 최종 종합 결과 |

## Acknowledgments

이 저장소의 자율 루프 구조와 패턴은 [ralph-claude-code](https://github.com/frankbria/ralph-claude-code)에서 영감을 받았습니다.

## License

MIT