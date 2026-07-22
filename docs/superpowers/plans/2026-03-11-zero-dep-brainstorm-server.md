# 의존성 없는(Zero-Dependency) 브레인스토밍 서버 구현 계획

> **에이전트 작업자 참고:** 필수: 이 계획을 구현하려면 superpowers:subagent-driven-development (서브에이전트가 사용 가능한 경우) 또는 superpowers:executing-plans를 사용하세요. 추적을 위해 단계에 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** 브레인스토밍 서버의 벤더링된 node_modules를 Node 내장 모듈만 사용하는 단일 의존성 없는 `server.js`로 교체합니다.

**아키텍처:** WebSocket 프로토콜(RFC 6455 텍스트 프레임), HTTP 서버(`http` 모듈), 파일 감시(`fs.watch`)를 지원하는 단일 파일입니다. 모듈로 가져올 때 단위 테스트를 위해 프로토콜 함수를 내보냅니다.

**기술 스택:** Node.js 내장 모듈만 사용: `http`, `crypto`, `fs`, `path`

**명세:** `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`

**기존 테스트:** `tests/brainstorm-server/ws-protocol.test.js` (단위 테스트), `tests/brainstorm-server/server.test.js` (통합 테스트)

---

## 파일 맵

- **생성:** `skills/brainstorming/scripts/server.js` — 의존성 없는 교체용 파일
- **수정:** `skills/brainstorming/scripts/start-server.sh:94,100` — `index.js`를 `server.js`로 변경
- **수정:** `.gitignore:6` — `!skills/brainstorming/scripts/node_modules/` 예외 제거
- **삭제:** `skills/brainstorming/scripts/index.js`
- **삭제:** `skills/brainstorming/scripts/package.json`
- **삭제:** `skills/brainstorming/scripts/package-lock.json`
- **삭제:** `skills/brainstorming/scripts/node_modules/` (714개 파일)
- **변경 없음:** `skills/brainstorming/scripts/helper.js`, `skills/brainstorming/scripts/frame-template.html`, `skills/brainstorming/scripts/stop-server.sh`

---

## 청크 1: WebSocket 프로토콜 레이어

### 작업 1: WebSocket 프로토콜 export 구현

**파일:**
- 생성: `skills/brainstorming/scripts/server.js`
- 테스트: `tests/brainstorm-server/ws-protocol.test.js` (이미 존재함)

- [ ] **단계 1: OPCODES 상수 및 computeAcceptKey를 포함하여 server.js 생성**

```js
const crypto = require('crypto');

const OPCODES = { TEXT: 0x01, CLOSE: 0x08, PING: 0x09, PONG: 0x0A };
const WS_MAGIC = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

function computeAcceptKey(clientKey) {
  return crypto.createHash('sha1').update(clientKey + WS_MAGIC).digest('base64');
}
```

- [ ] **단계 2: encodeFrame 구현**

서버 프레임은 마스킹되지 않습니다. 세 가지 길이 인코딩:
- 페이로드 < 126: 2바이트 헤더 (FIN+opcode, 길이)
- 126-65535: 4바이트 헤더 (FIN+opcode, 126, 16비트 길이)
- > 65535: 10바이트 헤더 (FIN+opcode, 127, 64비트 길이)

```js
function encodeFrame(opcode, payload) {
  const fin = 0x80;
  const len = payload.length;
  let header;

  if (len < 126) {
    header = Buffer.alloc(2);
    header[0] = fin | opcode;
    header[1] = len;
  } else if (len < 65536) {
    header = Buffer.alloc(4);
    header[0] = fin | opcode;
    header[1] = 126;
    header.writeUInt16BE(len, 2);
  } else {
    header = Buffer.alloc(10);
    header[0] = fin | opcode;
    header[1] = 127;
    header.writeBigUInt64BE(BigInt(len), 2);
  }

  return Buffer.concat([header, payload]);
}
```

- [ ] **단계 3: decodeFrame 구현**

클라이언트 프레임은 항상 마스킹됩니다. 불완전한 경우 `{ opcode, payload, bytesConsumed }` 또는 `null`을 반환합니다. 마스킹되지 않은 프레임에 대해서는 예외를 던집니다.

