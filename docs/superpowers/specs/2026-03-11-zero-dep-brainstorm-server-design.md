# 의존성 없는 브레인스토밍 서버 (Zero-Dependency Brainstorm Server)

brainstorm companion 서버의 벤더링된 node_modules (express, ws, chokidar — 714개 추적 파일)를 Node.js 내장 모듈만 사용하는 단일 zero-dependency `server.js`로 교체합니다.

## 동기

git 저장소에 node_modules를 벤더링하여 포함시키는 것은 공급망 리스크(supply chain risk)를 유발합니다: 고정된 의존성은 보안 패치를 받지 못하고, 714개 파일의 서드파티 코드가 감사 없이 커밋되며, 벤더링된 코드의 수정이 일반적인 커밋처럼 보입니다. 실제 리스크는 낮지만(localhost 전용 개발 서버), 이를 제거하는 것은 명확하고 간단합니다.

## 아키텍처

`http`, `crypto`, `fs`, `path`만 사용하는 단일 `server.js` 파일 (~250-300줄). 이 파일은 두 가지 역할을 수행합니다:

- **직접 실행 시** (`node server.js`): HTTP/WebSocket 서버를 시작함
- **불러올 시** (`require('./server.js')`): 단위 테스트를 위해 WebSocket 프로토콜 함수를 내보냄

### WebSocket 프로토콜

텍스트 프레임에 대해서만 RFC 6455를 구현합니다:

**핸드셰이크:** SHA-1 + RFC 6455 매직 GUID를 사용하여 클라이언트의 `Sec-WebSocket-Key`로부터 `Sec-WebSocket-Accept`를 계산. 101 Switching Protocols 반환.

**프레임 디코딩 (클라이언트 → 서버):** 세 가지 마스킹된 길이 인코딩을 처리:
- 소형: 페이로드 < 126 바이트
- 중형: 126-65535 바이트 (16-비트 확장)
- 대형: > 65535 바이트 (64-비트 확장)

4바이트 마스크 키를 사용하여 페이로드 XOR 마스크 해제. `{ opcode, payload, bytesConsumed }` 반환 또는 불완전한 버퍼의 경우 `null` 반환. 마스킹되지 않은 프레임은 거부.

**프레임 인코딩 (서버 → 클라이언트):** 동일한 세 가지 길이 인코딩을 가진 마스킹되지 않은 프레임.

**처리되는 오프코드:** TEXT (0x01), CLOSE (0x08), PING (0x09), PONG (0x0A). 인식되지 않는 오프코드는 상태 1003(Unsupported Data)을 포함한 닫기 프레임을 받음.

**의도적으로 건너뛴 항목:** 바이너리 프레임, 분할된 메시지(fragmented messages), 확장 기능(permessage-deflate), 서브프로토콜. 이들은 localhost 클라이언트 간의 작은 JSON 텍스트 메시지에는 불필요합니다. 확장 기능 및 서브프로토콜은 핸드셰이크 시 협상됩니다 — 이를 광고하지 않음으로써 절대 활성화되지 않습니다.

**버퍼 누적:** 각 연결은 버퍼를 유지합니다. `data` 이벤트 발생 시 버퍼에 추가하고 `decodeFrame`이 null을 반환하거나 버퍼가 빌 때까지 루프를 돕니다.

### HTTP 서버

세 가지 라우트:

1. **`GET /`** — mtime 기준 스크린 디렉토리에서 가장 최근의 `.html`을 서빙. 전체 문서 대 조각(fragment)을 감지하고, 조각을 프레임 템플릿으로 감싸며, helper.js를 주입. `text/html` 반환. `.html` 파일이 존재하지 않는 경우 helper.js가 주입된 하드코딩된 대기 페이지("Waiting for Claude to push a screen...")를 서빙.
2. **`GET /files/*`** — 하드코딩된 확장자 맵(html, css, js, png, jpg, gif, svg, json)의 MIME 타입 조회를 통해 스크린 디렉토리에서 정적 파일을 서빙. 발견되지 않은 경우 404 반환.
3. **그 외 나머지** — 404.

WebSocket 업그레이드는 요청 핸들러와 분리되어 HTTP 서버의 `'upgrade'` 이벤트를 통해 처리됩니다.

### 설정 (Configuration)

환경 변수 (모두 선택 사항):

