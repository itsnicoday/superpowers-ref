# Superpowers 테스트하기

Superpowers에는 두 가지 종류의 테스트가 있으며, 각각 별도의 디렉토리에 위치합니다:

- **`tests/`** — 플러그인의 비-LLM 코드가 정상 작동하나요? brainstorm-server JS, OpenCode 플러그인 로딩, codex-plugin 동기화, 분석 유틸리티를 위한 Bash + Node + Python 통합 테스트입니다.
- **`evals/`** — 에이전트가 실제 LLM 세션에서 올바르게 동작하나요? Claude Code / Codex / Gemini CLI의 실제 tmux 세션을 구동하는 Python 하네스로, LLM 액터와 검증기가 스킬 준수 여부를 판단합니다.

## 플러그인 테스트

`tests/`에 위치합니다. 현재 포함된 내용:

- `tests/brainstorm-server/` — brainstorm 서버 JS 코드를 위한 Node 테스트 스위트.
- `tests/opencode/` — OpenCode 플러그인 로딩, 부트스트랩 캐싱, 툴 등록을 위한 Bash 테스트.
- `tests/codex-plugin-sync/` — Bash 동기화 검증.
- `tests/kimi/` — Kimi 플러그인 매니페스트 와이어링을 위한 Bash/Python 검사.
- `tests/claude-code/test-helpers.sh`, `analyze-token-usage.py` — 남은 Bash 테스트에서 사용하는 유틸리티.
- `tests/claude-code/test-subagent-driven-development.sh` — 에이전트의 SDD 설명 가능 여부 테스트 (drill 대응 항목 없음; 동작이 아닌 설명 회상 능력을 테스트함).
- `tests/claude-code/test-subagent-driven-development-integration.sh` — 토큰 분석이 포함된 확장 SDD 통합 (drill은 YAGNI 서브셋을 다루고; Bash는 커밋 수, Claude Code 작업 추적 및 토큰 텔레메트리 단언을 추가함).
- `tests/claude-code/test-worktree-native-preference.sh` — worktree 스킬을 위한 RED-GREEN-REFACTOR 검증 (drill은 PRESSURE 단계를 다루고; Bash는 RED/GREEN 베이스라인도 다룸).
- `tests/explicit-skill-requests/` — drill에서 다루지 않는 Haiku 전용, 멀티턴, 스킬 이름 프롬프트 테스트.

관련 디렉토리의 `run-*.sh` 또는 `npm test`를 통해 플러그인 테스트를 실행하세요.

## 스킬 동작 평가 (Evals)

`evals/`에 위치합니다. Drill이 하네스이며; 시나리오는 `evals/scenarios/*.yaml`에 있습니다. 설정 방법은 `evals/README.md`를 참조하세요. 빠른 시작:

```bash
cd evals
uv sync --extra dev
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill 시나리오는 느리고 (각각 3~30분 이상 소요) 실제 LLM 세션을 실행합니다. 현재는 CI의 일부가 아니며; 자연스러운 추후 작업은 계층형 모델 (PR 시 빠른 서브셋 실행, 매일 밤 + 필요 시 전체 실행) 도입입니다.
