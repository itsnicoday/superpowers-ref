# Visual Companion 보안 강화 설계 (Auth Hardening Design)

**날짜:** 2026-06-10  
**상태:** Drew 검토용 초안

## 목표

companion의 핵심 워크플로우를 변경하거나 런타임 의존성을 추가하지 않고, PR #1720의 brainstorming visual companion에서 발견된 보안 및 신뢰성 공백을 수정합니다.

수정사항은 테스트 우선(test-first)으로 진행되어야 하며, 다음에 대한 명확한 자동화 증거를 남겨야 합니다:

- 교차 출처 브라우저 탭이 쿠키를 도용하여 companion 이벤트를 주입할 수 없음
- 재시작 후 재연결이 브라우저 쿠키 동작에만 의존하지 않고 작동함
- 부트스트랩 후 베어러(bearer) 키가 가시적 URL에 남아있지 않음
- `/files/*`가 콘텐츠 디렉토리 외부의 파일을 서빙할 수 없음
- 향후 동일 출처 벤더링 UI 라이브러리가 여전히 작동함

## 위협 모델

companion은 단일 브레인스토밍 세션을 위해 에이전트가 생성한 로컬 UI를 서빙합니다. 주요 자산은 다음과 같습니다:

- companion에서 서빙되는 스크린 콘텐츠
- 세션 키
- 에이전트가 사용자 피드백으로 읽어오는 `state/events`
- companion 세션 디렉토리 아래의 로컬 파일들

범위 내 공격자:

- 다른 `localhost` 포트에 있는 악성 브라우저 탭
- companion에 요청을 보낼 수 있지만 companion UI로 인증되어서는 안 되는 브라우저 페이지
- 서버가 비-루프백 인터페이스에 바인딩된 경우의 직접 원격 클라이언트
- URL 기록, 리퍼러(referrer), 또는 커밋된 로컬 상태를 통한 우발적 유출
- `/files/*`를 우회하는 콘텐츠 디렉토리 심볼릭 링크 또는 경로 속임수

이번 수정 범위 밖:

- 에이전트가 작성한 악성 스크린 HTML
- companion 스크린에 의해 로드된 악성 동일 출처 벤더링 JavaScript

이 범위 밖 경계는 의도된 것입니다. Companion 스크린은 에이전트 UI 영역의 일부입니다. 현재 인라인 스크립트를 사용할 수 있으며 언젠가 Alpine이나 Three.js 같은 동일 출처 벤더링 라이브러리를 사용할 수도 있습니다. 악성 스크린 HTML로부터 보호하려면 좁은 메시지 브릿지를 갖춘 더 큰 샌드박스 처리된 iframe 아키텍처가 필요합니다; 그것은 이 PR 보안 강화 패스의 범위가 아닙니다.

## 현재 결함 사항

자동화 및 헤드형 브라우저 테스트 결과 PR 브랜치에서 다음 결함이 발견되었습니다:

1. 교차 출처 localhost 페이지가 쿠키 인증된 WebSocket을 열고 실제 companion 페이지가 쿠키를 설정한 후 `state/events`에 공격자 제어 선택사항을 작성할 수 있음.
2. `/files/*`가 키가 포함된 URL을 담고 있는 `state/server-info` 심볼릭 링크를 포함하여 `content/` 외부를 가리키는 심볼릭 링크를 서빙함.
3. 세션 키가 실제 스크린 페이지의 URL에 남아있어, 동일 출처 스크린 JavaScript 및 우발적인 리퍼러/기록에 노출될 수 있음.
4. 헬퍼가 키 없는 `ws://host` URL로 재연결함. 헤드형 Chrome에서 동일 포트/동일 토큰 재시작 후 브라우저가 재시작된 서버에 쿠키 제공을 중단하여, 열린 탭이 수동 새로고침 전까지 툼스톤 상태에 갇혀 있음.
5. Codex에서 테스트 통과가 안정적이도록 shell lint 및 라이프사이클 테스트 정리 필요.

