# Visual Brainstorming 리팩토링: 브라우저 디스플레이, 터미널 명령어

**날짜:** 2026-02-19  
**상태:** 승인됨 (Approved)  
**범위:** `lib/brainstorm-server/`, `skills/brainstorming/visual-companion.md`, `tests/brainstorm-server/`

## 문제점

시각적 브레인스토밍 동안, Claude는 `wait-for-feedback.sh`를 백그라운드 작업으로 실행하고 `TaskOutput(block=true, timeout=600s)`에서 블로킹됩니다. 이는 TUI를 완전히 장악합니다 — 시각적 브레인스토밍이 실행 중일 때 사용자는 Claude에게 터미널로 타이핑할 수 없습니다. 브라우저가 유일한 입력 채널이 됩니다.

Claude Code의 실행 모델은 턴 기반(turn-based)입니다. 단일 턴 내에서 Claude가 두 개 채널을 동시에 수신 대기할 방법은 없습니다. 블로킹 방식의 `TaskOutput` 패턴은 플랫폼이 지원하지 않는 이벤트 기반 동작을 시뮬레이션한 잘못된 프리미티브였습니다.

## 설계

### 핵심 모델

**브라우저 = 대화형 디스플레이.** 모크업을 보여주고, 사용자가 클릭하여 옵션을 선택하도록 합니다. 선택 항목은 서버 측에 기록됩니다.

**터미널 = 대화 채널.** 항상 블로킹되지 않으며 항상 이용 가능합니다. 사용자는 여기서 Claude와 대화합니다.

### 루프 (The Loop)

1. Claude가 세션 디렉토리에 HTML 파일을 작성함
2. 서버가 chokidar를 통해 이를 감지하고, 브라우저로 WebSocket 리로드를 푸시함 (변경 없음)
3. Claude가 턴을 종료함 — 사용자에게 브라우저를 확인하고 터미널에서 응답하라고 알림
4. 사용자가 브라우저를 보고, 옵션을 클릭하여 선택(선택 사항)한 다음 터미널에 피드백을 타이핑함
5. 다음 턴에서 Claude가 브라우저 상호작용 스트림(클릭, 선택 항목)을 읽기 위해 `$SCREEN_DIR/.events`를 읽고 터미널 텍스트와 병합함
6. 반복하거나 다음 단계로 진행함

백그라운드 작업 없음. `TaskOutput` 블로킹 없음. 폴링 스크립트 없음.

### 핵심 삭제 항목: `wait-for-feedback.sh`

완전히 삭제됩니다. 이 스크립트의 목적은 "서버가 stdout에 이벤트를 로깅함"과 "Claude가 해당 이벤트를 수신해야 함"을 연결하는 것이었습니다. `.events` 파일이 이를 대체합니다 — 서버가 사용자 상호작용 이벤트를 직접 작성하고, Claude는 플랫폼이 제공하는 파일 읽기 메커니즘을 통해 이를 읽습니다.

### 핵심 추가 항목: `.events` 파일 (스크린별 이벤트 스트림)

서버는 라인당 하나의 JSON 객체로 모든 사용자 상호작용 이벤트를 `$SCREEN_DIR/.events`에 작성합니다. 이를 통해 Claude는 현재 스크린에 대한 전체 상호작용 스트림을 확보할 수 있습니다 — 단순한 최종 선택 항목뿐만 아니라 사용자의 탐색 경로(A를 클릭하고, B를 클릭한 후, C로 확정함)까지 파악 가능합니다.

사용자가 옵션을 탐색한 후의 예시 내용:

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 스크린 내에서 추가 전용(Append-only). 각 사용자 이벤트는 새로운 줄로 추가됩니다.
- chokidar가 새 HTML 파일(새 스크린 푸시)을 감지하면 이 파일은 정리(삭제)되어, 오래된 이벤트가 이월되는 것을 방지합니다.
- Claude가 읽을 때 파일이 존재하지 않는 경우, 브라우저 상호작용이 일어나지 않은 것이므로 Claude는 터미널 텍스트만 사용합니다.
- 파일에는 사용자 이벤트(`click` 등)만 포함되며, 서버 라이프사이클 이벤트(`server-started`, `screen-added`)는 포함되지 않습니다. 이를 통해 작고 집중된 상태를 유지합니다.
- Claude는 전체 스트림을 읽어 사용자의 탐색 패턴을 이해하거나, 최종 선택 항목에 대해 마지막 `choice` 이벤트만 살펴볼 수 있습니다.

