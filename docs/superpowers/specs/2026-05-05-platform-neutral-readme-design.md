# 플랫폼 중립적 README 정렬 — Phase C 설계

## 배경

Phase A 및 B(`2026-05-05-platform-neutral-prose-design.md` 및 `2026-05-05-platform-neutral-config-refs-design.md` 참조)를 통해 README에 있던 일반적인 Claude 산문과 설정 파일 참조가 이미 중립화되었습니다. 남아 있는 플랫폼 치우침 신호는 레이아웃입니다: README의 두 플랫폼 나열에서 Claude Code가 가장 먼저 나오고 다른 부분에서도 엄격하게 알파벳순으로 정렬되어 있지 않습니다.

이 단계는 정렬 순서를 수정합니다. 산문 변경은 없습니다.

## 작업 범위 (In scope)

1. **빠른 시작 플랫폼 목록** (`README.md:7`) — 지원되는 하네스의 인라인 링크 목록
2. **설치 섹션 순서** (`README.md:35–152`) — 하네스별 설치 서브섹션

## 작업 범위 외 (Out of scope)

- 산문, 마켓플레이스 이름, 플러그인 ID, URL — 모두 있는 그대로 사실에 부합함.
- Claude Code 섹션의 시각적 비중 (공식 Anthropic 마켓플레이스 및 Superpowers 마켓플레이스의 두 서브섹션을 가짐). 둘 다 실제 설치 경로입니다; 이를 축소하면 정확한 정보가 가려집니다.
- 각 설치 블록 내부의 섹션 제목 및 콘텐츠 — 블록의 정렬 순서만 변경됩니다.

## 치환 (Substitution)

두 나열 모두 엄격한 알파벳순으로 재정렬됩니다:

| Old order | New order |
|-----------|-----------|
| Claude Code | Claude Code |
| Codex CLI | Codex App |
| Codex App | Codex CLI |
| Factory Droid | Cursor |
| Gemini CLI | Factory Droid |
| OpenCode | Gemini CLI |
| Cursor | GitHub Copilot CLI |
| GitHub Copilot CLI | OpenCode |

세 가지 이동: Codex App이 Codex CLI와 위치를 바꿈; Cursor가 두 칸 위로 이동; GitHub Copilot CLI가 한 칸 위로 이동.

Claude Code는 알파벳 우연에 의해 첫 번째로 남아 있습니다 (`Cl…`이 `Co…`보다 앞섬).

## 커밋 계획

한쪽만 변경하고 다른 쪽을 남겨두면 빠른 시작과 설치 섹션 간 불일치가 발생하므로 두 나열을 모두 다루는 하나의 원자적 커밋으로 진행합니다.

## 검증 (Verification)

- 빠른 시작 앵커 (`#claude-code`, `#codex-app` 등)가 기존 `### …` 제목으로 여전히 올바르게 확인됨 — 제목 이름 변경 없음.
- 각 설치 서브섹션의 본문이 변경 전/후 바이트 단위로 동일함; 위치만 변경됨.
- `git diff README.md` 실행 시 섹션 이동만 표시되고 콘텐츠 편집은 없음.
