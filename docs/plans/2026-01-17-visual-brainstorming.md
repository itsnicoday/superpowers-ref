# 시각적 브레인스토밍 컴패니언 구현 계획

> **에이전트 작업자 참고:** 필수 서브 스킬: 이 계획을 작업별로 구현하려면 `superpowers:executing-plans`를 사용하세요.

**목표:** 브레인스토밍 세션을 위해 Claude에게 브라우저 기반의 시각적 컴패니언을 제공합니다. 터미널 대화와 함께 목업, 프로토타입 및 대화형 선택 항목을 보여줍니다.

**아키텍처:** Claude가 임시 파일에 HTML을 작성합니다. 로컬 Node.js 서버가 해당 파일을 감시(watch)하고 자동 주입된 헬퍼 라이브러리와 함께 제공합니다. 사용자 상호작용은 WebSocket을 통해 서버 stdout으로 전달되며, Claude는 백그라운드 작업 출력에서 이를 확인합니다.

**기술 스택:** Node.js, Express, ws (WebSocket), chokidar (파일 감시)

---

## Task 1: 서버 기반 생성

**파일:**
- 생성: `lib/brainstorm-server/index.js`
- 생성: `lib/brainstorm-server/package.json`

**Step 1: package.json 생성**

```json
{
  "name": "brainstorm-server",
  "version": "1.0.0",
  "description": "Visual brainstorming companion server for Claude Code",
  "main": "index.js",
  "dependencies": {
    "chokidar": "^3.5.3",
    "express": "^4.18.2",
    "ws": "^8.14.2"
  }
}
```

**Step 2: 실행 가능한 최소 서버 생성**

```javascript
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const chokidar = require('chokidar');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || 3333;
const SCREEN_FILE = process.env.BRAINSTORM_SCREEN || '/tmp/brainstorm/screen.html';
const SCREEN_DIR = path.dirname(SCREEN_FILE);

// Ensure screen directory exists
if (!fs.existsSync(SCREEN_DIR)) {
  fs.mkdirSync(SCREEN_DIR, { recursive: true });
}

// Create default screen if none exists
if (!fs.existsSync(SCREEN_FILE)) {
  fs.writeFileSync(SCREEN_FILE, `<!DOCTYPE html>
<html>
<head>
  <title>Brainstorm Companion</title>
  <style>
    body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
    h1 { color: #333; }
    p { color: #666; }
  </style>
</head>
<body>
  <h1>Brainstorm Companion</h1>
  <p>Waiting for Claude to push a screen...</p>
</body>
</html>`);
}

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// Track connected browsers for reload notifications
const clients = new Set();

wss.on('connection', (ws) => {
  clients.add(ws);
  ws.on('close', () => clients.delete(ws));

  ws.on('message', (data) => {
    // User interaction event - write to stdout for Claude
    const event = JSON.parse(data.toString());
    console.log(JSON.stringify({ type: 'user-event', ...event }));
  });
});

// Serve current screen with helper.js injected
app.get('/', (req, res) => {
  let html = fs.readFileSync(SCREEN_FILE, 'utf-8');

  // Inject helper script before </body>
  const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
  const injection = `<script>\n${helperScript}\n</script>`;

  if (html.includes('</body>')) {
    html = html.replace('</body>', `${injection}\n</body>`);
  } else {
    html += injection;
  }

  res.type('html').send(html);
});

// Watch for screen file changes
chokidar.watch(SCREEN_FILE).on('change', () => {
  console.log(JSON.stringify({ type: 'screen-updated', file: SCREEN_FILE }));
  // Notify all browsers to reload
  clients.forEach(ws => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'reload' }));
    }
  });
});

server.listen(PORT, '127.0.0.1', () => {
  console.log(JSON.stringify({ type: 'server-started', port: PORT, url: `http://localhost:${PORT}` }));
});
```

**Step 3: npm install 실행**

실행: `cd lib/brainstorm-server && npm install`
예상 결과: 의존성 설치 완료

**Step 4: 서버 시작 테스트**

실행: `cd lib/brainstorm-server && timeout 3 node index.js || true`
예상 결과: `server-started` 및 포트 정보가 포함된 JSON 표시

**Step 5: 커밋**

```bash
git add lib/brainstorm-server/
git commit -m "feat: add brainstorm server foundation"
```

---

## Task 2: 헬퍼 라이브러리 생성

**파일:**
- 생성: `lib/brainstorm-server/helper.js`

**Step 1: 이벤트 자동 캡처가 포함된 helper.js 생성**

```javascript
(function() {
  const WS_URL = 'ws://' + window.location.host;
  let ws = null;
  let eventQueue = [];

  function connect() {
    ws = new WebSocket(WS_URL);

    ws.onopen = () => {
      // Send any queued events
      eventQueue.forEach(e => ws.send(JSON.stringify(e)));
      eventQueue = [];
    };

    ws.onmessage = (msg) => {
      const data = JSON.parse(msg.data);
      if (data.type === 'reload') {
        window.location.reload();
      }
    };

    ws.onclose = () => {
      // Reconnect after 1 second
      setTimeout(connect, 1000);
    };
  }

  function send(event) {
    event.timestamp = Date.now();
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(event));
    } else {
      eventQueue.push(event);
    }
  }

  // Auto-capture clicks on interactive elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('button, a, [data-choice], [role="button"], input[type="submit"]');
    if (!target) return;

    // Don't capture regular link navigation
    if (target.tagName === 'A' && !target.dataset.choice) return;

    e.preventDefault();

    send({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice || null,
      id: target.id || null,
      className: target.className || null
    });
  });

  // Auto-capture form submissions
  document.addEventListener('submit', (e) => {
    e.preventDefault();
    const form = e.target;
    const formData = new FormData(form);
    const data = {};
    formData.forEach((value, key) => { data[key] = value; });

    send({
      type: 'submit',
      formId: form.id || null,
      formName: form.name || null,
      data: data
    });
  });

  // Auto-capture input changes (debounced)
  let inputTimeout = null;
  document.addEventListener('input', (e) => {
    const target = e.target;
    if (!target.matches('input, textarea, select')) return;

    clearTimeout(inputTimeout);
    inputTimeout = setTimeout(() => {
      send({
        type: 'input',
        name: target.name || null,
        id: target.id || null,
        value: target.value,
        inputType: target.type || target.tagName.toLowerCase()
      });
    }, 500); // 500ms debounce
  });

  // Expose for explicit use if needed
  window.brainstorm = {
    send: send,
    choice: (value, metadata = {}) => send({ type: 'choice', value, ...metadata })
  };

  connect();
})();
```

**Step 2: helper.js 구문 유효성 검증**

실행: `node -c lib/brainstorm-server/helper.js`
예상 결과: 구문 오류 없음

**Step 3: 커밋**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "feat: add browser helper library for event capture"
```