## 파일별 변경 사항

### `index.js` (서버)

**A. 사용자 이벤트를 `.events` 파일에 작성.**

WebSocket `message` 핸들러에서 이벤트를 stdout에 로깅한 후: `fs.appendFileSync`를 통해 이벤트를 JSON 라인으로 `$SCREEN_DIR/.events`에 추가합니다. 서버 라이프사이클 이벤트가 아닌 사용자 상호작용 이벤트(`source: 'user-event'`를 가진 이벤트)만 작성합니다.

**B. 새 스크린 발생 시 `.events` 정리.**

chokidar `add` 핸들러(새 `.html` 파일 감지)에서 `$SCREEN_DIR/.events` 파일이 존재하는 경우 삭제합니다. 이는 매 리로드마다 실행되는 GET `/`에서 정리하는 것보다 훨씬 확실한 "새 스크린" 신호입니다.

**C. `wrapInFrame` 콘텐츠 주입 교체.**

현재 정규식은 제거될 예정인 `<div class="feedback-footer">`에 앵커를 두고 있습니다. 주석 자리표시자로 교체합니다: `#claude-content` 내부의 기존 기본 콘텐츠(`<h2>Visual Brainstorming</h2>` 및 부제목 단락)를 제거하고 단일 `<!-- CONTENT -->` 마커로 교체합니다. 콘텐츠 주입은 `frameTemplate.replace('<!-- CONTENT -->', content)`가 됩니다. 더 단순하며 템플릿 포맷이 변경되어도 깨지지 않습니다.

### `frame-template.html` (UI 프레임)

**제거:**
- `feedback-footer` div (textarea, Send 버튼, label, `.feedback-row`)
- 관련 CSS (`.feedback-footer`, `.feedback-footer label`, `.feedback-row`, 내부의 textarea 및 버튼 스타일)

**추가:**
- 기본 텍스트를 교체하는 `#claude-content` 내부의 `<!-- CONTENT -->` 자리표시자
- 푸터가 있던 위치에 두 가지 상태를 가진 선택 표시 바 추가:
  - 기본값: "Click an option above, then return to the terminal"
  - 선택 후: "Option B selected — return to terminal to continue"
- 표시 바를 위한 CSS (은은하게, 기존 헤더와 유사한 시각적 비중)

**변경 없이 유지:**
- "Brainstorm Companion" 제목과 연결 상태가 있는 헤더 바
- `.main` 래퍼 및 `#claude-content` 컨테이너
- 모든 컴포넌트 CSS (`.options`, `.cards`, `.mockup`, `.split`, `.pros-cons`, 자리표시자, 모크 요소들)
- 다크/라이트 테마 변수 및 미디어 쿼리

### `helper.js` (클라이언트 측 스크립트)

**제거:**
- `sendToClaude()` 함수 및 "Sent to Claude" 페이지 장악 로직
- `window.send()` 함수 (제거된 Send 버튼과 연결되어 있었음)
- 폼 제출 핸들러 — 피드백 textarea 없이 의미 없음, 로그 노이즈 추가
- 입력 변경 핸들러 — 동일한 이유
- `pageshow` 이벤트 리스너 (textarea 지속성을 수정하기 위해 추가되었음 — 더 이상 textarea 없음)

**유지:**
- WebSocket 연결, 재연결 로직, 이벤트 큐
- 리로드 핸들러 (서버 푸시 시 `window.location.reload()`)
- 선택 강조를 위한 `window.toggleSelect()`
- `window.selectedChoice` 추적
- `window.brainstorm.send()` 및 `window.brainstorm.choice()` — 이들은 제거된 `window.send()`와는 다릅니다. 이들은 WebSocket을 통해 서버에 로깅하는 `sendEvent`를 호출합니다. 커스텀 전체 문서 페이지에 유용합니다.

**축소:**
- 클릭 핸들러: 모든 버튼/링크가 아닌 `[data-choice]` 클릭만 캡처. 광범위한 캡처는 브라우저가 피드백 채널이었을 때 필요했으나, 이제는 선택 추적만을 위함입니다.

**추가:**
- `data-choice` 클릭 시 선택 표시 바 텍스트를 업데이트하여 어떤 옵션이 선택되었는지 보여줌.

