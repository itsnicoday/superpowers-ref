# Visual Companion 최종 보안 강화 수정 설계 (Final Hardening Fixup Design)

**날짜:** 2026-06-11  
**상태:** Drew 검토용 초안

## 목표

PR #1720의 visual companion 보안 강화 패스를 마무리하여, 깨끗한 보안 동작, 결정론적 테스트, 동반 작업만 포함된 PR diff를 갖추어 Jesse 검토 준비를 완료합니다.

이 작업은 기존 인증 강화 설계를 바탕으로 하는 수정(fixup) 작업입니다. Companion을 재설계하거나 기능 범위를 확장해서는 안 됩니다.

## 배경

이전 보안 강화 패스에서는 키가 포함된 세션, 동일 출처 WebSocket 검사, URL 키 제거, `/files/*` 격리, 유출 방지 헤더, IPv6 URL 포맷팅, Windows 라이프사이클 커버리지, PR 증거 업데이트가 추가되었습니다.

최종 검토 패스에서 남은 5가지 이슈가 발견되었습니다:

1. 루트 `GET /` 스크린 선택 경로가 `content/` 아래에 존재하면서 콘텐츠 디렉토리 외부를 가리키는 심볼릭 링크나 하드링크를 여전히 서빙할 수 있음.
2. 선호하는 포트가 점유되어 있을 때, 폴백 서버가 영속화된 `.last-token`을 재사용하여 동일한 베어러 키를 가진 두 개의 라이브 동일 프로젝트 companion 서버가 생성될 수 있음.
3. 강력한 소유권 증명이 불가능할 때 `stop-server.sh`가 무관한 `node server.cjs` 프로세스에 시그널을 보낼 수 있음.
4. 일부 테스트가 잘못된 폴백 프로세스를 대상으로 통과하거나, 실패 시 백그라운드 프로세스를 유출하거나, Windows 유사 호스트에서 심볼릭 링크 지원을 전제할 수 있음.
5. 별도로 처리된 이전 `evals` 서브모듈 범프가 브랜치에 포함되어 있어 PR이 현재 충돌 상태임.

## 비목표 (Non-Goals)

- 이번 패스에서 HTTPS 터널이나 `wss://` 출처 의미론을 추가하지 마십시오.
- 비활성화(opt-out), 자유 형식 텍스트, 대비 헬퍼 companion 기능을 구현하지 마십시오.
- Alpine, Three.js 또는 다른 어떤 JavaScript 라이브러도 벤더링하지 마십시오.
- 에이전트가 작성한 악성 스크린 HTML을 샌드박스 처리하려 하지 마십시오.
- Drew가 해당 절충안을 명시적으로 승인하지 않는 한 오래된 stop-server PID 파일에 대한 하위 호환성을 추가하지 마십시오.

## 상속된 보안 불변 조건

이 수정 작업은 이미 설계 및 구현된 인증 강화를 보존합니다:

- `.last-token` 및 `state/server-info`는 소유자 전용의 민감한 상태로 남아있음.
- 폴백 토큰은 시작 JSON 및 `state/server-info`에 나타날 수 있지만, `.last-token`에 작성되어서는 안 됨.
- 쿠키는 포트 이름이 지정되고, `HttpOnly`, `SameSite=Strict`이며, `/` 범위로 고정됨.
- WebSocket 업그레이드는 여전히 유효한 키 또는 쿠키를 요구함.
- 브라우저가 `Origin` 헤더를 제공할 때 WebSocket `Origin` 검사가 계속 강제됨.
- 직접 연결하는 `Origin` 없는 클라이언트는 세션 키를 가진 경우에만 허용됨.
- 생성된 동일 출처 스크린 JavaScript 및 향후 동일 출처 벤더링 라이브러리는 신뢰됨. 악성 스크린 HTML 샌드박스화는 계속 연기됨.

## 설계

### 1. 현재 `dev` 위로 리베이스 (Rebase Onto Current `dev`)

구현 작업 전 `brainstorming-companion`을 현재 `origin/dev` 위로 리베이스합니다. `dev`를 채택하여 `evals` 서브모듈 충돌을 해결합니다.

리베이스 후:

