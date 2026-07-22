# 플랫폼 중립적 산문 — Phase A 설계

## 배경

Superpowers는 여러 에이전트 런타임(Claude Code, Codex, Cursor, OpenCode, Copilot CLI, Gemini CLI)에 배포됩니다. 스킬 내용과 보조 문서는 당초 Claude Code용으로 먼저 작성되었으며, 임의의 런타임 에이전트에 적용되는 부분에서도 "Claude"를 사용하고 있었습니다. OpenAI의 벤더링된 포크(openai/plugins#217)는 전체 재작성을 시도했으나, 역사적 기여 경로, 모델 이름, 플랫폼별 설치 지침을 수정하는 등 일부 지점에서 실제로 잘못된 결과를 초과했습니다 — 우리는 플랫폼 중심 산문이 진정으로 부차적인 위치에 있을 때 이를 제거하면서도 이러한 실수를 방지하고자 합니다.

전체 작업은 참조 범주별로 단계(Phase)로 나뉩니다. **이 스펙은 Phase A만 다룹니다:** 비플랫폼 전용 컨텍스트에서 "Claude"를 언급하는 일반적인 3인칭 산문. 이후 단계(설정 파일 참조, 마케팅 카피, 도구 이름 참조)는 본 범위 밖이며 자체 스펙을 갖게 됩니다.

## 작업 범위 (In scope)

다음 파일들에 포함된 일반적인 "Claude" 산문 언급:

- `skills/*/SKILL.md` 및 활성 스킬 디렉토리 내 보조 `.md` 파일들
- `skills/writing-skills/anthropic-best-practices.md`
- `README.md` (플랫폼 마케팅이 아닌 일반 산문 언급에 한함)

추가로 신조어 용어 변경 1건: `skills/writing-skills/SKILL.md` 내의 **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**.

## 작업 범위 외 (Out of scope)

- **플랫폼/런타임 진술** — "In Claude Code:", 설치 지침, 도구 매핑 참조. (Phase D 후보.)
- **설정 파일 참조** — CLAUDE.md, AGENTS.md, GEMINI.md 우선순위 목록 및 "프로젝트 컨벤션을 둘 위치" 콜아웃. (Phase B.)
- **도구 이름 참조** — `Skill`, `Bash`, `Read`, `Task`, `TodoWrite`. 스킬은 Claude Code의 도구 어휘로 작성되어 있으며, 기존 `references/{codex,copilot,gemini}-tools.md` 파일이 이를 매핑합니다. (이 스펙이 작성될 당시 계획은 이를 연기하거나 건너뛰는 것이었습니다. 그러나 Phase E에서 결국 이 작업을 수행했습니다 — 활성 스킬 전반에서 도구 이름을 행위 언어로 교체하고 동일한 어휘를 중심으로 플랫폼 도구 참조를 통합함.)
- **README의 마케팅 카피** — "Superpowers for Claude Code", 플랫폼 이름이 들어간 설치 섹션. (Phase C.)
- **역사적 아티팩트** — `docs/plans/*.md`, `docs/superpowers/specs/*.md`, `CREATION-LOG.md`. 이들은 날짜가 지정된 특정 시점의 문서이며, 이를 재작성하는 것은 역사를 재작성하는 것입니다.
- **모델 식별자** — Claude Haiku / Sonnet / Opus. 이들은 실제 제품 이름입니다.
- **파일 이름 / URL 참조** — `CLAUDE.md`, `claude.com`, `claude-plugin/`, `~/.claude/` 아래의 경로들.
- **`anthropic-best-practices.md` 파일 이름** — 내부 산문을 재작성하더라도 해당 파일은 소스 이름을 그대로 유지합니다.

## 치환 스타일 (Replacement style)

영어에서 자연스럽게 읽히는 조합을 사용합니다:

- **2인칭 — "your agent"**: 스킬 작성자에게 *그들의* 런타임에 대해 다룰 때
  - "your agent reads the description"
- **3인칭 — "the agent" / "agents" / "an agent"**: 시스템 동작을 일반화하여 설명할 때
  - "Future agents find your skills"
  - "Use words an agent would search for"
  - "Agents read SKILL.md only when the skill becomes relevant"

주변 문장에 잘 맞는 표현을 선택하십시오; 어색한 문구를 무릅쓰고 일관성을 강제하지 마십시오. 항상 "the agent"라고 하기보다 자연스러운 경우 복수형("future agents", "agents read")을 사용하십시오.

### "Claude"로 유지되는 예외 항목

- 모델 이름: Claude Haiku, Claude Sonnet, Claude Opus
- 파일 이름 및 URL: `CLAUDE.md`, `claude.com`, `~/.claude/`
- 런타임 자체를 지칭하는 경우 브랜딩된 플랫폼 이름 "Claude Code" (이후 단계에서 처리)

### 신조어 용어 변경

- **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**
  - `skills/writing-skills/SKILL.md`에서 섹션 제목 및 인근 산문에 나타납니다. 제목, 약어, 파일 내 상호 참조를 모두 변경합니다.

## 영향을 받는 파일들

예외 항목을 제외하도록 필터링된 `grep`을 기반으로 한 대략적인 수치:

| File | Generic-prose mentions |
|------|------------------------|
| `skills/writing-skills/SKILL.md` | ~12 (CSO 제목 + 본문 포함) |
| `skills/writing-skills/anthropic-best-practices.md` | ~30 |
| `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` | ~1 — 파일 이름 유지 (CLAUDE.md 테스트 아티팩트임); "Variant C: Claude.AI Emphatic Style" 제목도 유지 (특정 스타일을 명명하는 라벨임) |
| `README.md` | ~1 |

구현 중 필터링된 grep을 재실행하여 최종 목록을 확인했습니다.

## 커밋 계획

순서대로 4개의 원자적 커밋 진행:

1. `skills/writing-skills/SKILL.md`에서 **CSO → SDO 이름 변경**. 기계적이고 고립되어 있으며, 용어에 대한 생각이 바뀌면 쉽게 되돌릴 수 있음.
2. **활성 스킬 산문** — `anthropic-best-practices.md`를 제외한 `skills/*/SKILL.md` 및 보조 `.md` 전반에서 일반적인 "Claude" → "agent" 형태로 변경.
3. **`anthropic-best-practices.md` 산문** — 동일한 치환 규칙 적용. 이 파일은 외부 문서의 벤더링된 적응물이므로 별도 커밋으로 진행; 변경 사항을 고립시키면 업스트림과의 향후 조율 내용을 더 쉽게 읽을 수 있음.
4. **README.md 산문** *(필터링 후 일반 산문 언급이 남은 경우에만)*. 남은 것이 없으면 건너뜀.

각 커밋 메시지는 단계("Phase A")와 세부 구성("rename CSO to SDO", "agent prose in active skills" 등)을 명시하여 시리즈 자체가 자가 문서화되도록 합니다.

## 검증 (Verification)

각 커밋 이후:

- `grep -rn "Claude" <touched-paths>` 실행 — 남은 모든 항목은 문서화된 예외 항목(모델 이름, 파일 이름, URL, "Claude Code" 플랫폼 이름, 역사적 아티팩트)에 해당해야 함.
- 변경된 파일을 끝까지 읽기 — 치환으로 인해 문장 흐름, 대명사 일치, 목록 병렬 구조가 깨지지 않아야 함.
- 실행할 테스트는 없음; 이 작업은 산문 전용임.

최종 커밋 이후:

- 라이브 세션에서 수정된 각 스킬을 훑어보며 어색하게 읽히는 부분이 없는지 확인.

## 비목표 (Non-goals)

- 동작, 구조, 제목(CSO→SDO 제외), 예시, 코드 블록, YAML 프론트매터를 변경하지 마십시오.
- 새로운 섹션, 콜아웃, 호환성 노트를 도입하지 마십시오.
- 편집하는 동안 치환 작업 이상으로 산문을 "개선"하려 하지 마십시오.