## 설계

### 1. 부트스트랩 키 로드 (Bootstrap Keyed Loads)

`GET /?key=<token>`은 스크린 응답이 아닌 부트스트랩 응답이 됩니다.

키가 유효할 때 서버는:

1. 오늘날과 같이 HttpOnly 세션 쿠키를 설정함
2. 작은 HTML 부트스트랩 페이지를 반환함
3. 부트스트랩 페이지가 키를 탭 범주의 `sessionStorage`에 저장함
4. 부트스트랩 페이지가 `location.replace('/')`를 사용하여 `/`로 이동함

이후 가시적 스크린 URL은 `/?key=...`가 아닌 순수한 `/`가 됩니다.

유효한 쿠키가 포함된 `GET /`는 현재 스크린을 서빙합니다. 유효한 쿠키가 없는 `GET /`는 여전히 친절한 403 페이지를 반환합니다. `GET /?key=<wrong>`은 403을 반환합니다.

`sessionStorage`를 사용하는 이유: 헬퍼는 동일 포트 재시작에서도 살아남고 쿠키 동작에만 의존하지 않는 재연결 자격 증명이 필요합니다. 스크린 HTML은 신뢰할 수 있는 동일 출처 UI이므로, 탭 범주 저장소에 키를 저장하는 것은 이 위협 모델에서 수용 가능합니다. 이는 주소창, 기록, 리퍼러 영역에 키를 남겨두는 것보다 실질적으로 훨씬 낫습니다.

### 2. WebSocket 동일 출처 강제 (Same-Origin Enforcement)

WebSocket 업그레이드는 두 가지 검사를 모두 통과해야 합니다:

1. 쿼리 키 또는 쿠키에 의한 유효한 세션 인증
2. `Origin` 헤더가 존재하는 경우 요청 대상 출처와 일치해야 함

출처 검사는 다음을 비교해야 합니다:

```text
Origin === "http://" + req.headers.host
```

브라우저 공격자 페이지 예시:

```text
Origin: http://localhost:9999
Host: localhost:58088
```

브라우저가 companion 쿠키를 보내더라도 이는 거부되어야 합니다.

정상적인 companion 페이지 예시:

```text
Origin: http://localhost:58088
Host: localhost:58088
```

키 또는 쿠키가 유효할 때 이는 승인되어야 합니다.

직접 연결하는 비-브라우저 클라이언트는 `Origin`을 누락할 수 있습니다; 이들은 여전히 세션 키가 필요합니다.

### 3. 헬퍼 재연결 자격 증명 (Helper Reconnect Credential)

`helper.js`는 `sessionStorage`에서 탭 범주 키를 읽어 WebSocket URL에 추가해야 합니다:

```text
ws://<host>/?key=<stored-key>
```

저장된 키가 존재하지 않으면, 헬퍼는 현재의 쿠키 전용 `ws://<host>` 동작으로 폴백합니다. 이는 유효한 쿠키는 있지만 저장소 항목이 없는 이미 로드된 페이지에 대한 호환성을 보존합니다.

### 4. `/files/*` 격리 (Containment)

파일 서버는 빈 이름과 도트파일을 계속 거부해야 합니다. 또한 파일이 `CONTENT_DIR` 내부의 실제 일반 파일인지 확인해야 합니다.

realpath 격리를 경계로 사용합니다:

- `realContentDir = fs.realpathSync(CONTENT_DIR)` 계산
- `realFilePath = fs.realpathSync(filePath)` 계산
- `realFilePath`가 `realContentDir` 하위 경로와 일치할 때만 서빙
- 심볼릭 링크 및 콘텐츠 디렉토리 외부의 모든 것은 404로 거부

서버는 중첩된 경로가 지원되지 않도록 `path.basename`을 계속 사용해야 합니다.

### 5. 유출 방지 헤더 (Leak-Reduction Headers)