- `evals`가 PR diff에 나타나지 않아야 함.
- PR #1720은 다른 곳에서 실행된 이발 증거를 언급할 수 있지만, 명확한 외부 증거(이발 저장소 커밋, 시나리오 경로, 명령어, 결과 아티팩트 경로 또는 ID, RED/GREEN 결과)를 포함해야 함.
- PR 본문이 서브모듈 범프가 이 PR의 일부임을 암시해서는 안 됨.
- 서브모듈 범프가 포함되었음을 암시하는 이전 PR 본문 텍스트나 댓글은 최종 PR 본문 증거에 의해 대체되어야 함.

### 2. 루트 스크린 격리 (Root Screen Containment)

루트 스크린 라우트는 `/files/*`와 동일한 격리 경계를 사용해야 합니다.

`getNewestScreen()`은 일반 파일-콘텐츠 디렉토리 내부 가드를 통과하지 못하는 `.html` 후보를 무시해야 합니다. 해당 가드는 실제 경로를 해석하고 서빙되는 파일이 `CONTENT_DIR` 내부인지 확인해야 합니다. 또한 플랫폼이 링크 수를 보고할 때 링크 수가 정확히 1이 아닌 파일을 거부함으로써 기존 하드링크 보호를 보존해야 합니다.

예상 동작:

- `content/` 아래에서 `content/` 외부를 가리키는 심볼릭 링크는 무시됨.
- `fs.linkSync`가 성공하고 `lstat.nlink > 1`일 때 `state/server-info`로 연결된 `content/` 아래의 하드링크는 무시됨.
- 안전한 스크린 파일이 남지 않은 경우 대기 페이지가 서빙됨.
- 기존 `/files/*` 격리 동작은 변경되지 않고 유지됨: 빈 이름, 도트파일, 심볼릭 링크, 하드링크, 디렉토리는 여전히 404를 반환함.

### 3. 폴백 토큰 격리 (Fallback Token Isolation)

포트 폴백은 영속화된 `.last-token`에서 로드된 토큰을 재사용해서는 안 됩니다.

토큰 소스는 코드에서 명시적이어야 합니다:

- 환경 변수의 `BRAINSTORM_TOKEN`은 명시적인 운영자/테스트 재정의입니다. 명시적 환경 토큰이 설정된 동안 선호 포트가 점유되어 있으면, 점유된 서버가 동일한 명시적 토큰을 사용 중일 수 있으므로 서버는 폴백하지 않고 실패(fail closed)해야 합니다.
- `.last-token`은 동일 포트 재연결 편의를 위한 영속화된 상태입니다. 선호 포트가 점유되어 서버가 폴백하는 경우, 로드된 토큰을 버리고 폴백 프로세스를 위해 영속화되지 않는 신규 토큰을 생성합니다.
- `.last-token`에서 로드되지 않은 새로 생성된 토큰은 다른 라이브 프로세스가 이를 가지고 있는 것으로 알려지지 않았기 때문에 동일 프로세스 내에서 재사용될 수 있습니다.

폴백 서버는 `.last-port` 및 `.last-token` 덮어쓰기를 피하도록 계속 작동해야 합니다.

### 4. Stop-Server 소유권 증명

`start-server.sh`는 시작 시 서버 인스턴스 ID를 생성하고 무해한 command-line 인자로 Node에 전달해야 합니다. 예시:

```text
node server.cjs --brainstorm-server-id=<id>
```

이 ID는 인증 자격 증명이 아닙니다. 로컬 라이프사이클 스크립트를 위한 프로세스 소유권 증명일 뿐입니다. `server.cjs`는 이 인자를 무시할 수 있습니다.

ID는 `^[A-Za-z0-9_-]{32,64}$`와 같이 쉘/MSYS에 안전한 알파벳을 사용해야 합니다. 소유자 전용 권한으로 `state/server-instance-id`에 저장합니다.

`stop-server.sh`는 상태에서 예상 ID를 읽고 대상 프로세스 argv가 불완전한 부분 문자열이 아닌 전체 argv 토큰으로서 정확한 인자 `--brainstorm-server-id=<id>`를 포함할 때만 PID에 시그널을 보내야 합니다. 가능하면 `/proc/<pid>/cmdline`을 우선 사용하고, 광범위한 `ps` 출력으로 폴백합니다. 일치하는 인스턴스 ID는 `server-info`가 누락되거나 `lsof`를 사용할 수 없는 경우에도 충분한 증명이 됩니다. 기존 포트-PID 검사는 추가 증거로 남겨둘 수 있습니다.