---

## Task 3: 서버 테스트 작성

**파일:**
- 생성: `tests/brainstorm-server/server.test.js`
- 생성: `tests/brainstorm-server/package.json`

**Step 1: 테스트 package.json 생성**

```json
{
  "name": "brainstorm-server-tests",
  "version": "1.0.0",
  "scripts": {
    "test": "node server.test.js"
  }
}
```

**Step 2: 서버 테스트 작성**

```javascript
const { spawn } = require('child_process');
const http = require('http');
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');
const assert = require('assert');

const SERVER_PATH = path.join(__dirname, '../../lib/brainstorm-server/index.js');
const TEST_PORT = 3334;
const TEST_SCREEN = '/tmp/brainstorm-test/screen.html';

// Clean up test directory
function cleanup() {
  if (fs.existsSync(path.dirname(TEST_SCREEN))) {
    fs.rmSync(path.dirname(TEST_SCREEN), { recursive: true });
  }
}

async function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function fetch(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve({ status: res.statusCode, body: data }));
    }).on('error', reject);
  });
}

async function runTests() {
  cleanup();

  // Start server
  const server = spawn('node', [SERVER_PATH], {
    env: { ...process.env, BRAINSTORM_PORT: TEST_PORT, BRAINSTORM_SCREEN: TEST_SCREEN }
  });

  let stdout = '';
  server.stdout.on('data', (data) => { stdout += data.toString(); });
  server.stderr.on('data', (data) => { console.error('Server stderr:', data.toString()); });

  await sleep(1000); // Wait for server to start

  try {
    // Test 1: Server starts and outputs JSON
    console.log('Test 1: Server startup message');
    assert(stdout.includes('server-started'), 'Should output server-started');
    assert(stdout.includes(TEST_PORT.toString()), 'Should include port');
    console.log('  PASS');

    // Test 2: GET / returns HTML with helper injected
    console.log('Test 2: Serves HTML with helper injected');
    const res = await fetch(`http://localhost:${TEST_PORT}/`);
    assert.strictEqual(res.status, 200);
    assert(res.body.includes('brainstorm'), 'Should include brainstorm content');
    assert(res.body.includes('WebSocket'), 'Should have helper.js injected');
    console.log('  PASS');

    // Test 3: WebSocket connection and event relay
    console.log('Test 3: WebSocket relays events to stdout');
    stdout = ''; // Reset stdout capture
    const ws = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws.on('open', resolve));

    ws.send(JSON.stringify({ type: 'click', text: 'Test Button' }));
    await sleep(100);

    assert(stdout.includes('user-event'), 'Should relay user events');
    assert(stdout.includes('Test Button'), 'Should include event data');
    ws.close();
    console.log('  PASS');

    // Test 4: File change triggers reload notification
    console.log('Test 4: File change notifies browsers');
    const ws2 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws2.on('open', resolve));

    let gotReload = false;
    ws2.on('message', (data) => {
      const msg = JSON.parse(data.toString());
      if (msg.type === 'reload') gotReload = true;
    });

    // Modify the screen file
    fs.writeFileSync(TEST_SCREEN, '<html><body>Updated</body></html>');
    await sleep(500);

    assert(gotReload, 'Should send reload message on file change');
    ws2.close();
    console.log('  PASS');

    console.log('\nAll tests passed!');

  } finally {
    server.kill();
    cleanup();
  }
}

