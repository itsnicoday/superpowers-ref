# Kimi Code용 Superpowers

[Kimi Code](https://github.com/MoonshotAI/kimi-code)에서 Superpowers를 사용하기 위한 전체 가이드입니다.

## 설치

Superpowers는 Kimi Code의 플러그인 마켓플레이스에서 이용할 수 있습니다.

플러그인 관리자를 엽니다:

```text
/plugins
```

`Marketplace` > `Superpowers`로 이동하여 설치합니다.

다음과 같이 이 저장소에서 직접 설치할 수도 있습니다:

```text
/plugins install https://github.com/obra/superpowers
```

`dev` 브랜치에 대한 미출시 검증의 경우, 브랜치를 명시적으로 지정하여 설치합니다:

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

Kimi Code는 새로운 세션부터 플러그인 변경 사항을 적용합니다. 플러그인을 설치, 업데이트, 활성화, 비활성화 또는 다시 로드한 후에는 `/new` 명령어로 새 세션을 시작하십시오.

## 작동 방식

Kimi 플러그인 매니페스트는 `.kimi-plugin/plugin.json`에 위치합니다.

매니페스트는 세 가지 역할을 합니다:

1. Kimi Code가 기존 `skills/` 디렉토리를 가리키도록 함.
2. `sessionStart.skill`을 통해 세션 시작 시 `using-superpowers`를 로드함.
3. `skillInstructions`를 통해 Kimi 전용 도구 매핑을 제공함.

Kimi Code는 이 저장소에서 Superpowers 스킬을 읽어옵니다. 복사된 스킬, 심볼릭 링크, 훅, 또는 추가 런타임 의존성이 존재하지 않습니다.

## 도구 매핑 (Tool Mapping)

스킬은 하나의 런타임에 해당하는 도구 이름을 하드코딩하는 대신 행위(action)를 설명합니다. Kimi Code에서 이들은 다음과 같이 해석됩니다:

- "Ask the user" / "ask clarifying questions" -> `AskUserQuestion`
- "Create a todo" / "mark complete in todo list" -> `TodoList`
- "Dispatch a subagent" -> `Agent`
- "Invoke a skill" -> Kimi Code의 네이티브 `Skill` 도구
- "Read a file" / "write a file" / "edit a file" -> `Read`, `Write`, `Edit`
- "Run a shell command" -> `Bash`
- "Search file contents" -> `Grep`
- "Find files by path or pattern" -> `Glob`
- "Fetch a URL" -> `FetchURL`
- "Search the web" -> `WebSearch`

## 업데이트

Kimi Code의 플러그인 관리자를 사용하십시오:

```text
/plugins
```

Superpowers를 선택하고 거기서 업데이트합니다. 업데이트 후에는 `/new` 명령어로 새 세션을 시작하십시오.

## 문제 해결 (Troubleshooting)

### 플러그인이 로드되지 않음

1. `/plugins info superpowers`를 실행하고 진단 항목을 확인합니다.
2. 플러그인이 활성화되어 있는지 확인합니다.
3. 설치 또는 업데이트 후 `/new` 명령어로 새 세션을 시작합니다.

### 직접 GitHub 설치 시 이전 릴리스가 사용됨

Kimi Code는 베어 저장소 URL에 대해 최신 GitHub 릴리스가 존재하는 경우 이를 설치합니다. 다음 Superpowers 릴리스 전에 미출시 변경 사항을 테스트하려면 브랜치를 명시적으로 지정하여 설치하십시오:

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

### 스킬이 트리거되지 않음

1. `/plugins info superpowers` 명령어가 플러그인이 활성화됨을 표시하는지 확인합니다.
2. `/new` 명령어로 새 세션을 시작합니다.
3. 수락 프롬프트를 시도합니다: `Let's make a react todo list`. 올바르게 설치되었다면 코드를 작성하기 전에 `brainstorming` 스킬을 로드해야 합니다.