인라인 스크립트나 향후 동일 출처 벤더링 라이브러리를 차단하지 않는 보수적인 헤더를 추가합니다:

```text
Referrer-Policy: no-referrer
Cache-Control: no-store
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Cross-Origin-Resource-Policy: same-origin
```

이번 패스에서 제한적인 `script-src` CSP를 추가하지 마십시오. Companion은 현재 인라인 헬퍼 JavaScript를 주입하며 향후 스크린이 동일 출처 벤더링 라이브러리를 로드할 수 있습니다.

### 6. 지속성 세션 상태 Gitignore

저장소 루트 `.gitignore`에 `.superpowers/`를 추가하여 `--project-dir`을 사용할 때 영속화된 companion 상태 및 `.last-token`이 실수로 커밋되지 않도록 합니다.

### 7. 테스트 안정성 및 린트

수정된 시작/중지 스크립트의 shell lint 경고를 정리합니다.

`start-server.sh --idle-timeout-minutes`를 호출하는 라이프사이클 테스트를 업데이트하여 Codex의 `CODEX_CI` 포그라운드 자동 감지 하에서 중단되지 않도록 합니다. 스크립트가 시작 JSON을 반환할 것으로 기대할 때 테스트는 `--background`로 백그라운드 모드를 강제해야 합니다.

## 테스트 전략

모든 동작 변경은 TDD로 진행되어야 합니다:

1. 실패하는 집중 테스트 작성
2. 실행하여 예상된 이유로 실패하는지 확인
3. 최소한의 수정 구현
4. 집중 테스트 재실행
5. 전체 brainstorm-server 수트 재실행

필수 집중 회귀 테스트 항목:

- 유효한 키가 포함된 `/`는 스크린 콘텐츠가 아닌 부트스트랩을 반환함
- 부트스트랩은 `sessionStorage`에 키를 저장하고 URL을 제거함
- 쿠키 전용 `/`는 여전히 스크린 콘텐츠를 서빙함
- 헬퍼는 WebSocket URL에 `sessionStorage` 키를 사용함
- 동일 출처 쿠키 WebSocket이 열림
- 교차 출처 쿠키 WebSocket은 거부되고 이벤트를 작성하지 않음
- 직접 키 WebSocket은 `Origin` 없이도 여전히 열림
- `state/server-info`를 가리키는 `content/` 아래 심볼릭 링크는 404를 반환함
- 일반 HTML, 부트스트랩, 403, 파일 응답에 보안 헤더가 존재함
- 동일 포트/토큰 재시작 시 저장된 키로 재연결 인증 가능
- 수정된 쉘 스크립트에 대해 shell lint가 통과함
- Codex 하에서 라이프사이클 수트가 중단되지 않음

## 수탁 기준 (Acceptance Criteria)

- `cd tests/brainstorm-server && npm test`가 중단 없이 반복적으로 통과함.
- 이전에 다른 `localhost` 출처에서 `attacker-injected`를 작성했던 보안 프로브가 이제 WebSocket을 열지 못하고 `state/events`를 변경되지 않은 상태로 남겨둠.
- `server-info` 심볼릭 링크 프로브가 404를 반환함.
- 헤드형 또는 헤드리스 브라우저의 키 로드가 순수한 `/` URL로 끝나고 상태 표시 점이 Connected에 도달함.
- 동일 포트/토큰 재시작 시 수동 새로고침 없이 자동으로 재연결됨.
- 수정된 쉘 스크립트에 대해 `scripts/lint-shell.sh`가 통과함.

## 연기된 작업

향후 프로젝트에서 스크린 HTML을 신뢰할 수 없는 것으로 취급해야 하는 경우, 별도의 샌드박스 처리된 iframe 아키텍처를 설계하십시오. 해당 작업은 생성된 스크린을 별도의 출처나 샌드박스 프레임에 격리하고 사용자 선택을 위해 좁은 `postMessage` 브릿지만 노출해야 합니다. 이 수정 패스에 이를 포함시키지 마십시오.
