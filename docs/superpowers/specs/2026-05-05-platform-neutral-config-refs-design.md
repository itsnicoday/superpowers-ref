# 플랫폼 중립적 설정 파일 참조 — Phase B 설계

## 배경

Phase A(`2026-05-05-platform-neutral-prose-design.md` 참조)는 일반적인 3인칭 "Claude" 산문을 에이전트 중립적 형태로 교체했습니다. 이 단계에서는 다음 범주인 스킬 내부의 플랫폼별 지시사항 파일(CLAUDE.md, AGENTS.md, GEMINI.md) 참조를 다룹니다.

이 플러그인은 여러 하네스(harness)에서 실행되며, 각 하네스는 자체 지시사항 파일을 읽습니다. 스킬에서 CLAUDE.md를 유일한 파일인 것처럼 지칭하는 곳은 Claude Code 중심의 가정으로, Codex / Gemini CLI / OpenCode에서는 성립하지 않습니다.

## 작업 범위 (In scope)

활성 스킬 내의 두 구체적인 줄:

1. **`skills/writing-skills/SKILL.md:58`** — `Project-specific conventions (put in CLAUDE.md)`
2. **`skills/receiving-code-review/SKILL.md:30`** — `"You're absolutely right!" (explicit CLAUDE.md violation)`

## 작업 범위 외 (Out of scope)

- **`skills/using-superpowers/SKILL.md:22, 26`** — 지시사항 우선순위 목록. 이 목록은 이미 세 가지(CLAUDE.md, GEMINI.md, AGENTS.md)를 모두 포괄하여 명시하고 있으며, 이는 올바른 접근입니다: 이 섹션은 다중 플랫폼 플러그인에서 *무엇이 사용자 지시사항으로 간주되는지*에 대한 실제 주장을 하고 있습니다. 수정 불필요.
- **역사적 / 예시 아티팩트**:
  - `skills/systematic-debugging/CREATION-LOG.md` — 기여 경로(`~/.claude/CLAUDE.md`)는 역사적 사실입니다.
  - `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` — 파일 전체가 CLAUDE.md 콘텐츠 변형을 테스트하는 정교한 예시입니다. 파일 이름, 본문, 그리고 `testing-skills-with-subagents.md`에서의 참조는 모두 유지됩니다; 이를 표준화하면 예시의 목적이 손상됩니다.
- **플랫폼 도구 참조** — Phase D 후보:
  - `skills/using-superpowers/SKILL.md:40` (GEMINI.md에 대한 Gemini CLI 도구 매핑 참고사항)
  - `skills/using-superpowers/references/gemini-tools.md` (`save_memory`가 GEMINI.md에 영속화됨)

## 치환 규칙

작업 범위 내 줄당 하나씩, 두 가지 개별 결정.

### 규칙 1: "프로젝트 전용 컨벤션을 둘 위치"

`writing-skills/SKILL.md:58`:

- **변경 전:** `Project-specific conventions (put in CLAUDE.md)`
- **변경 후:** `Project-specific conventions (put in your instructions file)`

하나의 파일 이름을 선택하기보다 일반적인 문구를 사용합니다. 하네스마다 읽는 파일이 다르며(CLAUDE.md, AGENTS.md, GEMINI.md 등) 스킬이 하나만 가정해서는 안 됩니다. 플랫폼 도구 참조 문서(`references/{codex,copilot,gemini}-tools.md`)가 각 플랫폼의 선호 파일을 명시하기에 적절한 공간입니다.

### 규칙 2: "(explicit CLAUDE.md violation)" 괄호 표기

`receiving-code-review/SKILL.md:30`:

- **변경 전:** `"You're absolutely right!" (explicit CLAUDE.md violation)`
- **변경 후:** `"You're absolutely right!" (explicit instruction-file violation)`

괄호 안의 표현은 실제로 중요한 역할을 합니다 — 이 문구가 단순한 문체적 결함이 아니라, 많은 사용자가 지시사항 파일에 작성해 둔 규칙을 적극적으로 위반함을 나타냅니다. "Instruction file"은 AGENTS.md / CLAUDE.md / GEMINI.md를 통틀어 지칭하는 자연스러운 교차 플랫폼 용어이며, 하나의 파일 이름을 고르거나 "common"으로 약화시키지 않고 원본의 의도를 유지합니다.

## 커밋 계획

순서대로 원자적 커밋(atomic commits) 진행:

1. **`writing-skills/SKILL.md`** — "프로젝트 컨벤션을 둘 위치" 줄에서 CLAUDE.md → "your instructions file"
2. **`receiving-code-review/SKILL.md`** — 위반 괄호 표기에서 CLAUDE.md → instruction-file
3. **플랫폼 도구 참조 문서** — 독자가 "your instructions file"을 실제 파일 이름으로 해석할 수 있도록 각 `references/{codex,copilot,gemini}-tools.md`에 선호하는 플랫폼별 지시사항 파일 이름(CLAUDE.md, AGENTS.md, GEMINI.md 등) 추가.

각 커밋 메시지에는 "Phase B" 및 해당 구성을 명시합니다.

## 검증 (Verification)

각 커밋 이후:

- 주변 단락을 읽어 문법과 의미가 여전히 올바르게 해석되는지 확인합니다.
- `grep -n "CLAUDE\.md" <touched-file>` — 활성 산문에 남은 항목이 없는지 확인(예외 항목은 이미 문서화됨).

두 커밋 이후:

- `grep -rn "CLAUDE\.md" skills/` 실행 시 문서화된 예외 항목(CREATION-LOG, CLAUDE_MD_TESTING 및 관련 내부 참조, using-superpowers의 우선순위 목록)만 반환되어야 합니다.

## 비목표 (Non-goals)

- `using-superpowers/SKILL.md`의 우선순위 목록 순서를 수정하지 마십시오. CLAUDE.md / GEMINI.md / AGENTS.md의 순서를 바꾸는 것은 미학적 변경일 뿐 치환이 아니며, 본 작업 범위를 벗어납니다.
- `examples/CLAUDE_MD_TESTING.md`의 이름을 변경하거나 내용을 수정하지 마십시오.
- Gemini-CLI 전용 도구 참조(Phase D 후보)를 수정하지 마십시오.

## 구현 참고사항

여기에 작성된 Phase B는 세 개의 커밋과 세 개의 비-Claude-Code 플랫폼 도구 참조를 다루었습니다. 실제 구현은 한 단계 더 나아갔습니다: 커밋 `8505703`에서 대칭성을 위해 네 번째 참조인 `references/claude-code-tools.md`가 추가되었습니다. 이로써 Claude Code의 지시사항 파일 컨벤션 및 도구 이름 목록이 주변 스킬 산문에 암묵적으로 남아있지 않고 다른 것들과 함께 정돈되었습니다. 해당 추가는 이 스펙에서 예상하지 않았으나 의도와 일치합니다.
