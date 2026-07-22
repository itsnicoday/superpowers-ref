# Claude Code를 위한 크로스 플랫폼 폴리글랏 훅

Claude Code 플러그인에는 Windows, macOS 및 Linux에서 작동하는 훅이 필요합니다. 이 문서는 `hooks/run-hook.cmd`에서 사용되는 단일 범용 디스패처 패턴을 설명합니다.

> **권위 있는 출처:** `hooks/run-hook.cmd`가 정식 구현입니다. 이 문서와 코드가 다를 경우 코드를 신뢰하세요.

## 문제점

Claude Code는 시스템의 기본 셸을 통해 훅 명령을 실행합니다:
- **Windows**: CMD.exe
- **macOS/Linux**: bash 또는 sh

이로 인해 몇 가지 과제가 발생합니다:

1. **스크립트 실행**: Windows CMD는 `.sh` 파일을 직접 실행할 수 없음
2. **경로 형식**: Windows는 백슬래시(`C:\path`)를 사용하고 Unix는 슬래시(`/path`)를 사용함
3. **환경 변수**: `$VAR` 구문이 CMD에서 작동하지 않음
4. **`.sh` 자동 접두사 추가**: Windows의 Claude Code는 경로에 `.sh`가 포함된 모든 명령 앞에 자동으로 `bash`를 덧붙입니다 — 이는 스크립트에 확장자가 있는 경우 디스패처를 방해합니다.

## 해결책: 확장자 없는 스크립트 + 단일 범용 디스패처

이 리포지토리는 모든 훅에 대해 하나의 범용 `run-hook.cmd` 디스패처를 사용합니다. 훅 스크립트는 **확장자가 없습니다** (`session-start.sh`가 아닌 `session-start`). 이것은 의도적인 설계입니다: Claude Code의 Windows 자동 감지가 디스패처 명령 앞에 `bash`를 덧붙여 디스패처를 고장내는 것을 방지합니다.

### 파일 구조

```
hooks/
├── hooks.json          # 확장자 없는 스크립트 이름으로 run-hook.cmd를 가리킴
├── run-hook.cmd        # 크로스 플랫폼 디스패처 (폴리글랏 래퍼)
└── session-start       # 실제 훅 로직 — 확장자 없는 bash 스크립트
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

`${CLAUDE_PLUGIN_ROOT}`에 공백이 포함될 수 있으므로 경로는 큰따옴표로 감싸집니다.

## `run-hook.cmd`의 하이 레벨 작동 방식

`run-hook.cmd`는 폴리글랏 스크립트입니다: Windows는 첫 번째 블록을 배치 명령으로 취급하는 반면, Unix 셸은 해당 블록을 무시 항목(no-op) heredoc으로 취급하고 그 뒤에서 계속 진행합니다.

이 문서에서 구현을 복사하지 마세요. 디스패처를 변경할 때는 `hooks/run-hook.cmd`를 직접 읽고 그 후에 `tests/hooks/test-session-start.sh`를 실행하세요.

### Windows (CMD.exe)에서의 작동 방식

1. 배치 섹션은 스크립트 이름을 검증하고 디스패처 자체의 위치에서 훅 디렉토리를 구합니다.
2. 다음 세 위치에서 bash를 시도합니다:
   - `C:\Program Files\Git\bin\bash.exe`
   - `C:\Program Files (x86)\Git\bin\bash.exe`
   - `PATH` 상의 `bash` (MSYS2, Cygwin 또는 기본값이 아닌 Git 설치)
3. bash를 찾으면 훅 디렉토리에서 지정된 확장자 없는 훅 스크립트를 실행합니다.
4. bash를 찾지 못하면 디스패처는 소리 없이 `0`으로 종료됩니다 — 플러그인은 계속 작동하며 단지 훅을 건너뛸 뿐입니다.
5. `exit /b`는 CMD가 Unix 섹션에 도달하기 전에 차단합니다.

### Unix (bash/sh)에서의 작동 방식

1. `: << 'CMDBLOCK'`은 무시 항목(no-op) 명령에서 heredoc을 엽니다.
2. 전체 CMD 배치 블록은 heredoc에 의해 소비되어 무시됩니다.
3. `CMDBLOCK` 이후 bash는 스크립트 디렉토리를 해독하고 지정된 확장자 없는 스크립트를 직접 `exec`합니다.

### 핵심 설계 결정 사항

| 결정 사항 | 이유 |
|----------|-----|
| 확장자 없는 스크립트 | Claude Code의 Windows `.sh` 자동 접두사 추가가 디스패처 명령을 방해하는 것을 방지 |
| `-l` (로그인 셸) 없음 | 필요하지 않음; 훅 스크립트는 자체 포함되어야 하며 로그인 셸 PATH 설정에 의존해서는 안 됨 |
| `cygpath` 없음 | Bash가 Windows 경로를 직접 받아 올바르게 처리함; `cygpath`는 이전의 `-c "..."` 호출 패턴에 필요했던 것이지 직접 실행에는 필요하지 않음 |
| bash 부재 시 소리 없는 종료 | Git for Windows가 없는 사용자에 대한 플러그인 손상을 방지함; 훅 컨텍스트 주입은 정상적으로 건너뜀 |

## 크로스 플랫폼 훅 스크립트 작성하기

훅 로직은 확장자 없는 스크립트 파일에 들어갑니다. 몇 가지 이식 가능한 패턴:

### 권장 사항
- 가능한 경우 순수 bash 빌트인을 사용
- 백틱 대신 `$(command)` 사용
- 모든 변수 확장에 큰따옴표 사용: `"$VAR"`

### 피해야 할 사항
- 대체 방법 없이 PATH 의존적 툴에 의존하기 (훅은 `-l` 없이 실행되므로 로그인 셸 PATH가 설정되지 않음)
- 스크립트에 `.sh` 확장자 부여하기 — 이는 Claude Code의 Windows 자동 접두사 추가를 트리거함

### 예시: 외부 툴 없는 JSON 이스케이프 처리

```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 문제 해결

### "bash is not recognized"

CMD가 디스패처가 시도하는 세 위치 중 어느 곳에서도 bash를 찾지 못했습니다. 디스패처는 오류를 내는 대신 소리 없이 종료(0)되므로 훅을 건너뜁니다. 표준 경로에 Git for Windows를 설치하거나 `PATH`에 `bash`가 있는지 확인하세요.

### 훅이 Unix에서는 실행되지만 Windows에서는 아무 작업도 하지 않음

`hooks.json`에서 스크립트 파일 이름에 **확장자가 없는지** 확인하세요. `run-hook.cmd session-start.sh`와 같은 명령은 Claude Code의 `.sh` 자동 감지를 트리거하여 의도한 CMD 디스패처 경로를 우회하거나 존재하지 않는 `session-start.sh` 스크립트를 실행하려고 시도할 수 있습니다.

### 훅이 전혀 실행되지 않음

`hooks.json`의 `matcher`가 하네스가 내보내는 이벤트 타입과 일치하는지 확인하세요. Claude Code는 `startup|clear|compact`를 사용하고; Cursor는 `sessionStart`를 사용합니다. Cursor 변형에 대해서는 `hooks-cursor.json`을 확인하세요.

## 관련 이슈

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) — Windows에서 `.sh` 스크립트가 에디터에서 열림
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) — Windows에서 훅이 작동하지 않음
