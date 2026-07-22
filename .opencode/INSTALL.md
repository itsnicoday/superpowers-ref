# OpenCode용 Superpowers 설치 가이드

## 사전 요구 사항

- [OpenCode.ai](https://opencode.ai) 설치 완료

## 설치 방법

`opencode.json` (전역 또는 프로젝트 수준)의 `plugin` 배열에 superpowers를 추가합니다:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

OpenCode를 재시작합니다. 플러그인이 OpenCode의 플러그인 관리자를 통해 설치되고 모든 스킬을 등록합니다.

다음 질문을 통해 설치를 확인하세요: "Tell me about your superpowers"

OpenCode는 자체 플러그인 설치 체계를 사용합니다. Claude Code, Codex 또는 다른 하네스도 함께 사용하는 경우 각각에 대해 Superpowers를 별도로 설치하세요.

## 기존 심볼릭 링크 기반 설치에서 마이그레이션

이전에 `git clone` 및 심볼릭 링크를 사용하여 superpowers를 설치한 경우 기존 설정을 제거하세요:

```bash
# 구 심볼릭 링크 제거
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 선택 사항: 클론된 저장소 제거
rm -rf ~/.config/opencode/superpowers

# superpowers를 위해 추가했던 경우 opencode.json에서 skills.paths 제거
```

그 후 위의 설치 단계를 따르세요.

## 사용법

OpenCode의 네이티브 `skill` 도구를 사용하세요:

```
use skill tool to list skills
use skill tool to load brainstorming
```

## 업데이트

OpenCode는 git 기반 패키지 명세를 통해 Superpowers를 설치합니다. 일부 OpenCode 및 Bun 버전은 확인된 git 의존성을 락파일이나 캐시에 고정하므로 재시작 시 최신 Superpowers 커밋을 가져오지 못할 수 있습니다. 업데이트가 반영되지 않으면 OpenCode의 패키지 캐시를 비우거나 플러그인을 재설치하세요.

특정 버전을 고정하려면:

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 문제 해결 (Troubleshooting)

### 플러그인이 로드되지 않는 경우

1. 로그 확인: `opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. `opencode.json` 파일의 플러그인 줄 확인
3. 최신 버전의 OpenCode를 실행 중인지 확인

### Windows 설치 문제

일부 Windows OpenCode 빌드에서는 `git+https` URL의 캐시 경로 및 일반 터미널에서 `git.exe`가 동작함에도 Bun이 찾지 못하는 문제 등 git 기반 플러그인 명세와 관련한 업스트림 설치 프로그램 이슈가 있습니다. OpenCode가 플러그인을 설치하지 못하는 경우 시스템 npm으로 설치하고 OpenCode가 로컬 패키지를 가리키도록 설정하세요:

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

그 후 `opencode.json`에서 설치된 패키지 경로를 사용하세요:

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### 스킬을 찾을 수 없는 경우

1. `skill` 도구를 사용하여 탐색된 목록 확인
2. 플러그인이 정상적으로 로드되는지 확인 (위 내용 참조)

### 도구 매핑 (Tool mapping)

스킬은 동작 지향 언어로 전달됩니다("할 일 생성", "서브에이전트 디스패치", "파일 읽기"). OpenCode에서는 다음과 같이 해석됩니다:

- "Create a todo" / "mark complete in todo list" → `todowrite`
- `Subagent (general-purpose):` 템플릿 → `task` 도구 (`subagent_type: "general"`, 코드베이스 탐색 시 `"explore"`)
- "Invoke a skill" → OpenCode 네이티브 `skill` 도구
- "Read a file" → `read`
- "Create a file" / "edit a file" / "delete a file" → `apply_patch`
- "Run a shell command" → `bash`
- "Search file contents" / "find files by name" → `grep`, `glob`
- "Fetch a URL" → `webfetch`

## 도움받기 (Getting Help)

- 이슈 제보: https://github.com/obra/superpowers/issues
- 전체 문서: https://github.com/obra/superpowers/blob/main/docs/README.opencode.md