**`window.brainstorm` API에서 제거:**
- `brainstorm.sendToClaude` — 더 이상 존재하지 않음

### `visual-companion.md` (스킬 지침)

**"The Loop" 섹션을 위에서 설명한 비블로킹 흐름으로 재작성.** 다음 항목에 대한 모든 참조 제거:
- `wait-for-feedback.sh`
- `TaskOutput` 블로킹
- 타임아웃/재시도 로직 (600초 타임아웃, 30분 상한)
- `send-to-claude` JSON을 설명하는 "User Feedback Format" 섹션

**다음 내용으로 교체:**
- 새로운 루프 (HTML 작성 → 턴 종료 → 사용자가 터미널에서 응답 → `.events` 읽기 → 반복)
- `.events` 파일 포맷 문서화
- 터미널 메시지가 주요 피드백이며, `.events`는 추가 컨텍스트를 위해 전체 브라우저 상호작용 스트림을 제공한다는 가이드라인

**유지:**
- 서버 시작/종료 지침
- 콘텐츠 조각 대 전체 문서 가이드라인
- CSS 클래스 참조 및 이용 가능한 컴포넌트
- 설계 팁 (질문에 맞게 정확도 조절, 스크린당 2-4개 옵션 등)

### `wait-for-feedback.sh`

**완전히 삭제됨.**

### `tests/brainstorm-server/server.test.js`

업데이트가 필요한 테스트:
- 조각 응답에서 `feedback-footer` 존재를 어설션하는 테스트 — 선택 표시 바 또는 `<!-- CONTENT -->` 교체를 어설션하도록 업데이트
- `helper.js`가 `send`를 포함함을 어설션하는 테스트 — 축소된 API를 반영하도록 업데이트
- `sendToClaude` CSS 변수 사용을 어설션하는 테스트 — 제거 (함수가 더 이상 존재하지 않음)

## 플랫폼 호환성

서버 코드(`index.js`, `helper.js`, `frame-template.html`)는 순수 Node.js 및 브라우저 JavaScript로 구성되어 플랫폼 독립적입니다. Claude Code 전용 참조가 없습니다. 백그라운드 터미널 상호작용을 통해 Codex에서 작동함이 이미 입증되었습니다.

스킬 지침(`visual-companion.md`)은 플랫폼 적응 레이어입니다. 각 플랫폼의 Claude는 자체 도구를 사용하여 서버를 시작하고, `.events`를 읽는 등의 작업을 수행합니다. 비블로킹 모델은 특정 플랫폼에 종속된 블로킹 프리미티브에 의존하지 않으므로 플랫폼 전반에서 자연스럽게 작동합니다.

## 이것이 가능하게 하는 것

- 시각적 브레인스토밍 동안 **TUI가 항상 반응함**
- **혼합 입력** — 브라우저에서 클릭 + 터미널에서 타이핑이 자연스럽게 병합됨
- **우아한 성능 저하 (Graceful degradation)** — 브라우저가 죽거나 사용자가 열지 않는가? 터미널은 여전히 작동함
- **더 단순한 아키텍처** — 백그라운드 작업 없음, 폴링 스크립트 없음, 타임아웃 관리 없음
- **크로스 플랫폼** — 동일한 서버 코드가 Claude Code, Codex 및 향후의 모든 플랫폼에서 작동함

## 이것이 포기하는 것

- **순수 브라우저 피드백 워크플로우** — 사용자는 계속 진행하기 위해 터미널로 돌아와야 합니다. 선택 표시 바가 사용자를 안내하지만, 이전의 클릭-Send-후-대기 흐름에 비해 한 단계가 추가됩니다.
- **브라우저에서의 인라인 텍스트 피드백** — textarea가 사라졌습니다. 모든 텍스트 피드백은 터미널을 통합니다. 이는 의도된 것입니다 — 터미널은 프레임 내의 작은 textarea보다 더 나은 텍스트 입력 채널입니다.
- **브라우저 Send 시 즉각적인 응답** — 이전 시스템에서는 사용자가 Send를 클릭하는 즉시 Claude가 응답했습니다. 이제 사용자가 터미널로 전환하는 동안 간격이 존재합니다. 실제로 이는 수초에 불과하며, 사용자는 터미널 메시지에 컨텍스트를 추가할 수 있습니다.