```js
function decodeFrame(buffer) {
  if (buffer.length < 2) return null;

  const firstByte = buffer[0];
  const secondByte = buffer[1];
  const opcode = firstByte & 0x0F;
  const masked = (secondByte & 0x80) !== 0;
  let payloadLen = secondByte & 0x7F;
  let offset = 2;

  if (!masked) throw new Error('Client frames must be masked');

  if (payloadLen === 126) {
    if (buffer.length < 4) return null;
    payloadLen = buffer.readUInt16BE(2);
    offset = 4;
  } else if (payloadLen === 127) {
    if (buffer.length < 10) return null;
    payloadLen = Number(buffer.readBigUInt64BE(2));
    offset = 10;
  }

  const maskOffset = offset;
  const dataOffset = offset + 4;
  const totalLen = dataOffset + payloadLen;
  if (buffer.length < totalLen) return null;

  const mask = buffer.slice(maskOffset, dataOffset);
  const data = Buffer.alloc(payloadLen);
  for (let i = 0; i < payloadLen; i++) {
    data[i] = buffer[dataOffset + i] ^ mask[i % 4];
  }

  return { opcode, payload: data, bytesConsumed: totalLen };
}
```

- [ ] **단계 4: 파일 하단에 모듈 export 추가**

```js
module.exports = { computeAcceptKey, encodeFrame, decodeFrame, OPCODES };
```

- [ ] **단계 5: 단위 테스트 실행**

실행: `cd tests/brainstorm-server && node ws-protocol.test.js`
예상 결과: 모든 테스트 통과 (핸드셰이크, 인코딩, 디코딩, 경계값, 엣지 케이스)

- [ ] **단계 6: 커밋**

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add WebSocket protocol layer for zero-dep brainstorm server"
```

---

## 청크 2: HTTP 서버 및 애플리케이션 로직

### 작업 2: HTTP 서버, 파일 감시, WebSocket 연결 처리 추가

**파일:**
- 수정: `skills/brainstorming/scripts/server.js`
- 테스트: `tests/brainstorm-server/server.test.js` (이미 존재함)

- [ ] **단계 1: server.js 상단(require 구문 다음)에 설정 및 상수 추가**

```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
const HOST = process.env.BRAINSTORM_HOST || '127.0.0.1';
const URL_HOST = process.env.BRAINSTORM_URL_HOST || (HOST === '127.0.0.1' ? 'localhost' : HOST);
const SCREEN_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm';

const MIME_TYPES = {
  '.html': 'text/html', '.css': 'text/css', '.js': 'application/javascript',
  '.json': 'application/json', '.png': 'image/png', '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg', '.gif': 'image/gif', '.svg': 'image/svg+xml'
};
```

- [ ] **단계 2: WAITING_PAGE, 모듈 스코프 템플릿 로딩 및 헬퍼 함수 추가**

모듈 스코프에서 `frameTemplate` 및 `helperInjection`을 로드하여 `wrapInFrame` 및 `handleRequest`에서 접근할 수 있도록 합니다. 이는 `__dirname`(scripts 디렉터리)에서만 파일을 읽으므로 모듈로 require되든 직접 실행되든 유효합니다.

```js
const WAITING_PAGE = `<!DOCTYPE html>
<html>
<head><title>Brainstorm Companion</title>
<style>body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
h1 { color: #333; } p { color: #666; }</style>
</head>
<body><h1>Brainstorm Companion</h1>
<p>Waiting for Claude to push a screen...</p></body></html>`;

const frameTemplate = fs.readFileSync(path.join(__dirname, 'frame-template.html'), 'utf-8');
const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
const helperInjection = '<script>\n' + helperScript + '\n</script>';

function isFullDocument(html) {
  const trimmed = html.trimStart().toLowerCase();
  return trimmed.startsWith('<!doctype') || trimmed.startsWith('<html');
}

function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}