소유권을 증명할 수 없을 때 실패로 처리:

- PID 파일 누락
- 서버 ID 누락 또는 잘못 형성됨
- 대상 커맨드 라인 사용 불가
- 대상 커맨드 라인에 예상 ID가 포함되어 있지 않음
- 새 ID가 없는 오래된/남아있는 세션 메타데이터

이는 무관한 프로세스를 종료시키는 것보다 오래된 프로세스를 살려두는 것을 의도적으로 선호합니다.

운영자 대상 결과는 명시적이어야 합니다:

- 누락된 PID 파일은 `not_running` 반환
- 누락되거나 잘못 형성된 서버 ID는 `stale_pid` 반환
- 사용 불가능한 커맨드 라인은 `stale_pid` 반환
- 잘못되거나 누락된 argv ID는 `stale_pid` 반환
- 성공적인 종료는 `stopped` 반환

`stale_pid` 및 `stopped` 결과 시, 향후의 종료 시도가 동일한 모호한 프로세스를 계속 타겟팅하지 않도록 `server.pid` 및 `server-instance-id`를 제거합니다. 영속 세션 콘텐츠는 제거하지 마십시오.

### 5. 테스트 강화

테스트 패스는 검증에 사용되는 macOS 및 Windows Git Bash 호스트 전반에서 결정론적이어야 합니다.

필수 변경사항:

- 고정 포트 수트는 서버가 폴백 포트를 보고하는 경우 빠르게 실패하거나, 보고된 시작 포트에서 모든 클라이언트를 구동해야 함.
- `stop-server.test.sh`는 백그라운드 프로세스가 시작되기 전에 최상위 정리 트랩(cleanup trap)을 필요로 함.
- 심볼릭 링크 전용 어설션은 심볼릭 링크 기능을 탐색하고 호스트가 사용 가능한 테스트 심볼릭 링크를 생성할 수 없을 때 해당 어설션만 건너뛰어야 함.
- 사칭(impostor) 프로세스를 생성하는 테스트는 라이프사이클 메타데이터가 누락되거나 불충분할 때 사칭 프로세스가 살아남음을 어설션해야 함.
- Windows/MSYS 시작 서버 테스트는 Windows 유사 감지가 여전히 `BRAINSTORM_OWNER_PID`를 지우고, 적절할 때 여전히 자동 포그라운드 처리하며, 인스턴스 ID argv를 정확히 전달함을 어설션해야 함.

### 6. 문서 및 PR 일관성

Jesse 검토 전, 검토자 대상 문서 및 PR 메타데이터를 조율합니다:

- 이슈 카탈로그를 업데이트하여 조치 내용이 이 PR이 실제로 배송하는 것과 일치하도록 함.
- 구현된 `--open` 동작과 일치하도록 자동 열기 문서를 유지함.
- 문서화된 기본 유휴 타임아웃을 모든 곳에서 4시간으로 유지함.
- 리베이스 후 템플릿에 맞추어 PR 본문을 검토함.
- 구체적인 명령어 및 결과와 함께 macOS, Windows, 브라우저/수동, 외부 이발 증거를 PR 본문에 기록함.

## 테스트 전략

각 동작 변경에 TDD를 사용합니다:

1. 집중 회귀 테스트 추가 또는 강화.
2. 실행하고 예상된 이유로 실패하는지 확인.
3. 가장 작은 수정사항 구현.
4. 집중 테스트 재실행.
5. 전체 brainstorm-server 수트 재실행.

필수 집중 회귀 테스트 항목:

