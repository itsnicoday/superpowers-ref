# OpenCode용 Superpowers

[OpenCode.ai](https://opencode.ai)에서 Superpowers를 사용하기 위한 전체 가이드입니다.

## 설치

`opencode.json` (전역 또는 프로젝트 수준)의 `plugin` 배열에 superpowers를 추가하세요:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

OpenCode를 재시작하세요. 플러그인이 OpenCode의 플러그인 매니저를 통해 설치되고 모든 스킬이 등록됩니다.

다음을 물어보아 검증하세요: "Tell me about your superpowers"

OpenCode는 자체 플러그인 설치 방식을 사용합니다. Claude Code, Codex 또는 다른 하네스도 함께 사용하는 경우 각각에 대해 Superpowers를 별도로 설치하세요.

### 이전 심볼릭 링크 기반 설치에서 마이그레이션하기

이전에 `git clone`과 심볼릭 링크를 사용하여 superpowers를 설치했다면 이전 설정을 제거하세요:

```bash
# 이전 심볼릭 링크 제거
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 선택 사항: 클론된 리포지토리 제거
rm -rf ~/.config/opencode/superpowers

# superpowers용으로 추가했던 경우 opencode.json에서 skills.paths 제거
```

그런 다음 위의 설치 단계를 따르세요.

## 사용법

### 스킬 찾기

OpenCode의 네이티브 `skill` 툴을 사용하여 사용 가능한 모든 스킬을 나열하세요:

```
use skill tool to list skills
```

### 스킬 로드하기

```
use skill tool to load brainstorming
```

### 개인 스킬

`~/.config/opencode/skills/`에 자신만의 스킬을 생성하세요:

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

`~/.config/opencode/skills/my-skill/SKILL.md`를 생성하세요:

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 프로젝트 스킬

프로젝트 내의 `.opencode/skills/`에 프로젝트 전용 스킬을 생성하세요.

**스킬 우선순위:** 프로젝트 스킬 > 개인 스킬 > Superpowers 스킬

## 업데이트

OpenCode는 git 기반 패키지 사양을 통해 Superpowers를 설치합니다. 일부 OpenCode 및 Bun 버전은 분기된 git 의존성을 락파일이나 캐시에 고정하므로 재시작 시 최신 Superpowers 커밋이 반영되지 않을 수 있습니다. 업데이트가 표시되지 않는 경우 OpenCode의 패키지 캐시를 비우거나 플러그인을 다시 설치하세요.

특정 버전을 고정하려면 브랜치나 태그를 사용하세요:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 작동 방식

플러그인은 두 가지 작업을 수행합니다:

1. **`experimental.chat.messages.transform` 훅을 통해 부트스트랩 컨텍스트를 주입하여**, 모든 대화에 superpowers 인식을 추가합니다.
2. **`config` 훅을 통해 스킬 디렉토리를 등록하여**, OpenCode가 심볼릭 링크나 수동 설정 없이 모든 superpowers 스킬을 발견하도록 합니다.

### 툴 매핑

스킬은 특정 런타임의 툴 이름을 명시하는 대신 작업 단위로 이야기합니다. OpenCode에서 이것은 다음과 같이 해석됩니다:

- "Create a todo" / "mark complete in todo list" → `todowrite`
- `Subagent (general-purpose):` 템플릿 → `subagent_type: "general"` (또는 코드베이스 탐색의 경우 `"explore"`)과 함께 사용되는 OpenCode의 `task` 툴
- "Invoke a skill" → OpenCode의 네이티브 `skill` 툴
- "Read a file" → `read`
- "Create a file" / "edit a file" / "delete a file" → `apply_patch`
- "Run a shell command" → `bash`
- "Search file contents" / "find files by name" → `grep`, `glob`
- "Fetch a URL" → `webfetch`

(설치된 OpenCode CLI의 툴 인벤토리와 검증됨.)

## 문제 해결

### 플러그인이 로드되지 않음

1. OpenCode 로그 확인: `opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. `opencode.json` 파일의 플러그인 라인이 올바른지 검증
3. 최신 버전의 OpenCode를 실행 중인지 확인

### Windows 설치 문제

일부 Windows OpenCode 빌드에는 `git+https` URL에 대한 캐시 경로 및 표준 터미널에서 실행되는 경우에도 Bun이 `git.exe`를 찾지 못하는 문제를 포함하여, git 기반 플러그인 사양의 업스트림 설치 프로그램 이슈가 존재합니다. OpenCode가 플러그인을 설치할 수 없다면 시스템 npm으로 설치하고 OpenCode가 로컬 패키지를 가리키도록 시도하세요:

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

그런 다음 `opencode.json`에서 설치된 패키지 경로를 사용하세요:

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### 스킬을 찾을 수 없음

1. OpenCode의 `skill` 툴을 사용하여 사용 가능한 스킬을 나열
2. 플러그인이 로드되고 있는지 확인 (위 내용 참조)
3. 각 스킬에는 유효한 YAML 프론트매터가 포함된 `SKILL.md` 파일이 필요함

### 부트스트랩이 나타나지 않음

1. OpenCode 버전이 `experimental.chat.messages.transform` 훅을 지원하는지 확인
2. 설정 변경 후 OpenCode 재시작

## 도움 받기

- 이슈 제보: https://github.com/obra/superpowers/issues
- 메인 문서: https://github.com/obra/superpowers
- OpenCode 문서: https://opencode.ai/docs/
