# Visual Brainstorming Companion — 이슈 및 변경 카탈로그

**날짜:** 2026-06-09  
**상태:** 분석 / 분류. 이를 직접 구현하며, 참조된 커뮤니티 PR은 증거 및 참고 자료일 뿐 머지할 코드가 **아닙니다**.

## 목적

visual brainstorming companion(`skills/brainstorming/scripts/`의 로컬 서버)과 관련된 모든 오픈 이슈 및 PR을 수집하고, 근본적인 문제와 변경할 사항을 요약하여 관리하는 단일 공간입니다. 각 항목은 PR 작성자의 설명이 아닌 현재 코드를 기준으로 검토됩니다.

## 범위 결정 (Jesse, 2026-06-09)

- **Alpine.js를 벤더링하지 않음.** PR #1639 (벤더링된 Alpine 빌드를 통한 대화형 모크업)는 **폐기**되었습니다. E3를 참조하십시오.
- **E1 (터미널 대 HTML 하드 게이트)은 워크숍 항목입니다.** 함께 설계할 예정이며, 여기에는 규정되지 않습니다.
- **E2 (저장 위치, #975/#977)는 연기**되었습니다.
- **원격 서빙은 최우선 시나리오입니다.** Superpowers는 범용 도구입니다; 사용자는 원격 접속(SSH 터널, Tailscale, `--host 0.0.0.0`) 환경에서 연결합니다. 보안 수정 사항은 루프백뿐만 아니라 이러한 사용자도 반드시 보호해야 합니다. **결정사항: Host 허용 목록이 아닌 세션별 비밀 키(per-session secret key)** 사용. Host 허용 목록은 루프백 브라우저 대리인 혼동(confused deputy) 공격만 방지합니다; 직접 원격 클라이언트는 예상되는 `Host`를 전송하므로 허용 목록은 원격 노출에 대해 무의미합니다. 비밀 키만이 루프백, 터널, 직접 원격 전반에서 클라이언트를 일관되게 인증하고 DNS 리바인딩도 차단합니다. A1을 참조하십시오.

## 컴포넌트 맵

| File | Role |
|------|------|
| `skills/brainstorming/scripts/server.cjs` | 의존성 없는 HTTP + WebSocket 서버 (RFC 6455 직접 구현). 최신 스크린 서빙, `content/` 감시, `state/events`에 이벤트 기록. |
| `skills/brainstorming/scripts/helper.js` | 모든 페이지에 주입됨. WebSocket 클라이언트, 클릭 캡처, `window.brainstorm` API. |
| `skills/brainstorming/scripts/frame-template.html` | 콘텐츠 조각을 감싸는 프레임 (헤더, 테마 CSS, 상태 점, 표시 줄). |
| `skills/brainstorming/scripts/start-server.sh` | 실행 래퍼. 세션 디렉토리, host/url-host, 소유자 PID 확인, 플랫폼 백그라운드 처리. |
| `skills/brainstorming/scripts/stop-server.sh` | PID 파일로 서버를 종료하고 `/tmp` 세션을 정리. |
| `skills/brainstorming/visual-companion.md` | 에이전트가 companion을 수락할 때 읽는 운영자 가이드. |
| `skills/brainstorming/SKILL.md` | companion이 제안되고 질문별 결정이 이루어지는 위치. |

## 조치 요약

| ID | Item | Source | Disposition |
|----|------|--------|-------------|
| A1 | `/`, `/files/*`, WS에 세션별 비밀 키 적용 (Host 허용 목록 대체) | issues #1014, PRs #1110/#1553 | **Do** — 채택된 접근 방식 |
| A2 | Host 허용 목록; 브라우저 WS Origin 검사 | PRs #1110/#1553 | Host 허용 목록은 폐기됨; 브라우저 대리인 혼동 방지를 위해 인증 후 WS Origin 검사는 유지 |
| A3 | `null` / 비객체 WS 페이로드 시 크래시 | PR #1504 | Do |
| A4 | `decodeFrame` 내 프레임 길이 제한 | issue #1446 | 이미 수정됨 — 확인 후 닫기 |
| B1 | 도트파일 스크린이 콘텐츠로 서빙됨 (`._*.html`) | PR #950 | Do |
| B2 | `stop-server.sh`가 재사용된/오래된 PID를 종료함 | PR #1703 | Do |
| B3 | WS 클라이언트 재연결 백오프 + 상태 표시기 | PR #856 | Do |
| C1 | 유휴 타임아웃이 너무 짧거나 설정 불가; 종료 시 WS가 닫히지 않음 | issue #1237 (PR #1689) | Do |
| C2 | 서버 사망을 사용자/에이전트가 감지할 수 없음 | issue #1237 (잔여) | Do |
| D1 | companion의 영구적인 비활성화(opt-out) | issue #892 | 연기 - PR #1720에 포함되지 않음 |
| D2 | 브라우저에서의 자유 형식 피드백 | issue #957 | 연기 - PR #1720에 포함되지 않음 |
| D3 | companion URL 자동 열기 | PR #759 (#755) | `--open`을 통해 PR #1720에서 완료됨 |
| D4 | 프레임의 라이트/다크 대비 헬퍼 | PR #1683 | 연기 - PR #1720에 포함되지 않음 |
| E1 | 질문별 터미널 대 HTML 하드 게이트 | PR #1037 | **Workshop** |
| E2 | 세션 상태를 작업 트리 외부로 이동 | issue #975 (PR #977) | **Deferred** |
| E3 | 대화형 모크업을 위한 Alpine.js 벤더링 | PR #1639 | **Dropped** |
| E4 | 시작/중지 스크립트의 Shell-lint 경고 | PR #1677 | 기회 발생시에만 진행 |

---

## A. 서버 보안 강화 (`server.cjs`)

### A1 — 세션별 비밀 키 (채택된 접근 방식)

**위협 모델.** 두 가지 자산: 서빙되는 스크린 (`/`) 및 파일 (`/files/*`)의 기밀성과 `state/events`의 무결성 — 참인 `choice`를 가진 WebSocket 클라이언트가 해당 위치에 기록하며(`server.cjs:243-246`), 에이전트는 이를 다음 턴에 사용자의 선택으로 읽어옵니다. 즉, **전체 도구 접근 권한을 가진 라이브 세션에 대한 프롬프트 주입** 공격이 가능합니다. 접근자: 기본 `127.0.0.1` 바인딩의 경우 사용자의 브라우저 내 악성 페이지(대리인 혼동 — 공격자 JS를 실행하면서 루프백에 접근 가능); 원격 바인딩(`--host 0.0.0.0`, tailnet/LAN)의 경우 포트로 라우팅할 수 있는 모든 호스트가 동일 출처 정책 제약 없이 직접 접근 가능합니다. 현재 `handleUpgrade` (`server.cjs:176`)는 `Sec-WebSocket-Key`만 검사하고, `handleRequest` (`server.cjs:138`)는 아무것도 검사하지 않으므로 둘 다 완전히 열려 있습니다.

**Host 허용 목록이 아닌 키를 사용하는 이유.** Host 허용 목록은 루프백 브라우저 대리인 공격만 보호합니다. 직접 원격 클라이언트는 예상되는 `Host`를 전송하고 `Origin`을 위조/누락하므로, 허용 목록은 보호해야 할 원격 사례에 대해 단순 전시에 불과합니다. 세션별 비밀 키는 루프백, SSH 터널, 직접 원격 전반에서 클라이언트를 일관되게 인증하며, DNS 리바인딩도 막아냅니다(리바인딩된 페이지는 키를 알지 못하며 호스트 범위 쿠키도 받지 못함). 따라서 키는 A1/A2의 Host 허용 목록을 완전히 **대체**하며, `BRAINSTORM_ALLOWED_HOSTS`는 필요하지 않습니다.

**설계.** 랜덤 토큰 (`crypto.randomBytes(32)` 헥사)을 시작 시 `server.cjs`에서 생성 (확정적 테스트를 위해 `BRAINSTORM_TOKEN`으로 재정의 가능):

1. **URL에 포함:** `?key=<token>` 형태. 서버는 이미 `server-started` JSON (`server.cjs:351`)에서 `url`을 생성하고 `state/server-info`에 작성하므로, 여기에 `?key=`를 추가하면 `start-server.sh` (JSON을 grep하여 출력) 및 스킬 (사용자에게 URL 전달)을 **수정할 필요가 없습니다**.
2. **쿠키 부트스트랩:** `/`의 유효한 `?key`는 `brainstorm-key-<port>=<token>; HttpOnly; SameSite=Strict; Path=/`를 설정합니다. 그 후 브라우저는 동일 출처 서브리소스(`/files/*`) 및 WebSocket 핸드셰이크에 이를 자동으로 첨부하므로, 에이전트가 어떤 URL 스타일을 작성하든 동작하며 `helper.js`를 수정할 필요가 없습니다. 쿠키 이름은 Jupyter 다중 서버 충돌을 방지하기 위해 **포트별**로 지정됩니다(쿠키는 포트 범위가 아님). `SameSite=Strict`는 CDN/Unsplash 콘텐츠에 안전합니다 — 해당 쿠키는 호스트 범위이므로 아웃바운드 CDN 요청에는 전달되지 않으며, SameSite는 우리 출처로의 요청만 제어합니다.
3. **인증 게이트** = `/`, `/files/*`, WS 업그레이드 시 유효한 `?key` **또는** 유효한 쿠키 (`crypto.timingSafeEqual`로 비교). 누락되거나 잘못된 키 → 친절한 **403 HTML 페이지** ("이 페이지에는 코딩 에이전트가 제공한 `?key=…`를 포함한 전체 URL이 필요합니다" — Codex/Gemini/Copilot에서도 제공되므로 "Claude"가 아닌 일반적인 "코딩 에이전트" 표기). WS 업그레이드 → 소켓 파괴.

쿼리 토큰이 신뢰할 수 있는 단일 소스(source of truth)이며, 쿠키는 초기 인증 부담을 지지 않는 편의 기능입니다.

**영향 범위.** `server.cjs` (모든 로직). `helper.js` 선택적 한 줄 (쿠키가 차단된 경우를 대비해 `location.search`에서 `?key=`를 WS URL에 추가). `start-server.sh` 변경 없음. `visual-companion.md` 문서 참고 (URL에 이제 `?key=`가 포함되므로 제거하지 말 것). 토큰을 전달하도록 테스트 업데이트.

### A2 — Host 허용 목록 폐기; 브라우저 WS Origin 유지

A1에 통합됨. 비밀 키는 하나의 메커니즘으로 WS 주입 벡터(#1014), HTTP/WS DNS 리바인딩 읽기 벡터(PR #1553), 교차 출처 WS 벡터(PR #1110)를 모두 차단하며, 허용 목록과 달리 원격 바인딩 사례도 실제로 보호합니다. `BRAINSTORM_ALLOWED_HOSTS`와 Host 허용 목록은 제공되지 않습니다. 최종 구현에서는 세션 인증 후 브라우저 WebSocket `Origin`을 여전히 검사하여 교차 출처 localhost 탭이 companion 쿠키를 도용할 수 없도록 합니다.

### A3 — `null` / 기본형 WS 페이로드 시 서버 크래시

**문제.** `handleMessage` (`server.cjs:233`)는 `JSON.parse(text)` 후 `server.cjs:243`에서 `if (event.choice)`를 수행합니다. 4바이트 텍스트 프레임 `null`을 보낸 클라이언트는 `event === null`을 생성하며, `null.choice`는 예외를 발생시킵니다. 이 예외는 **포착되지 않으며** — `handleMessage`는 `decodeFrame`만 감싸는 `try/catch` 외부의 `socket.on('data')` 핸들러 (`server.cjs:207`)에서 호출됩니다. 결과적으로 포착되지 않은 예외와 프로세스 종료가 발생합니다. 로컬 클라이언트라면 누구든 서버를 종료할 수 있습니다.

**변경사항.** 접근 보호: `if (event && event.choice)`. 최소한이며 정확함 — `JSON.parse`는 `undefined`를 생성하지 않으며, 기본형(primitive)은 예외 발생 없이 `.choice`에 대해 `undefined`를 반환하므로 `null`만 실제 위협입니다. (최상위 `try/catch`나 `process.on('uncaughtException')` 같은 광범위한 수정은 다른 버그를 가릴 수 있으므로 피합니다.)

### A4 — `decodeFrame` 내 프레임 길이 제한 (인접 항목)

PR #1504에서 #1446으로 참조됨. 현재 코드는 확장 프레임 길이를 **이미** 제한하고 있습니다: `MAX_FRAME_PAYLOAD_BYTES = 10MB` (`server.cjs:10`)가 `Buffer.alloc` 이전인 `server.cjs:58-67`에서 강제됩니다. 조치사항: 재구현하지 않고 현재 `dev`에서 #1446을 확인 후 해결되었으면 닫음.

---

## B. 서버 안정성 / 정확성

### B1 — macOS 리소스 포크 도트파일이 스크린 콘텐츠로 서빙됨

**문제.** 최신 스크린 선택기는 `f.endsWith('.html')`만 필터링합니다 (`server.cjs:127-128`). macOS/ExFAT 환경에서 `._screen.html` 리소스 포크 파일이 해당 필터를 통과하고 실제 파일과 함께 생성되어 최신 파일로 정렬될 수 있습니다 — 이로 인해 브라우저가 모크업 대신 바이너리 메타데이터를 받게 됩니다. 4개 읽기 지점이 이 취약한 필터를 공유합니다: `getNewestScreen` (`server.cjs:127`), `knownFiles` 초기화 (`server.cjs:279`), `fs.watch` 핸들러 (`server.cjs:286`), `/files/` 엔드포인트 (`server.cjs:154-156`).

**변경사항.** 4개 지점 모두에서 도트파일(`!f.startsWith('.')`)을 거부. `._*`, `.DS_Store` 등을 처리함.

### B2 — `stop-server.sh`가 재사용된 PID를 종료할 수 있음

**문제.** `stop-server.sh`는 `state/server.pid` (`stop-server.sh:20`)에서 PID를 읽어와 해당 PID가 당사 서버에 속하는지 확인하지 않고 `kill` (`:23`, `:35`에서 `-9`로 에스컬레이션)을 실행합니다. 재부팅이나 PID 재사용 후 파일이 무관한 프로세스를 가리킬 수 있으며, 이 경우 해당 프로세스에 SIGKILL을 보내게 됩니다.

**변경사항.** 시그널을 보내기 전에 소유권을 검증 — PID의 명령어가 당사의 `server.cjs`를 실행하는 `node`인지, 가급적 이 세션과 일치하는지 확인합니다. 소유권을 증명할 수 없으면 실패로 처리합니다 (`stale_pid` 보고, kill 하지 않음). 실제 케이스에 대한 기존 `stopped` / `not_running` 출력은 유지합니다.

### B3 — WebSocket 클라이언트: 무음 재연결, 오래된 "Connected" 상태

**문제.** `helper.js`는 고정된 1초 타이머로 재연결을 시도하며 (`helper.js:21-23`), `onerror` 핸들러가 없고, 닫힐 때 `ws`를 null로 설정하지 않으며, 보류 중인 재연결 타이머를 지우지 않습니다. 프레임의 상태 요소는 "Connected"로 하드코딩되어 있고 점은 `var(--success)`로 고정되어 있습니다 (`frame-template.html:77,200`). 노트북이 절전 모드에 들어가거나 서버가 재시작되면, 페이지는 죽은 소켓 위에서 "Connected"를 표시하며 아무런 피드백 없이 이벤트를 큐에 쌓습니다.

**변경사항.**
- `helper.js`: 지수 백오프 (500ms → ×2 → 최대 30s, open 시 리셋); `onclose`로 위임하는 `onerror`; close 시 `ws = null`; 재연결 전 `clearTimeout`.
- `frame-template.html`: JS가 Connected (초록) / Reconnecting (노랑) / Disconnected (빨강)을 전환할 수 있도록 `--status-color` 사용자 정의 속성으로 상태 점을 제어.

---

## C. 라이프사이클 / 타임아웃 (issue #1237)

### C1 — 유휴 타임아웃이 너무 짧고 설정 불가능하며, WS가 프로세스를 살려둠

**문제.** `IDLE_TIMEOUT_MS`가 30분으로 하드코딩되어 있으며 (`server.cjs:258`), 60초 라이프사이클 검사로 강제됩니다 (`server.cjs:329-332`). 사용자가 생각하거나 자리를 비우는 동안 단일 브레인스토밍 질문이 30분 이상 대기할 수 있어 세션 중간에 서버가 종료될 수 있습니다. 별개로, `shutdown()` (`server.cjs:310-321`)은 `server.close()`를 호출하지만 `clients` (`server.cjs:174`)의 업그레이드된 소켓을 닫지 않으므로, 열린 브라우저 연결이 종료 시간 이후에도 Node 프로세스를 계속 살려둘 수 있습니다.

**변경사항.**
- 기본값을 4시간으로 늘리고 설정 가능하게 변경: `start-server.sh`에서 `--idle-timeout-minutes` → 환경 변수 → `IDLE_TIMEOUT_MS`, Node 타이머 오버플로 검증 포함.
- 시작 JSON / `state/server-info`에 유효 타임아웃 노출.
- `shutdown()`에서 `clients` 내의 모든 소켓을 닫아 프로세스가 실제로 종료되도록 함.

### C2 — 서버 사망이 감지되지 않음

**문제.** 서버가 종료될 때 `state/server-stopped`를 기록하고 `state/server-info`를 제거하며 (`server.cjs:312-317`), 스킬에는 해당 파일들을 검사하도록 *안내*되어 있습니다 (`visual-companion.md:108`) — 하지만 이는 모델이 건너뛰는 소프트 가이드일 뿐이며 브라우저는 일반적인 "연결할 수 없음"만 표시합니다. 사용자가 수동으로 진단해야 하며 에이전트는 죽은 URL을 계속 참조합니다.

**변경사항 (C1과 독립적인 두 부분):**
- **브라우저 대상 툼스톤(tombstone).** 연결 오류 대신 "이 companion이 만료되었습니다 — Claude에게 재시작을 요청하세요"라고 표시되는 안내를 마지막 서빙 URL에 남겨둡니다. 검토할 옵션: 백오프 시간 이후에도 소켓이 닫혀 있을 때 `helper.js`가 배너를 렌더링하는 방법 (페이지가 로드되어 있는 동안만 동작함) vs 툼스톤 페이지를 서빙하는 최소 응답기를 살려두는 보다 복잡한 접근 방식.
- **엄격해진 스킬 검사.** `visual-companion.md` / `SKILL.md`를 강화하여 "URL을 참조하거나 스크린을 푸시하기 전에 `server-info`/`server-stopped` 검사"를 단순 참고사항이 아닌 필수 단계로 만듭니다. 에이전트가 항상 실행하는 한 줄 헬퍼 형태로 가볍게 유지합니다.

---

## D. 기능

### D1 — visual companion의 영구 비활성화 (issue #892)

**문제.** companion은 매 세션마다 별도의 메시지로 제안됩니다 (`SKILL.md:25,151-152`). 이를 전혀 원하지 않는 사용자는 매번 왕복 시간과 HTML 생성 비용을 지불해야 합니다. "다시는 제안하지 마라"고 설정할 방법이 없습니다.

**변경사항.** 제안 단계 전 스킬이 사용자 수준 설정을 확인하고, 비활성화(opt-out)가 설정되어 있으면 제안을 완전히 건너뜁니다.

**설계 선택 사항.** 메커니즘이 결정되지 않음:
- 스킬이 읽도록 지시받는 환경 변수 (예: `SUPERPOWERS_VISUAL_COMPANION=off`) — 가장 단순하고 이슈 요청과 일치하며 `.zshrc`에 위치함.
- 플러그인 설정 파일 (`.claude/superpowers.local.md` 프론트매터) — 더 구조화되어 있고 프로젝트별 설정이 가능하지만 더 무겁고 프로젝트 범주에 국한됨.
- 이슈에서의 신뢰성 주의사항: 별도의 "no-companion" 스킬은 트리거 단어에서 경합하며 신뢰할 수 없음 — 거부됨.

메커니즘을 선택하면 소규모 `SKILL.md` 변경 및 문서화된 노브가 추가됩니다.

### D2 — 브라우저에서의 자유 형식 피드백 (issue #957)

**문제.** 클라이언트는 `[data-choice]` 클릭만 캡처합니다 (`helper.js:36-62`). 모크업에 주석을 달고 싶은 사용자("파란색 음영이 잘못됨")는 터미널로 전환해야 하므로 시각적 흐름이 끊깁니다.

**변경사항.** 제출 시 기존 `window.brainstorm.send` 경로 (`helper.js:82-85`)를 통해 `{"type":"feedback","text":...,"timestamp":...}`를 전송하는 피드백 `<textarea>`를 추가합니다.

**교차 영역 — 서버 변경 필요.** `handleMessage`는 `event.choice`가 참일 때만 이벤트를 저장합니다 (`server.cjs:243`). `feedback` 이벤트에는 `choice`가 없으므로 지금은 로깅만 되고 **`state/events`에 작성되지 않으며**, 에이전트가 이를 볼 수 없습니다. 지속성 조건이 `feedback` 이벤트도 허용해야 합니다. `visual-companion.md` (Browser Events Format, `:247-259`)에 새 이벤트 형식을 문서화합니다. 제출 트리거(버튼 대 blur 대 둘 다)와 textarea가 렌더링될 위치(프레임 레벨 대 스크린별 선택 적용)를 결정합니다.

### D3 — companion URL 자동 열기 (PR #759, issue #755)

**문제.** `start-server.sh`는 URL을 출력하기만 하며 사용자가 수동으로 엽니다. 특히 WSL2에서는 브라우저가 자동으로 열리기를 기대합니다.

**변경사항.** `server-started` JSON이 파싱된 후 최선형(best-effort) 오프너 실행: Windows/WSL → `rundll32.exe url.dll,FileProtocolHandler <url>`, macOS → `open`, Linux → `DISPLAY`/`WAYLAND_DISPLAY`가 설정된 경우에만 `xdg-open`. 실패는 무시하고 시작을 차단하지 않으며 URL 출력을 유지합니다. `visual-companion.md`에 문서화합니다. (브라우저를 띄우는 것이 부적절한 헤드리스/원격 실행에 대한 비활성화 옵션 고려 — D1의 설정 메커니즘과 연계됨.)

### D4 — 라이트/다크 대비 헬퍼 (PR #1683)

**문제.** 콘텐츠 조각이 OS 감지 프레임(`frame-template.html`)에 감싸여 있습니다. 다크 모드에서 빠른 모크업은 종종 흰색 인라인 배경을 사용하면서 대비가 낮은 프레임 텍스트를 상속받아 카드가 읽기 어려워집니다.

**변경사항.** `.light-surface` / `.dark-surface` 헬퍼 클래스를 추가하고 일반적인 인라인 밝은 배경에 대한 보수적인 폴백을 추가하며, `visual-companion.md`의 CSS 참조 문서에 기록합니다. `frame-template.html` 내의 순수 CSS입니다.

---

## E. 워크숍 / 연기 / 폐기

### E1 — 질문별 터미널 대 HTML 하드 게이트 (PR #1037) — WORKSHOP

소프트 가이드라인은 이미 존재합니다: "질문별로 결정하되", `SKILL.md:156-161` 및 `visual-companion.md:5-25`에 브라우저 대 터미널 테스트가 포함되어 있습니다. 불만 사항은 모델이 텍스트성 콘텐츠(A/B 목록, 확인 질문)에 대해 HTML을 렌더링하여 토큰과 턴을 낭비한다는 점입니다. PR #1037은 결정을 `<HARD-GATE>`로 감쌉니다. **Jesse의 의견에 따라 문구/메커니즘을 함께 워크숍할 예정입니다** — 이는 동작을 결정하는 스킬 내용이며 여기에 규정되지 않습니다.

### E2 — 세션 상태를 작업 트리 외부로 이동 (issue #975 / PR #977) — DEFERRED

현재 `--project-dir`은 세션 상태를 `<project>/.superpowers/brainstorm/` (`start-server.sh:80-84`)에 작성하고 스킬은 사용자에게 이를 gitignore 하도록 안내합니다 (`visual-companion.md:58`). 요청 사항은 저장소 외부(XDG)에 기본값으로 `--state-dir` / `SUPERPOWERS_STATE_DIR`을 두고 `--project-dir`을 별칭으로 유지하는 것입니다. **Jesse에 의해 당분간 연기됨.** 유실되지 않도록 수집됨.

### E3 — 대화형 모크업을 위한 Alpine.js 벤더링 (PR #1639) — DROPPED

모크업이 수동 JS 작성 없이 대화형(탭, 아코디언, 폼)으로 작동할 수 있도록 벤더링된 Alpine 빌드를 추가합니다. **Jesse의 의견에 따라 폐기됨** — companion 런타임에 외부 서드파티 의존성을 포함시키지 않습니다. 대화형 모크업이라는 근본적 요구사항은 이 방식으로 추진되지 않습니다.

### E4 — Shell-lint 경고 (PR #1677) — OPPORTUNISTIC

`start-server.sh` / `stop-server.sh` 내의 SC2034 등. 사소한 내용임; 별도 변경이 아닌 해당 스크립트를 편집할 때 B2/C1/D3에 통합합니다.

---

## 구현을 위한 권장 그룹화

이들은 몇 가지 일관된 패스로 그룹화됩니다 (각 패스는 `tests/brainstorm-server/`에 대해 독립적으로 테스트 가능):

1. **보안 패스** (진행 중, 브랜치 `brainstorm-companion-session-key`) — A1 세션별 키 (A2 대체) + A3 null-크래시 가드. A4 확인/닫기. *최우선 순위.*
2. **라이프사이클 패스** — C1 + C2 함께 진행 (둘 다 `shutdown()` 및 서버 사망 관련).
3. **안정성 패스** — B1, B2, B3 (독립적, 소규모).
4. **연기된 기능 패스** - D1, D2, D4는 PR #1720의 일부가 아닙니다. D3는 `--open` 흐름을 통해 제공됩니다.

E1은 별도의 워크숍 세션입니다. E2/E3는 이번 라운드 범위를 벗어납니다.