- `BRAINSTORM_PORT` — 바인딩할 포트 (기본값: 랜덤 고위 포트 49152-65535)
- `BRAINSTORM_HOST` — 바인딩할 인터페이스 (기본값: `127.0.0.1`)
- `BRAINSTORM_URL_HOST` — 시작 JSON의 URL에 사용할 호스트 이름 (기본값: host가 `127.0.0.1`일 때 `localhost`, 그렇지 않으면 host와 동일)
- `BRAINSTORM_DIR` — 스크린 디렉토리 경로 (기본값: `/tmp/brainstorm`)

### 시작 시퀀스

1. `SCREEN_DIR`이 존재하지 않는 경우 생성 (`mkdirSync` 재귀 실행)
2. `__dirname`에서 프레임 템플릿 및 helper.js 로드
3. 설정된 host/port에서 HTTP 서버 시작
4. `SCREEN_DIR`에서 `fs.watch` 시작
5. 성공적으로 수신 대기 시 stdout에 `server-started` JSON 기록: `{ type, port, host, url_host, url, screen_dir }`
6. stdout이 숨겨진 경우(백그라운드 실행) 에이전트가 연결 세부 정보를 찾을 수 있도록 동일한 JSON을 `SCREEN_DIR/.server-info`에 작성

### 애플리케이션 수준 WebSocket 메시지

클라이언트로부터 TEXT 프레임이 도착할 때:

1. JSON으로 파싱. 파싱 실패 시 stderr에 로깅하고 계속 진행.
2. stdout에 `{ source: 'user-event', ...event }` 형태로 로깅.
3. 이벤트에 `choice` 속성이 포함되어 있는 경우, `SCREEN_DIR/.events`에 JSON을 추가 (이벤트당 한 줄).

### 파일 감시 (File Watching)

`fs.watch(SCREEN_DIR)`가 chokidar를 대체합니다. HTML 파일 이벤트 시:

- 새 파일 발생 시 (존재하는 파일에 대한 `rename` 이벤트): `.events` 파일이 존재하는 경우 삭제(`unlinkSync`), stdout에 `screen-added`를 JSON으로 로깅
- 파일 변경 시 (`change` event): stdout에 `screen-updated`를 JSON으로 로깅 (`.events`를 삭제하지 **않음**)
- 두 이벤트 모두: 연결된 모든 WebSocket 클라이언트에 `{ type: 'reload' }` 전송

중복 이벤트를 방지하기 위해 파일 이름별 디바운스(~100ms 타임아웃) 적용 (macOS 및 Linux에서 흔히 발생).

### 에러 처리

- WebSocket 클라이언트로부터 잘못 형성된 JSON 수신 시: stderr에 로깅, 계속 진행
- 처리되지 않은 오프코드: 상태 1003으로 닫기
- 클라이언트 연결 해제 시: 브로드캐스트 세트에서 제거
- `fs.watch` 에러: stderr에 로깅, 계속 진행
- 정상 종료(graceful shutdown) 로직 없음 — 쉘 스크립트가 SIGTERM을 통해 프로세스 라이프사이클을 관리함

## 변경 사항

| Before | After |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714개 `node_modules` 파일 | `server.js` (단일 파일) |
| express, ws, chokidar 의존성 | 없음 |
| 정적 파일 서빙 없음 | `/files/*`가 스크린 디렉토리로부터 서빙함 |

## 변경되지 않고 유지되는 사항

- `helper.js` — 변경 없음
- `frame-template.html` — 변경 없음
- `start-server.sh` — 한 줄 업데이트: `index.js`를 `server.js`로 변경
- `stop-server.sh` — 변경 없음
- `visual-companion.md` — 변경 없음
- 모든 기존 서버 동작 및 외부 계약

## 플랫폼 호환성

- `server.js`는 크로스 플랫폼 Node 내장 모듈만 사용합니다
- `fs.watch`는 macOS, Linux, Windows의 단일 평탄 디렉토리에 대해 신뢰할 수 있습니다
- 쉘 스크립트는 bash가 필요합니다 (Claude Code를 위해 요구되는 Windows의 Git Bash)

## 테스트

**단위 테스트** (`ws-protocol.test.js`): `server.js` 내보내기를 요구(require)하여 WebSocket 프레임 인코딩/디코딩, 핸드셰이크 계산, 프로토콜 엣지 케이스를 직접 테스트합니다.

**통합 테스트** (`server.test.js`): HTTP 서빙, WebSocket 통신, 파일 감시, 브레인스토밍 워크플로우 등 전체 서버 동작을 테스트합니다. 테스트 전용 클라이언트 의존성으로 `ws` npm 패키지를 사용합니다 (최종 사용자에게 배포되지 않음).