| Behavior | Test File | Focused Command | Expected RED | Expected GREEN |
| --- | --- | --- | --- | --- |
| 루트 라우트가 심볼릭 링크 이탈 무시 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 인증된 `GET /`가 외부 콘텐츠 링크 서빙 | 응답이 대기 페이지 또는 안전한 스크린 서빙 |
| 루트 라우트가 지원되는 하드링크 이탈 무시 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 인증된 `GET /`가 하드링크된 `server-info` 서빙 | `nlink > 1`일 때 하드링크 후보 무시됨 |
| `/files/*` 격리가 변경 없이 유지됨 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 기존 격리 테스트 퇴보 | 빈 값, 도트파일, 디렉토리, 심볼릭 링크, 하드링크 케이스 404 유지 |
| 영속 토큰 폴백이 토큰 회전 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 폴백 URL 키가 영속된 선호 포트 키와 동일함 | 폴백 URL 키가 달라지며 `.last-token`에 작성되지 않음 |
| 명시적 토큰 폴백이 실패로 처리됨 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | `BRAINSTORM_TOKEN`이 설정된 동안 서버 폴백 발생 | 프로세스가 0이 아닌 값으로 종료되고 폴백 시작 안 함 |
| 폴백 키가 원래 서버 인증 불가 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 폴백 키가 원래 포트로부터 200 수신 | 원래 포트가 폴백 키 거부 |
| 올바른 인스턴스 ID가 정지 허용 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 실제 start-server로 실행된 서버가 살아남음 | 정지가 `stopped`를 반환하고 프로세스 종료됨 |
| 잘못되거나 누락되거나 잘못 형성되거나 오래된 ID가 안전함 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 사칭 프로세스에 시그널 전송 | 정지가 `stale_pid`를 반환하고 사칭 프로세스 살아남음 |
| 고정 포트 수트가 폴백을 통해 통과 불가 | `tests/brainstorm-server/server.test.js`, `tests/brainstorm-server/auth.test.js` | 각각의 `node` 명령어 | 테스트가 폴백 포트와 암묵적으로 통과 | 테스트가 명확히 실패하거나 의도적으로 보고된 포트 사용 |
| 실패 시 쉘 정리 트랩 실행 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 실패가 자식 프로세스를 남김 | 트랩이 백그라운드 자식 프로세스 수거 |
| Windows/MSYS 시작 동작이 라이프사이클 불변 조건 유지 | `tests/brainstorm-server/start-server.test.sh`, `tests/brainstorm-server/windows-lifecycle.test.sh` | macOS 및 `ballmer`에서의 `bash` 테스트 명령어 | 소유자 PID 또는 argv 처리가 퇴보함 | 소유자 PID 정리됨, 포그라운드 감지 유지됨, ID argv 존재함 |

각 RED/GREEN 사이클은 PR 본문을 위한 짧은 증거 노트를 남겨야 합니다: 집중 명령어, 수정 전 실패 어설션, 수정 후 통과 어설션, 증거 수집 환경(macOS 또는 Windows).

## 검증 (Verification)

수정 완료 선언 전 다음을 실행:

- `git fetch origin dev && git rebase origin/dev`
- `git diff --quiet origin/dev...HEAD -- evals`
- `gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid`
- `cd tests/brainstorm-server && npm test`
- TDD 중 사용된 관련 집중 테스트 명령어
- `git diff --check`
- 수정된 JavaScript 파일들에 대한 Node 문법 검사
- 수정된 쉘 파일들에 대한 shell lint
- `ballmer`에서의 Windows 검증: 전체 실행 가능한 brainstorm-server 수트 및 독립 실행형 Windows 라이프사이클 프로브

자동화 패스가 초록색을 기록한 후에만 수동/브라우저 테스트를 진행합니다.

## 수탁 기준 (Acceptance Criteria)

- PR #1720이 현재 `dev` 위로 깨끗하게 리베이스됨.
- `evals`가 PR diff에 포함되지 않음.
- 루트 스크린 서빙이 심볼릭 링크나 지원되는 하드링크 이탈을 통해 `content/` 외부를 읽을 수 없음.
- `/files/*` 격리 보호가 변경 없이 유지됨.
- 어떤 폴백 서버도 점유된 선호 포트 서버와 공유될 수 있는 토큰으로 실행되지 않음.
- 소유권 증명이 누락되거나 모호할 때 `stop-server.sh`가 무관한 프로세스에 시그널을 보내지 않음.
- `server-info`나 `lsof`를 사용할 수 없을 때도 `stop-server.sh`가 일치하는 인스턴스 ID를 가진 정상 서버를 종료할 수 있음.
- 각 회귀 항목에 대해 집중 RED/GREEN 증거가 기록됨.
- macOS 및 Windows 검증 증거가 PR 본문에 기록됨.
- PR 본문이 브랜치에 있는 내용과 외부에서 수집된 증거를 정확하게 설명함.