function getNewestScreen() {
  const files = fs.readdirSync(SCREEN_DIR)
    .filter(f => f.endsWith('.html'))
    .map(f => {
      const fp = path.join(SCREEN_DIR, f);
      return { path: fp, mtime: fs.statSync(fp).mtime.getTime() };
    })
    .sort((a, b) => b.mtime - a.mtime);
  return files.length > 0 ? files[0].path : null;
}
```

- [ ] **단계 3: HTTP 요청 핸들러 추가**

```js
function handleRequest(req, res) {
  if (req.method === 'GET' && req.url === '/') {
    const screenFile = getNewestScreen();
    let html = screenFile
      ? (raw => isFullDocument(raw) ? raw : wrapInFrame(raw))(fs.readFileSync(screenFile, 'utf-8'))
      : WAITING_PAGE;

    if (html.includes('</body>')) {
      html = html.replace('</body>', helperInjection + '\n</body>');
    } else {
      html += helperInjection;
    }

    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(html);
  } else if (req.method === 'GET' && req.url.startsWith('/files/')) {
    const fileName = req.url.slice(7); // strip '/files/'
    const filePath = path.join(SCREEN_DIR, path.basename(fileName));
    if (!fs.existsSync(filePath)) {
      res.writeHead(404);
      res.end('Not found');
      return;
    }
    const ext = path.extname(filePath).toLowerCase();
    const contentType = MIME_TYPES[ext] || 'application/octet-stream';
    res.writeHead(200, { 'Content-Type': contentType });
    res.end(fs.readFileSync(filePath));
  } else {
    res.writeHead(404);
    res.end('Not found');
  }
}
```

- [ ] **단계 4: WebSocket 연결 처리 추가**

```js
const clients = new Set();

function handleUpgrade(req, socket) {
  const key = req.headers['sec-websocket-key'];
  if (!key) { socket.destroy(); return; }

  const accept = computeAcceptKey(key);
  socket.write(
    'HTTP/1.1 101 Switching Protocols\r\n' +
    'Upgrade: websocket\r\n' +
    'Connection: Upgrade\r\n' +
    'Sec-WebSocket-Accept: ' + accept + '\r\n\r\n'
  );

  let buffer = Buffer.alloc(0);
  clients.add(socket);

  socket.on('data', (chunk) => {
    buffer = Buffer.concat([buffer, chunk]);
    while (buffer.length > 0) {
      let result;
      try {
        result = decodeFrame(buffer);
      } catch (e) {
        socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
        clients.delete(socket);
        return;
      }
      if (!result) break;
      buffer = buffer.slice(result.bytesConsumed);

      switch (result.opcode) {
        case OPCODES.TEXT:
          handleMessage(result.payload.toString());
          break;
        case OPCODES.CLOSE:
          socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
          clients.delete(socket);
          return;
        case OPCODES.PING:
          socket.write(encodeFrame(OPCODES.PONG, result.payload));
          break;
        case OPCODES.PONG:
          break;
        default:
          // Unsupported opcode — close with 1003
          const closeBuf = Buffer.alloc(2);
          closeBuf.writeUInt16BE(1003);
          socket.end(encodeFrame(OPCODES.CLOSE, closeBuf));
          clients.delete(socket);
          return;
      }
    }
  });

  socket.on('close', () => clients.delete(socket));
  socket.on('error', () => clients.delete(socket));
}

function handleMessage(text) {
  let event;
  try {
    event = JSON.parse(text);
  } catch (e) {
    console.error('Failed to parse WebSocket message:', e.message);
    return;
  }
  console.log(JSON.stringify({ source: 'user-event', ...event }));
  if (event.choice) {
    const eventsFile = path.join(SCREEN_DIR, '.events');
    fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
  }
}

function broadcast(msg) {
  const frame = encodeFrame(OPCODES.TEXT, Buffer.from(JSON.stringify(msg)));
  for (const socket of clients) {
    try { socket.write(frame); } catch (e) { clients.delete(socket); }
  }
}
```

- [ ] **단계 5: 디바운스 타이머 맵 추가**

```js
const debounceTimers = new Map();
```

파일 감시 로직은 와처의 수명주기를 서버 수명주기와 함께 유지하고 명세에 따라 `error` 핸들러를 포함하도록 `startServer`(단계 6) 내에 인라인됩니다.

- [ ] **단계 6: startServer 함수 및 조건부 메인 실행 추가**

`frameTemplate` 및 `helperInjection`은 이미 모듈 스코프(단계 2)에 있습니다. `startServer`는 화면 디렉터리를 생성하고, HTTP 서버와 와처를 시작하며, 시작 정보를 로깅합니다.

```js
function startServer() {
  if (!fs.existsSync(SCREEN_DIR)) fs.mkdirSync(SCREEN_DIR, { recursive: true });

  const server = http.createServer(handleRequest);
  server.on('upgrade', handleUpgrade);

  const watcher = fs.watch(SCREEN_DIR, (eventType, filename) => {
    if (!filename || !filename.endsWith('.html')) return;
    if (debounceTimers.has(filename)) clearTimeout(debounceTimers.get(filename));
    debounceTimers.set(filename, setTimeout(() => {
      debounceTimers.delete(filename);
      const filePath = path.join(SCREEN_DIR, filename);
      if (eventType === 'rename' && fs.existsSync(filePath)) {
        const eventsFile = path.join(SCREEN_DIR, '.events');
        if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);
        console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      } else if (eventType === 'change') {
        console.log(JSON.stringify({ type: 'screen-updated', file: filePath }));
      }
      broadcast({ type: 'reload' });
    }, 100));
  });
  watcher.on('error', (err) => console.error('fs.watch error:', err.message));

  server.listen(PORT, HOST, () => {
    const info = JSON.stringify({
      type: 'server-started', port: Number(PORT), host: HOST,
      url_host: URL_HOST, url: 'http://' + URL_HOST + ':' + PORT,
      screen_dir: SCREEN_DIR
    });
    console.log(info);
    fs.writeFileSync(path.join(SCREEN_DIR, '.server-info'), info + '\n');
  });
}

