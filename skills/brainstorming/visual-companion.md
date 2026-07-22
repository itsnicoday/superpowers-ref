# 시각적 동반자 가이드 (Visual Companion Guide)

목업, 다이어그램, 옵션을 보여주기 위한 브라우저 기반 시각적 브레인스토밍 동반자입니다.

## 사용 시점 (When to Use)

세션 단위가 아니라 **질문 단위로 결정**하세요. 테스트 기준: **사용자가 글을 읽는 것보다 눈으로 보는 것이 더 잘 이해될까?**

콘텐츠 자체가 시각적일 때 **브라우저를 사용**하세요:

- **UI 목업** — 와이어프레임, 레이아웃, 내비게이션 구조, 컴포넌트 디자인
- **아키텍처 다이어그램** — 시스템 컴포넌트, 데이터 흐름, 관계도
- **나란히 비교하는 시각적 비표 (Side-by-side visual comparisons)** — 두 개의 레이아웃, 두 개의 컬러 스킴, 두 개의 디자인 방향 비교
- **디자인 다듬기 (Design polish)** — 질문이 룩앤필, 여백, 시각적 계층 구조에 관한 것일 때
- **공간적 관계 (Spatial relationships)** — 다이어그램으로 렌더링된 상태 머신, 순서도, 엔티티 관계도

콘텐츠가 텍스트나 테이블 형태일 때 **터미널을 사용**하세요:

- **요구사항 및 범주 질문** — "X는 무엇을 의미하나요?", "어떤 기능이 범위에 포함되나요?"
- **개념적 A/B/C 선택** — 글로 설명된 접근 방식 중 선택
- **트레이드오프 목록** — 장단점, 비교 테이블
- **기술적 결정** — API 디자인, 데이터 모델링, 아키텍처 접근 방식 선택
- **명확화 질문 (Clarifying questions)** — 답변이 시각적 선호도가 아닌 글인 경우

UI 주제에 *관한* 질문이라고 해서 자동으로 시각적 질문이 되는 것은 아닙니다. "어떤 종류의 위저드를 원하십니까?"는 개념적이므로 터미널을 사용하세요. "이 위저드 레이아웃 중 어느 것이 더 적절하게 느껴지나요?"는 시각적이므로 브라우저를 사용하세요.

## 작동 방식 (How It Works)

서버는 HTML 파일이 있는 디렉토리를 감시하고 가장 최신 파일을 브라우저로 제공합니다. `screen_dir`에 HTML 콘텐츠를 작성하면 사용자는 브라우저에서 이를 확인하고 클릭하여 옵션을 선택할 수 있습니다. 선택한 항목은 다음 차례에 읽을 수 있도록 `state_dir/events`에 기록됩니다.

**콘텐츠 조각 vs 전체 문서:** HTML 파일이 `<!DOCTYPE` 또는 `<html`로 시작하면 서버는 이를 있는 그대로 제공합니다 (헬퍼 스크립트만 주입). 그렇지 않으면 서버는 헤더, CSS 테마, 연결 상태, 모든 대화형 인프라를 추가하여 포맷 템플릿으로 콘텐츠를 자동으로 감쌉니다. **기본적으로 콘텐츠 조각(content fragments)을 작성하세요.** 페이지에 대한 완전한 제어가 필요할 때만 전체 문서를 작성하세요.

## 세션 시작하기 (Starting a Session)

```bash
# 사용자가 동반자 사용을 승인한 후 시작하세요. --open은 첫 화면에서 브라우저를 자동 오픈합니다.
# --project-dir은 목업을 보존하고 동일 포트 재시작을 지원합니다.
scripts/start-server.sh --project-dir /path/to/project --open

# 반환값: {"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

응답에서 `screen_dir`과 `state_dir`을 저장하세요. `--open` 옵션을 사용하면 첫 화면을 푸시할 때 브라우저가 자동으로 열립니다. 사용자의 오픈을 따로 요청할 필요는 없지만, 폴백을 위해 URL을 계속 공유하세요 (헤드리스/원격 환경에서는 자동 오픈되지 않음).

**URL에는 세션 키(`?key=…`)가 포함되어 있습니다.** 서버는 키가 없는 요청을 거부하므로, 항상 `url` 필드의 **전체** URL을 사용자에게 제공해야 합니다 — 쿼리 스트링을 지우지 마시고, `http://host:port`만 전달하지 마세요. 키는 HTTP 및 WebSocket 접근을 차단하여 네트워크상의 다른 기기나 다른 브라우저 탭이 화면을 읽거나 이벤트를 주입하는 것을 방지합니다. 첫 로드 이후 브라우저는 쿠키를 통해 키를 기억하므로, 새로고침 및 `/files/*` 에셋 접근 시 재입력할 필요가 없습니다.

**연결 정보 찾기:** 서버는 시작 JSON을 `$STATE_DIR/server-info`에 기록합니다. 서버를 백그라운드에서 실행하고 stdout을 캡처하지 못한 경우, 해당 파일을 읽어 URL 및 포트를 확인하세요. `--project-dir`을 사용할 때는 `<project>/.superpowers/brainstorm/`에서 세션 디렉토리를 확인하세요.