runTests().catch(err => {
  console.error('Test failed:', err);
  process.exit(1);
});
```

**Step 3: 테스트 실행**

실행: `cd tests/brainstorm-server && npm install ws && node server.test.js`
예상 결과: 모든 테스트 통과

**Step 4: 커밋**

```bash
git add tests/brainstorm-server/
git commit -m "test: add brainstorm server integration tests"
```

---

## Task 4: 브레인스토밍 스킬에 Visual Companion 추가

**파일:**
- 수정: `skills/brainstorming/SKILL.md`
- 생성: `skills/brainstorming/visual-companion.md` (참고 문서)

**Step 1: 참고 문서 생성**

`skills/brainstorming/visual-companion.md` 생성:

```markdown
# Visual Companion Reference

## Starting the Server

Run as a background job:

```bash
node ${PLUGIN_ROOT}/lib/brainstorm-server/index.js
```

Tell the user: "I've started a visual companion at http://localhost:3333 - open it in a browser."

## Pushing Screens

Write HTML to `/tmp/brainstorm/screen.html`. The server watches this file and auto-refreshes the browser.

## Reading User Responses

Check the background task output for JSON events:

```json
{"type":"user-event","type":"click","text":"Option A","choice":"optionA","timestamp":1234567890}
{"type":"user-event","type":"submit","data":{"notes":"My feedback"},"timestamp":1234567891}
```

Event types:
- **click**: User clicked button or `data-choice` element
- **submit**: User submitted form (includes all form data)
- **input**: User typed in field (debounced 500ms)

## HTML Patterns

### Choice Cards

```html
<div class="options">
  <button data-choice="optionA">
    <h3>Option A</h3>
    <p>Description</p>
  </button>
  <button data-choice="optionB">
    <h3>Option B</h3>
    <p>Description</p>
  </button>
</div>
```

### Interactive Mockup

```html
<div class="mockup">
  <header data-choice="header">App Header</header>
  <nav data-choice="nav">Navigation</nav>
  <main data-choice="main">Content</main>
</div>
```

### Form with Notes

```html
<form>
  <label>Priority: <input type="range" name="priority" min="1" max="5"></label>
  <textarea name="notes" placeholder="Additional thoughts..."></textarea>
  <button type="submit">Submit</button>
</form>
```

### Explicit JavaScript

```html
<button onclick="brainstorm.choice('custom', {extra: 'data'})">Custom</button>
```
```

**Step 2: 브레인스토밍 스킬에 시각적 컴패니언 섹션 추가**

`skills/brainstorming/SKILL.md`에서 "Key Principles" 항목 뒤에 추가합니다:

```markdown

## Visual Companion (Optional)

When brainstorming involves visual elements - UI mockups, wireframes, interactive prototypes - use the browser-based visual companion.

**When to use:**
- Presenting UI/UX options that benefit from visual comparison
- Showing wireframes or layout options
- Gathering structured feedback (ratings, forms)
- Prototyping click interactions

**How it works:**
1. Start the server as a background job
2. Tell user to open http://localhost:3333
3. Write HTML to `/tmp/brainstorm/screen.html` (auto-refreshes)
4. Check background task output for user interactions

The terminal remains the primary conversation interface. The browser is a visual aid.

**Reference:** See `visual-companion.md` in this skill directory for HTML patterns and API details.
```

**Step 3: 편집 내용 검증**

실행: `grep -A5 "Visual Companion" skills/brainstorming/SKILL.md`
예상 결과: 새 섹션을 보여줌

**Step 4: 커밋**

```bash
git add skills/brainstorming/
git commit -m "feat: add visual companion to brainstorming skill"
```

---

## Task 5: 플러그인 Ignore에 서버 추가 (선택적 정리)

**파일:**
- `.gitignore`에서 `lib/brainstorm-server`의 node_modules 제외 필요성 확인

**Step 1: 현재 gitignore 확인**

실행: `cat .gitignore 2>/dev/null || echo "No .gitignore"`

**Step 2: 필요한 경우 node_modules 추가**

아직 존재하지 않는 경우 추가합니다:
```
lib/brainstorm-server/node_modules/
```

**Step 3: 변경된 경우 커밋**

```bash
git add .gitignore
git commit -m "chore: ignore brainstorm-server node_modules"
```

---

## 요약

모든 작업 완료 후:

1. `lib/brainstorm-server/`에 **서버** 생성 - HTML 파일을 감시하고 이벤트를 전달하는 Node.js 서버
2. 클릭, 폼, 입력을 캡처하는 **헬퍼 라이브러리**가 자동 주입됨
3. `tests/brainstorm-server/`에 서버 동작을 검증하는 **테스트** 추가
4. Visual Companion 섹션 및 `visual-companion.md` 참고 문서로 **Brainstorming 스킬** 업데이트 완료

**사용 방법:**
1. 백그라운드 작업으로 서버 시작: `node lib/brainstorm-server/index.js &`
2. 사용자에게 `http://localhost:3333`을 열도록 안내
3. `/tmp/brainstorm/screen.html`에 HTML 작성
4. 사용자 이벤트 발생 시 작업 출력 확인