if (require.main === module) {
  startServer();
}
```

- [ ] **단계 7: 통합 테스트 실행**

테스트 디렉터리에는 의존성으로 `ws`가 포함된 `package.json`이 이미 있습니다. 필요한 경우 설치하고 테스트를 실행하세요.

실행: `cd tests/brainstorm-server && npm install && node server.test.js`
예상 결과: 모든 테스트 통과

- [ ] **단계 8: 커밋**

```bash
git add skills/brainstorming/scripts/server.js
git commit -m "Add HTTP server, WebSocket handling, and file watching to server.js"
```

---

## 청크 3: 교체 및 정리

### 작업 3: start-server.sh 업데이트 및 이전 파일 제거

**파일:**
- 수정: `skills/brainstorming/scripts/start-server.sh:94,100`
- 수정: `.gitignore:6`
- 삭제: `skills/brainstorming/scripts/index.js`
- 삭제: `skills/brainstorming/scripts/package.json`
- 삭제: `skills/brainstorming/scripts/package-lock.json`
- 삭제: `skills/brainstorming/scripts/node_modules/` (전체 디렉터리)

- [ ] **단계 1: start-server.sh 업데이트 — `index.js`를 `server.js`로 변경**

변경할 두 줄:

94줄: `env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js`

100줄: `nohup env BRAINSTORM_DIR="$SCREEN_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" node server.js > "$LOG_FILE" 2>&1 &`

- [ ] **단계 2: node_modules에 대한 gitignore 예외 제거**

`.gitignore`에서 6번째 줄 삭제: `!skills/brainstorming/scripts/node_modules/`

- [ ] **단계 3: 이전 파일 삭제**

```bash
git rm skills/brainstorming/scripts/index.js
git rm skills/brainstorming/scripts/package.json
git rm skills/brainstorming/scripts/package-lock.json
git rm -r skills/brainstorming/scripts/node_modules/
```

- [ ] **단계 4: 두 테스트 수트 모두 실행**

실행: `cd tests/brainstorm-server && node ws-protocol.test.js && node server.test.js`
예상 결과: 모든 테스트 통과

- [ ] **단계 5: 커밋**

```bash
git add skills/brainstorming/scripts/ .gitignore
git commit -m "Remove vendored node_modules, swap to zero-dep server.js"
```

### 작업 4: 수동 스모크 테스트

- [ ] **단계 1: 서버 수동 시작**

```bash
cd skills/brainstorming/scripts
BRAINSTORM_DIR=/tmp/brainstorm-smoke BRAINSTORM_PORT=9876 node server.js
```

예상 결과: 포트 9876이 포함된 `server-started` JSON 출력

- [ ] **단계 2: 브라우저에서 http://localhost:9876 열기**

예상 결과: "Waiting for Claude to push a screen..."이 포함된 대기 페이지

- [ ] **단계 3: 화면 디렉터리에 HTML 파일 작성**

```bash
echo '<h2>Hello from smoke test</h2>' > /tmp/brainstorm-smoke/test.html
```

예상 결과: 브라우저가 새로고침되고 프레임 템플릿으로 감싸진 "Hello from smoke test" 표시

- [ ] **단계 4: WebSocket 작동 검증 — 브라우저 콘솔 확인**

브라우저 개발자 도구를 엽니다. WebSocket 연결이 연결됨 상태로 표시되어야 합니다(콘솔 에러 없음). 프레임 템플릿의 상태 표시기에 "Connected"가 표시되어야 합니다.

- [ ] **단계 5: Ctrl-C로 서버 중지 및 정리**

```bash
rm -rf /tmp/brainstorm-smoke
```