**참고:** `--project-dir`에 프로젝트 루트를 전달하면 목업이 `.superpowers/brainstorm/`에 보존되어 서버가 재시작되어도 유지됩니다. 이 옵션이 없으면 파일이 `/tmp`로 들어가 삭제됩니다. `.gitignore`에 `.superpowers/`가 추가되어 있지 않다면 사용자에게 추가하도록 안내하세요.

**플랫폼별 서버 실행 방법:**

**Claude Code:**
```bash
# 기본 모드가 작동합니다 — 스크립트 자체가 서버를 백그라운드로 돌립니다.
scripts/start-server.sh --project-dir /path/to/project --open
```

Windows에서는 스크립트가 자동 감지하여 포그라운드 모드로 전환됩니다 (툴 호출을 차단함). Bash 툴 호출 시 `run_in_background: true`를 사용하여 대화 차근이 바뀌어도 서버가 유지되도록 하고, 다음 차례에 `$STATE_DIR/server-info`를 읽어 URL과 포트를 가져오세요.

**Codex:**
```bash
# Codex는 백그라운드 프로세스를 정리합니다. 스크립트가 CODEX_CI를 자동 감지하고
# 포그라운드 모드로 전환합니다. 추가 플래그 없이 정상 실행하세요.
scripts/start-server.sh --project-dir /path/to/project --open
```

**Copilot CLI:**
```bash
# --foreground를 사용하고 bash 툴의 mode: "async" 옵션으로 서버를 시작하여
# 대화 차례가 지나도 프로세스가 유지되도록 하세요. 나중에 상호작용해야 할 경우를 대비해
# 반환된 shellId를 read_bash / stop_bash용으로 캡처해 두세요.
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**기타 환경:** 서버는 대화 차례가 진행되는 동안 백그라운드에서 계속 실행되어야 합니다. 실행 환경이 분리된 프로세스를 정리하는 경우, `--foreground`를 사용하고 해당 플랫폼의 백그라운드 실행 메커니즘으로 명령을 실행하세요.

브라우저에서 URL에 접근할 수 없는 경우(원격/컨테이너 환경에서 흔함), 루프백이 아닌 호스트를 바인딩하세요:

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

반환되는 URL JSON에 표시되는 호스트 이름을 제어하려면 `--url-host`를 사용하세요.

## 루프 (The Loop)

1. **서버 작동 확인**, 그 후 `screen_dir`에 새 파일로 **HTML 작성**:
   - **필수: URL을 언급하거나 화면을 푸시하기 전에 서버가 정상 작동 중인지 확인하세요.** `$STATE_DIR/server-info`가 존재하고 `$STATE_DIR/server-stopped`가 존재하지 않는지 확인합니다. 종료된 경우 **동일한 `--project-dir`**로 `start-server.sh`를 실행해 재시작합니다 — 동일 포트를 재사용하므로 사용자의 열려 있는 탭이 자동으로 재연결되며 (서버 중지 중에는 "일시 정지" 오버레이 표시), 새 URL을 보낼 필요가 없습니다. 서버는 4시간 유휴 상태 시 자동 종료됩니다 (`--idle-timeout-minutes`로 설정 가능).
   - 의미 있는 파일명 사용: `platform.html`, `visual-style.html`, `layout.html`
   - **파일 이름을 재사용하지 마세요** — 각 화면은 새 파일을 사용해야 합니다
   - 파일 생성 툴을 사용하세요 — **cat/heredoc은 절대 사용 금지** (터미널에 노이즈를 출력함)
   - 서버가 가장 최신 파일을 자동으로 제공합니다

2. **사용자에게 기대 사항을 알리고 차례를 마칩니다:**
   - 첫 단계뿐만 아니라 매 단계마다 URL을 상기시켜 주세요.
   - 화면에 표시된 내용을 간단히 텍스트로 요약해 주세요 (예: "홈페이지 레이아웃 옵션 3가지를 보여줍니다").
   - 터미널에서 응답하도록 요청하세요: "확인해 보시고 의견을 알려주세요. 원하시면 클릭하여 옵션을 선택하셔도 됩니다."

3. **다음 차례 시** — 사용자가 터미널에서 응답한 후:
   - `$STATE_DIR/events` 파일이 존재하면 읽으세요 — 사용자 브라우저 상호작용(클릭, 선택)이 JSON 라인으로 포함되어 있습니다.
   - 사용자의 터미널 텍스트와 병합하여 전체적인 내용을 파악하세요.
   - 터미널 메시지가 주된 피드백이며, `state_dir/events`는 구조화된 상호작용 데이터를 제공합니다.

4. **반복 검토 및 진전** — 피드백에 의해 현재 화면 수정이 필요하면 새 파일을 작성하세요 (예: `layout-v2.html`). 현재 단계가 검증되었을 때만 다음 질문으로 넘어가세요.

5. **터미널로 돌아올 때 언로드** — 다음 단계에 브라우저가 필요 없는 경우(예: 명확화 질문, 트레이드오프 논의), 오래된 콘텐츠를 지우기 위해 대기 화면을 푸시하세요:

   ```html
   <!-- filename: waiting.html (or waiting-2.html, etc.) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   이를 통해 대화가 진행되었음에도 사용자가 이미 해결된 선택 화면을 응시하는 것을 방지합니다. 다음 시각적 질문이 나오면 평소처럼 새 콘텐츠 파일을 푸시하세요.

