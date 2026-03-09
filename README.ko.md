# ralph-super-simple

이 리포지토리는 특정 에이전트 구현이 아니라, ralph 아이디어에서  착안한 **반복 실행 패턴**에 대한 설명입니다.

스킬 실행을 위해 [Claude CLI](https://claude.com/claude-code), [OMC(oh-my-claudecode)](https://github.com/nicobailey-omc/oh-my-claudecode), [ralph-claude-code](https://github.com/frankbria/ralph-claude-code)를 사용하고, 반복 토론을 위해 [Codex CLI](https://github.com/openai/codex), [Gemini CLI](https://github.com/google-gemini/gemini-cli)를 함께 사용합니다.

---

## 핵심

```python
for phase in phases:
    prompt = read(f"{phase}/PROMPT.md")
    while "EXIT_SIGNAL: true" not in run(f"claude -p '{prompt}'"):
        prompt = "fix_plan.md 확인하고 다음 작업 수행하라."
    run(f"bash {phase}/verify.sh")
```

이것이 전부입니다. 계획을 단계별 파일로 쪼개고, 각 단계를 독립 세션으로 반복 실행합니다. 한 세션에서는 하나의 단계 구현을 목표로 합니다.

**단계별 계획이 핵심입니다.** Phase별 구조, 종료 조건, 검증 기준이 포함된 계획을 생성하고, 그 계획을 제공 되는 스킬을 사용하여 구현합니다. oh-my-claudecode의 `/ralplan`과 같은 스킬이 단계별 계획 생성에 잘 맞습니다.

---

## 스킬

| 스킬 | 용도 | 필요 도구 |
|------|------|-----------|
| **ralphss-import** | 계획을 `claude -p` 루프로 독립 반복 실행 | Claude CLI |
| **ralphcc-import** | ralph-claude-code 에이전트와 연동하여 반복 실행 | Claude CLI + ralph-claude-code |
| **ralph-discussion** | 특정 토픽에 대해 멀티 모델 교대 토론 | Codex CLI + Gemini CLI |

### ralphss-import — `claude -p` 독립 반복

구현 계획을 `.ralphss/` 파일 구조로 분배합니다. 해당 스킬은 `claude -p`로 동작합니다. 주 사용 CLI를 다른 것(codex, gemini 등)으로 교체 편집하여 사용할 수도 있습니다.

```bash
/ralphss-import my-plan.md     # .ralphss/ 생성
bash .ralphss/run.sh           # 외부 터미널에서 실행
rm -rf .ralphss/               # 정리
```

**/ralphss-import 실행 결과:**
```
.ralphss/
├── run.sh                     # 자율 루프 (claude -p 직접 호출)
├── MASTER_PLAN.md             # 전체 계획
├── AGENT.md                   # 빌드/테스트 명령어
├── specs/requirements.md      # 상세 스펙
└── phases/
    ├── phase-1/
    │   ├── PROMPT.md           # 루프 제어 지시문
    │   ├── fix_plan.md         # 커밋 단위 태스크 체크리스트
    │   └── verify.sh           # Phase별 자동 검증 스크립트
    └── ...
```

### ralphcc-import — ralph-claude-code 에이전트 연동

구현 계획을 `.ralph/` 파일 구조로 분배합니다. ralph-claude-code의 `--resume` 세션 관리와 EXIT_SIGNAL 감지를 활용합니다.

```bash
/ralphcc-import                # .ralph/ 생성
bash .ralph/run.sh             # 외부 터미널에서 실행 (Phase 자동 전환)
/ralphcc-clear                 # ralph-claude-code 생성 파일 정리
```

**/ralphcc-import 실행 결과:**
```
.ralph/
├── run.sh                     # Phase 전환 + ralph 호출
├── MASTER_PLAN.md             # 전체 계획
├── AGENT.md                   # 빌드/테스트 명령어
├── specs/requirements.md      # 상세 스펙
└── phases/
    ├── phase-1/
    │   ├── PROMPT.md
    │   ├── fix_plan.md
    │   └── verify.sh
    └── ...
```

### ralph-discussion — 멀티 모델 설계 토론

특정 토픽에 대해 Codex, Gemini가 교대로 토론합니다. 3~5라운드.

```bash
/ralph-discussion my-topic.md  # .discussion/ 생성
bash .discussion/run.sh        # codex ↔ gemini 교대 실행
```

**/ralph-discussion 실행 결과:**
```
.discussion/
├── run.sh                     # 라운드 순회 실행기
├── topic.md                   # 원본 토픽 (복사)
├── synthesis.md               # 마지막 라운드 output 복사 (런타임 생성)
└── rounds/
    ├── round-1-codex/
    │   ├── instruction.md     # 라운드 지시 (import 시 생성)
    │   ├── prompt.md          # 실제 프롬프트 (런타임 생성)
    │   └── output.md          # 응답 (런타임 생성)
    ├── round-2-gemini/
    │   └── ...
    ├── round-3-codex/
    │   └── ...
    └── ...
```

---

## 용어

| 용어 | 설명 |
|------|------|
| `PROMPT.md` | Phase의 루프 제어 지시문입니다. 가볍게 유지하고 상세 스펙은 `specs/`에 분리합니다 |
| `fix_plan.md` | 커밋 단위 태스크 체크리스트입니다. 1 태스크 = 1 커밋, 최대 3~5 파일 변경을 권장합니다 |
| `verify.sh` | Phase 완료를 검증하는 스크립트입니다. `grep`, `pytest`, `curl` 등 구체적 명령으로 확인합니다 |
| `EXIT_SIGNAL` | 루프 종료 신호입니다. [ralph-claude-code](https://github.com/frankbria/ralph-claude-code)에서 유래했습니다 |
| `MASTER_PLAN.md` | 멀티 Phase 전체 계획입니다 |
| `AGENT.md` | 빌드/테스트 명령어를 정의합니다 |
| `run.sh` | 자율 루프 + Phase 전환을 수행하는 통합 스크립트입니다 |
| `instruction.md` | discussion 라운드별 구체적 목표를 지시합니다 |
| `synthesis.md` | discussion 최종 종합 결과입니다 |

---

## 설치

```bash
cp -r skills/* your-project/.claude/skills/
```

---

## License

MIT