6. 완료될 때까지 반복합니다.

## 콘텐츠 조각 작성하기 (Writing Content Fragments)

페이지 내부에 들어갈 콘텐츠만 작성하세요. 서버가 프레임 템플릿(헤더, 테마 CSS, 연결 상태 및 모든 대화형 인프라)으로 이를 자동 감싸줍니다.

**최소 예시:**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

이게 전부입니다. `<html>`, CSS, `<script>` 태그는 필요하지 않습니다. 서버가 이 모든 것을 제공합니다.

## 사용 가능한 CSS 클래스 (CSS Classes Available)

프레임 템플릿은 콘텐츠용으로 다음 CSS 클래스들을 제공합니다:

### 옵션 (A/B/C 선택)

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**다중 선택:** 컨테이너에 `data-multiselect`를 추가하면 사용자가 여러 옵션을 선택할 수 있습니다. 클릭할 때마다 해당 항목의 선택 스타일이 토글됩니다.

```html
<div class="options" data-multiselect>
  <!-- 동일한 옵션 마크업 — 사용자가 여러 항목을 선택/해제할 수 있음 -->
</div>
```

### 카드 (시각적 디자인)

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### 목업 컨테이너 (Mockup container)

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### 분할 뷰 (나란히 배치)

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### 장단점 (Pros/Cons)

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### 목업 엘리먼트 (와이어프레임 구성 요소)

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 타이포그래피 및 섹션

- `h2` — 페이지 제목
- `h3` — 섹션 헤딩
- `.subtitle` — 제목 아래 보조 텍스트
- `.section` — 하단 여백이 있는 콘텐츠 블록
- `.label` — 소문자/대문자 라벨 텍스트

## 브라우저 이벤트 형식 (Browser Events Format)

사용자가 브라우저에서 옵션을 클릭하면 그 상호작용이 `$STATE_DIR/events`에 기록됩니다 (한 줄당 하나의 JSON 객체). 이 파일은 새 화면을 푸시할 때 자동으로 지워집니다.

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

전체 이벤트 스트림은 사용자의 탐색 경로를 보여줍니다 — 최종 결정 전 여러 옵션을 클릭해 볼 수 있습니다. 마지막 `choice` 이벤트가 보통 최종 선택이지만, 클릭 패턴을 통해 사용자 질문이나 유의미한 선호도를 파악할 수 있습니다.

`$STATE_DIR/events` 파일이 없다면 사용자가 브라우저와 상호작용하지 않은 것이므로 터미널 텍스트만 사용하세요.

## 디자인 팁 (Design Tips)

- **질문에 맞춰 완성도를 조절하세요** — 레이아웃 질문에는 와이어프레임, 디테일 질문에는 다듬어진 완성도 적용
- **각 페이지마다 질문을 설명하세요** — 단순한 "하나를 선택하세요"가 아니라 "어떤 레이아웃이 더 전문적으로 느껴지나요?"와 같이 작성
- **다음 단계로 넘어가기 전에 수정 반영하세요** — 피드백으로 인해 현재 화면이 달라진 경우 새 버전 작성
- **화면당 2~4개 옵션 권장**
- **중요한 경우에는 실제 콘텐츠 사용** — 포토그래피 포트폴리오의 경우 실제 이미지(Unsplash 등)를 사용하세요. 플레이스홀더 콘텐츠는 디자인 문제를 가릴 수 있습니다.
- **목업은 단순하게 유지** — 픽셀 단위의 완벽한 디자인보다 레이아웃과 구조에 집중하세요

## 파일 이름 지정 (File Naming)

- 의미 있는 파일명 사용: `platform.html`, `visual-style.html`, `layout.html`
- 파일 이름을 절대로 재사용하지 마세요 — 각 화면은 새 파일이어야 합니다
- 수정 버전의 경우: `layout-v2.html`, `layout-v3.html`과 같이 버전 접미사 추가
- 서버는 수정 시간 기준으로 가장 최신 파일을 제공합니다

## 정리하기 (Cleaning Up)

```bash
scripts/stop-server.sh $SESSION_DIR
```

세션에서 `--project-dir`을 사용한 경우, 목업 파일은 향후 참조를 위해 `.superpowers/brainstorm/`에 보존됩니다. `/tmp` 세션만 중지 시 삭제됩니다.

## 참조 (Reference)

- 프레임 템플릿 (CSS 참조): `scripts/frame-template.html`
- 헬퍼 스크립트 (클라이언트 측): `scripts/helper.js`
