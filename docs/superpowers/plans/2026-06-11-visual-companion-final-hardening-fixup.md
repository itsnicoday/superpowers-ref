# 비주얼 컴패니언 최종 강화 보완(Fixup) 구현 계획

> **에이전트 작업자 참고:** 필수 하위 스킬: 이 계획을 작업별로 구현하려면 superpowers:subagent-driven-development (권장) 또는 superpowers:executing-plans를 사용하세요. 각 단계는 추적을 위해 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** 테스트 우선 변경 사항, 깨끗한 리베이스 상태, 리뷰어 제공용 검증 근거를 바탕으로 PR #1720의 최종 강화 보완(fixup)을 완료합니다.

**스펙:** `docs/superpowers/specs/2026-06-11-visual-companion-final-hardening-fixup-design.md`

**아키텍처:** 컴패니언의 의존성 제로(zero-dependency) 및 로컬 우선(local-first) 특성을 유지합니다. 기존 서버 및 셸 스크립트에 집중적인 가드(guard)를 추가합니다: 루트 화면 선택 시 `/files/*` 격리 가드를 재사용하고, 폴백 토큰 처리는 토큰 소스를 추적하며, 수명 주기 종료(lifecycle shutdown) 시 소유권 증명을 위해 시작 시 생성된 커맨드라인 인스턴스 ID를 사용합니다.

**기술 스택:** Node.js 내장 모듈 (`http`, `fs`, `path`, `crypto`), 기존 `ws` 테스트 의존성, Bash 스크립트, Windows 환경의 Git Bash, PR 메타데이터용 `gh` CLI.

**커밋 지침:** 각 작업에는 제안된 커밋이 포함되어 있습니다. 하위 에이전트 기반 실행 시, 오케스트레이터는 작업자의 diff를 검토하고 작업 검증을 실행한 후 커밋을 수행합니다.

---

## 파일 맵

- 수정: `skills/brainstorming/scripts/server.cjs`
  - `isRegularFileInsideContentDir()`를 통해 루트 화면 후보를 필터링합니다.
  - 토큰 소스를 추적하고 폴백 시 순환(rotate)하거나 닫힌 상태로 실패(fail closed) 처리합니다.
- 수정: `skills/brainstorming/scripts/start-server.sh`
  - `state/server-instance-id`를 생성합니다.
  - `server.cjs` 뒤에 `--brainstorm-server-id=<id>`를 전달합니다.
- Modify: `skills/brainstorming/scripts/stop-server.sh`
  - PID에 신호를 보내기 전에 정확한 instance-id argv 증명을 요구합니다.
  - 오래되었거나 종료된 결과 발생 시 더 이상 유효하지 않은 `server.pid` 및 `server-instance-id`를 제거합니다.
- 수정: `tests/brainstorm-server/server.test.js`
  - 고정 포트 시작 가드를 추가합니다.
  - 심볼릭 링크 기능에 대해 건너뛰기 지원(skip-aware) 테스트 하네스를 추가합니다.
  - 루트 심볼릭 링크 및 하드 링크 탈출 회귀 테스트를 추가합니다.
- 수정: `tests/brainstorm-server/auth.test.js`
  - 고정 포트 시작 가드를 추가합니다.
- 수정: `tests/brainstorm-server/lifecycle.test.js`
  - 폴백 토큰 순환, 명시적 토큰 닫힌 실패(fail-closed), 폴백 키 거부 회귀 테스트를 추가합니다.
- 수정: `tests/brainstorm-server/stop-server.test.sh`
  - 최상위 정리 트랩(trap)을 추가합니다.
  - 긍정 및 부정 server-instance-id 소유권 테스트를 추가합니다.
- 수정: `tests/brainstorm-server/start-server.test.sh`
  - Windows 스타일의 가짜 node 경로가 정확한 서버 ID argv를 수신하고 유효한 ID 파일을 쓰는지 단정(assert)합니다.
- 수정: `tests/brainstorm-server/windows-lifecycle.test.sh`
  - 직접 Node stop-server 커버리지를 위해 서버 ID argv를 전달합니다.
  - ID argv에 대한 Windows 가짜 node 단정(assertion)을 추가합니다.
- 수정: `skills/brainstorming/visual-companion.md`
  - 자동 열기 동작을 유지해야 하는 플랫폼 명령에 `--open`을 추가합니다.
- 수정: `docs/superpowers/plans/2026-06-09-visual-companion-issues.md`
  - 배포된 범위, WS Origin 어휘, 기본 타임아웃 및 지연된 기능 항목을 조율합니다.
- 추적 파일 외부 업데이트: PR #1720 본문
  - 리베이스 후 diff 상태, RED/GREEN 증거, macOS/Windows 검증, 수동 브라우저 스모크 테스트 및 외부 평가 증거를 기록합니다.

## 작업 0: 리베이스 및 베이스라인 상태

**파일:**
- 소스 수정 없음
- 검증 대상: git 브랜치 상태

- [ ] **1단계: 최신 dev 가져오기**

실행:

```bash
git fetch origin dev
```

예상: 명령어 종료 코드 0.

- [ ] **2단계: 최신 dev 위로 리베이스**

실행:

```bash
git rebase origin/dev
```

예상: 명령어 종료 코드 0, 또는 `evals`에 대해 `origin/dev`를 가져와서 해결해야 하는 충돌에서만 중단됨.

- [ ] **3단계: dev를 취하여 evals 충돌 해결**

`evals`에서 리베이스가 중단되면 다음을 실행합니다:

```bash
git restore --source=origin/dev --staged --worktree evals
git add evals
git rebase --continue
```

예상: 리베이스가 계속 진행됨. 리베이스 후 `git diff --name-only origin/dev...HEAD -- evals` 실행 시 아무것도 출력되지 않음.

- [ ] **4단계: 베이스라인 상태 기록**

실행:

```bash
git status --short --branch
git diff --name-only origin/dev...HEAD -- evals
```

예상: status는 `origin/dev` 위의 브랜치를 표시함; 두 번째 명령어는 경로를 출력하지 않음.

## 작업 1: 루트 화면 격리 (Containment)

**파일:**
- 수정: `tests/brainstorm-server/server.test.js`
- 수정: `skills/brainstorming/scripts/server.cjs`

- [ ] **1단계: 고정 포트 가드 및 건너뛰기 지원 테스트 헬퍼 추가**

`tests/brainstorm-server/server.test.js`에서 `waitForServer()` 다음에 다음 헬퍼를 추가합니다:

```js
class SkipTest extends Error {
  constructor(message) {
    super(message);
    this.skip = true;
  }
}

function skip(message) {
  throw new SkipTest(message);
}

function serverStartedMessage(out) {
  const line = out.trim().split('\n').find(l => l.includes('server-started'));
  assert(line, 'server-started JSON should be present');
  return JSON.parse(line);
}

function assertStartedOnExpectedPort(out) {
  const msg = serverStartedMessage(out);
  assert.strictEqual(
    msg.port,
    TEST_PORT,
    `server.test.js expected fixed port ${TEST_PORT}, got ${msg.port}; fixed-port tests must not run through fallback`
  );
  return msg;
}

function ensureSymlinkWorks(target, link) {
  try {
    fs.symlinkSync(target, link);
    fs.unlinkSync(link);
  } catch (e) {
    try { fs.unlinkSync(link); } catch (ignore) {}
    skip(`symlink creation unavailable on this host: ${e.message}`);
  }
}
```

그 다음 시작 섹션을 다음에서:

```js
  const { stdout: initialStdout } = await waitForServer(server);
  let passed = 0;
  let failed = 0;
```

다음으로 변경합니다:

```js
  const { stdout: initialStdout } = await waitForServer(server);
  assertStartedOnExpectedPort(initialStdout);
  let passed = 0;
  let failed = 0;
  let skipped = 0;
```

건너뛰기를 처리하도록 `test()` 헬퍼 catch 블록을 변경합니다:

```js
    }).catch(e => {
      if (e && e.skip) {
        console.log(`  SKIP: ${name}`);
        console.log(`    ${e.message}`);
        skipped++;
        return;
      }
      console.log(`  FAIL: ${name}`);
      console.log(`    ${e.message}`);
      failed++;
    });
```

요약 줄을 다음으로 변경합니다:

```js
    console.log(`\n--- Results: ${passed} passed, ${failed} failed, ${skipped} skipped ---`);
```

- [ ] **2단계: 기존 `/files/*` 심볼릭 링크 테스트를 건너뛰기 가능하도록 설정**

`does not serve symlinks that escape content dir via /files/` 내부의 설정을 다음으로 교체합니다:

```js
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'linked-server-info.txt');
      try { fs.unlinkSync(link); } catch (e) {}
      ensureSymlinkWorks(target, link);
      fs.symlinkSync(target, link);
```

예상 동작: 사용 가능한 심볼릭 링크를 생성할 수 없는 호스트는 이 단정문만 건너뜁니다.

- [ ] **3단계: 루트 심볼릭 링크 및 하드 링크 탈출에 대한 RED 테스트 추가**

기존 `/files/*` 하드 링크 테스트 뒤에 다음 테스트를 추가합니다:

```js
    await test('does not serve symlinks that escape content dir via root screen selection', async () => {
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'root-linked-server-info.html');
      try { fs.unlinkSync(link); } catch (e) {}
      ensureSymlinkWorks(target, link);
      fs.symlinkSync(target, link);
      const future = new Date(Date.now() + 2000);
      fs.utimesSync(target, future, future);
      await sleep(300);

      const res = await fetch(`http://localhost:${TEST_PORT}/`);
      assert.strictEqual(res.status, 200);
      assert(!res.body.includes('"type":"server-started"'), 'root screen must not serve state/server-info through a symlink');
      assert(!res.body.includes('"state_dir"'), 'root screen must not include server-info body');
    });

    await test('does not serve hard links that escape content dir via root screen selection', async () => {
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'root-hard-linked-server-info.html');
      try { fs.unlinkSync(link); } catch (e) {}
      try {
        fs.linkSync(target, link);
      } catch (e) {
        skip(`hardlink creation unavailable on this host: ${e.message}`);
      }
      const linkStat = fs.lstatSync(link);
      if (linkStat.nlink <= 1) {
        skip(`hardlink nlink did not expose multiple links: ${linkStat.nlink}`);
      }
      const future = new Date(Date.now() + 3000);
      fs.utimesSync(target, future, future);
      await sleep(300);

      const res = await fetch(`http://localhost:${TEST_PORT}/`);
      assert.strictEqual(res.status, 200);
      assert(!res.body.includes('"type":"server-started"'), 'root screen must not serve state/server-info through a hardlink');
      assert(!res.body.includes('"state_dir"'), 'root screen must not include server-info body');
    });
```

- [ ] **4단계: RED 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node server.test.js
```

예상: 루트 화면 선택이 `state/server-info`를 읽을 수 있으므로 프로덕션 수정 전 최소 하나의 새 루트 격리 테스트가 실패함.

- [ ] **5단계: 루트 격리 구현**

`skills/brainstorming/scripts/server.cjs`에서 `getNewestScreen()`을 다음으로 교체합니다:

```js
function getNewestScreen() {
  const files = fs.readdirSync(CONTENT_DIR)
    .filter(f => !f.startsWith('.') && f.endsWith('.html'))
    .map(f => {
      const fp = path.join(CONTENT_DIR, f);
      if (!isRegularFileInsideContentDir(fp)) return null;
      return { path: fp, mtime: fs.statSync(fp).mtime.getTime() };
    })
    .filter(Boolean)
    .sort((a, b) => b.mtime - a.mtime);
  return files.length > 0 ? files[0].path : null;
}
```

- [ ] **6단계: GREEN 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node server.test.js
```

예상: 루트 심볼릭 링크 및 지원되는 하드 링크 테스트가 통과하거나 지원되지 않는 호스트 기능에 대해서만 건너뜁니다. 기존 `/files/*` 격리 테스트는 계속 성공 상태를 유지합니다.

- [ ] **7단계: 커밋**

실행:

```bash
git add tests/brainstorm-server/server.test.js skills/brainstorming/scripts/server.cjs
git commit -m "Harden root screen containment"
```

## 작업 2: 폴백 토큰 격리

**파일:**
- 수정: `tests/brainstorm-server/lifecycle.test.js`
- 수정: `skills/brainstorming/scripts/server.cjs`

- [ ] **1단계: HTTP 상태 헬퍼 추가**

`tests/brainstorm-server/lifecycle.test.js`에서 `openCaptureCommand()` 다음에 이 헬퍼를 추가합니다:

```js
function httpStatus(port, key) {
  return new Promise(resolve => {
    const pathWithKey = key ? '/?key=' + encodeURIComponent(key) : '/';
    require('http')
      .get({ hostname: '127.0.0.1', port, path: pathWithKey }, res => {
        res.resume();
        resolve(res.statusCode);
      })
      .on('error', () => resolve(0));
  });
}
```

- [ ] **2단계: 지속 토큰(persisted-token) 폴백 순환에 대한 RED 테스트 추가**

`falls back to a random port when the preferred port is taken` 다음에 이 테스트를 추가합니다:

```js
  await test('fallback with persisted token generates a fresh unpersisted key', async () => {
    const dir = fs.mkdtempSync('/tmp/bs-port-');
    const portFile = path.join(dir, '.last-port');
    const tokenFile = path.join(dir, '.last-token');
    const preferredToken = 'abababababababababababababababab';
    let a = null, b = null;

    try {
      a = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'a'),
          BRAINSTORM_PORT: 3422,
          BRAINSTORM_TOKEN: preferredToken,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outA = ''; a.stdout.on('data', d => outA += d.toString());
      for (let i = 0; i < 60 && !outA.includes('server-started'); i++) await sleep(50);
      assert(outA.includes('server-started'), 'preferred-port server should start');

      fs.writeFileSync(portFile, '3422');
      fs.writeFileSync(tokenFile, preferredToken, { mode: 0o600 });

      b = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'b'),
          BRAINSTORM_PORT_FILE: portFile,
          BRAINSTORM_TOKEN_FILE: tokenFile,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outB = ''; b.stdout.on('data', d => outB += d.toString());
      for (let i = 0; i < 60 && !outB.includes('server-started'); i++) await sleep(50);
      const infoB = firstServerStarted(outB);
      const fallbackKey = new URL(infoB.url).searchParams.get('key');
      const persistedAfter = fs.readFileSync(tokenFile, 'utf8').trim();
      const originalStatus = await httpStatus(3422, fallbackKey);

      assert.notStrictEqual(infoB.port, 3422, 'fallback should use a different port');
      assert.notStrictEqual(fallbackKey, preferredToken, 'fallback must not reuse persisted key');
      assert.strictEqual(persistedAfter, preferredToken, 'fallback must not overwrite .last-token');
      assert.strictEqual(originalStatus, 403, 'fallback key must not authenticate to original server');
    } finally {
      await killAndWait(a);
      await killAndWait(b);
      fs.rmSync(dir, { recursive: true, force: true });
    }
  });
```

- [ ] **3단계: 명시적 토큰(explicit-token) 폴백 닫힌 실패(fail-closed)에 대한 RED 테스트 추가**

지속 토큰 폴백 테스트 바로 뒤에 다음 테스트를 추가합니다:

```js
  await test('fallback with explicit BRAINSTORM_TOKEN fails closed', async () => {
    const dir = fs.mkdtempSync('/tmp/bs-port-');
    const portFile = path.join(dir, '.last-port');
    const explicitToken = 'cdcdcdcdcdcdcdcdcdcdcdcdcdcdcdcd';
    let a = null, b = null;

    try {
      a = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'a'),
          BRAINSTORM_PORT: 3423,
          BRAINSTORM_TOKEN: explicitToken,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outA = ''; a.stdout.on('data', d => outA += d.toString());
      for (let i = 0; i < 60 && !outA.includes('server-started'); i++) await sleep(50);
      assert(outA.includes('server-started'), 'preferred-port server should start');

      fs.writeFileSync(portFile, '3423');
      b = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'b'),
          BRAINSTORM_PORT_FILE: portFile,
          BRAINSTORM_TOKEN: explicitToken,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outB = ''; let errB = '';
      b.stdout.on('data', d => outB += d.toString());
      b.stderr.on('data', d => errB += d.toString());
      for (let i = 0; i < 60 && !outB.includes('server-started') && b.exitCode === null; i++) await sleep(50);
      const exited = await waitForExit(b, 1500);

      assert(exited, 'explicit-token fallback process should exit');
      assert.notStrictEqual(b.exitCode, 0, 'explicit-token fallback should fail non-zero');
      assert(!outB.includes('server-started'), 'explicit-token fallback must not start on a random port');
      assert(/BRAINSTORM_TOKEN/.test(errB), `stderr should explain explicit token fallback refusal, got: ${errB}`);
    } finally {
      await killAndWait(a);
      await killAndWait(b);
      fs.rmSync(dir, { recursive: true, force: true });
    }
  });
```

- [ ] **4단계: RED 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node lifecycle.test.js
```

예상: 지속 토큰 폴백 테스트는 폴백이 `.last-token`을 재사용하므로 실패하고, 명시적 토큰 폴백 테스트는 현재 폴백이 시작되므로 실패함.

- [ ] **5단계: 프로덕션 코드에서 토큰 소스 추적**

`skills/brainstorming/scripts/server.cjs`에서 기존 `const TOKEN = (() => { ... })();` 블록을 다음으로 교체합니다:

```js
function generateToken() {
  return crypto.randomBytes(32).toString('hex');
}

function initialToken() {
  if (process.env.BRAINSTORM_TOKEN) {
    return { value: process.env.BRAINSTORM_TOKEN, source: 'env' };
  }
  if (TOKEN_FILE) {
    try {
      const t = fs.readFileSync(TOKEN_FILE, 'utf-8').trim();
      if (/^[0-9a-f]{32,}$/i.test(t)) return { value: t, source: 'file' };
    } catch (e) { /* no prior token recorded */ }
  }
  return { value: generateToken(), source: 'generated' };
}

const tokenInfo = initialToken();
let TOKEN = tokenInfo.value;
let tokenSource = tokenInfo.source;
```

- [ ] **6단계: EADDRINUSE 폴백 시 순환(rotate)하거나 닫힌 상태로 실패(fail closed)**

`server.on('error', ...)` 핸들러에서 `EADDRINUSE` 분기를 다음으로 교체합니다:

```js
    if (err.code === 'EADDRINUSE' && !triedFallback) {
      if (tokenSource === 'env') {
        console.error('Server failed to bind: preferred port is in use and BRAINSTORM_TOKEN is set; refusing fallback with explicit token');
        process.exit(1);
      }
      triedFallback = true;
      PORT = randomPort();
      if (tokenSource === 'file') {
        TOKEN = generateToken();
        tokenSource = 'generated-fallback';
      }
      server.listen(PORT, HOST, onListen);
    } else {
```

- [ ] **7단계: GREEN 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node lifecycle.test.js
```

예상: 폴백 토큰 순환 및 명시적 토큰 닫힌 실패를 포함하여 모든 라이프사이클 테스트가 통과함.

- [ ] **8단계: 커밋**

실행:

```bash
git add tests/brainstorm-server/lifecycle.test.js skills/brainstorming/scripts/server.cjs
git commit -m "Isolate companion fallback tokens"
```

## 작업 3: stop-server 인스턴스 ID 소유권

**파일:**
- 수정: `tests/brainstorm-server/stop-server.test.sh`
- 수정: `skills/brainstorming/scripts/start-server.sh`
- 수정: `skills/brainstorming/scripts/stop-server.sh`

- [ ] **1단계: stop-server 테스트에 정리 추적 및 id 헬퍼 추가**

`tests/brainstorm-server/stop-server.test.sh`에서 `PASS=0; FAIL=0` 다음에 다음을 추가합니다:

```bash
PIDS=()
DIRS=()

cleanup() {
  for pid in "${PIDS[@]}"; do
    kill -9 "$pid" 2>/dev/null || true
    wait "$pid" 2>/dev/null || true
  done
  for dir in "${DIRS[@]}"; do
    rm -rf "$dir"
  done
}
trap cleanup EXIT

track_dir() { DIRS+=("$1"); }
track_pid() { PIDS+=("$1"); }
new_server_id() {
  printf 'testid%026d\n' "$RANDOM"
}
```

각 테스트가 `SESS="$(mktemp -d)"`를 생성할 때 바로 다음을 추가합니다:

```bash
track_dir "$SESS"
```

테스트가 `UNRELATED`, `SRV`, 또는 `IMPOSTOR`를 시작할 때 즉시 상응하는 추적 호출을 추가합니다:

```bash
track_pid "$UNRELATED"
track_pid "$SRV"
track_pid "$IMPOSTOR"
```

- [ ] **2단계: RED 소유권 테스트 추가**

현재 실제 서버 및 가장(impostor) 섹션을 다음 케이스들로 교체합니다:

```bash
# --- Test 2: a real brainstorm server with matching instance id IS stopped ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/content" "$SESS/state"
SERVER_ID="$(new_server_id)"
printf '%s\n' "$SERVER_ID" > "$SESS/state/server-instance-id"
BRAINSTORM_DIR="$SESS" BRAINSTORM_PORT=3399 node "$SERVER" "--brainstorm-server-id=$SERVER_ID" > /dev/null 2>&1 &
SRV=$!
track_pid "$SRV"
disown "$SRV" 2>/dev/null || true
for _ in $(seq 1 40); do kill -0 "$SRV" 2>/dev/null && break; sleep 0.1; done
sleep 0.4
echo "$SRV" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
sleep 0.3
if kill -0 "$SRV" 2>/dev/null; then
  bad "real brainstorm server still running after stop" "$OUT"
else
  case "$OUT" in
    *stopped*) ok "real brainstorm server with matching instance id is stopped" ;;
    *) bad "server stopped but status was not 'stopped'" "$OUT" ;;
  esac
fi

# --- Test 4: a node server.cjs impostor with missing instance id is spared ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/state"
( exec -a "node server.cjs" sleep 600 ) &
IMPOSTOR=$!
track_pid "$IMPOSTOR"
disown "$IMPOSTOR" 2>/dev/null || true
echo "$IMPOSTOR" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
if kill -0 "$IMPOSTOR" 2>/dev/null; then
  case "$OUT" in
    *stale_pid*) ok "missing instance id leaves node server.cjs impostor alone" ;;
    *) bad "impostor survived but status was not stale_pid" "$OUT" ;;
  esac
else
  bad "killed a node server.cjs impostor with missing instance id" "$OUT"
fi

# --- Test 5: a node server.cjs impostor with wrong instance id is spared ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/state"
EXPECTED_ID="$(new_server_id)"
WRONG_ID="$(new_server_id)"
printf '%s\n' "$EXPECTED_ID" > "$SESS/state/server-instance-id"
( exec -a "node server.cjs --brainstorm-server-id=$WRONG_ID" sleep 600 ) &
IMPOSTOR=$!
track_pid "$IMPOSTOR"
disown "$IMPOSTOR" 2>/dev/null || true
echo "$IMPOSTOR" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
if kill -0 "$IMPOSTOR" 2>/dev/null; then
  case "$OUT" in
    *stale_pid*) ok "wrong instance id leaves node server.cjs impostor alone" ;;
    *) bad "wrong-id impostor survived but status was not stale_pid" "$OUT" ;;
  esac
else
  bad "killed a node server.cjs impostor with wrong instance id" "$OUT"
fi

# --- Test 6: malformed instance id is fail-closed ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/state"
printf '%s\n' 'bad id with spaces' > "$SESS/state/server-instance-id"
( exec -a "node server.cjs --brainstorm-server-id=bad-id-with-spaces" sleep 600 ) &
IMPOSTOR=$!
track_pid "$IMPOSTOR"
disown "$IMPOSTOR" 2>/dev/null || true
echo "$IMPOSTOR" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
if kill -0 "$IMPOSTOR" 2>/dev/null; then
  case "$OUT" in
    *stale_pid*) ok "malformed instance id is fail-closed" ;;
    *) bad "malformed-id impostor survived but status was not stale_pid" "$OUT" ;;
  esac
else
  bad "killed process despite malformed instance id" "$OUT"
fi
```

관련 없는 PID 및 누락된 PID 테스트는 유지합니다.

- [ ] **3단계: RED 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers
bash tests/brainstorm-server/stop-server.test.sh
```

예상: 일치하는 instance-id를 가진 실제 서버가 구현 전에는 `stale_pid`로 보고되고, 가장(impostor) 케이스 중 하나는 기존 명령어 이름 증명에 의해 종료될 수 있음.

- [ ] **4단계: start-server에서 인스턴스 ID 생성 및 전달**

`skills/brainstorming/scripts/start-server.sh`에서 `LOG_FILE="${STATE_DIR}/server.log"` 다음에 다음을 추가합니다:

```bash
SERVER_ID_FILE="${STATE_DIR}/server-instance-id"
```

`mkdir -p "${SESSION_DIR}/content" "$STATE_DIR"` 다음에 다음을 추가합니다:

```bash
SERVER_ID=""
if [[ -r /dev/urandom ]]; then
  SERVER_ID="$(od -An -N24 -tx1 /dev/urandom 2>/dev/null | tr -d ' \n' || true)"
fi
if ! [[ "$SERVER_ID" =~ ^[A-Za-z0-9_-]{32,64}$ ]]; then
  SERVER_ID="$(printf '%08x%08x%08x%08x' "$$" "$(date +%s)" "${RANDOM:-0}" "${RANDOM:-0}")"
fi
printf '%s\n' "$SERVER_ID" > "$SERVER_ID_FILE"
chmod 600 "$SERVER_ID_FILE" 2>/dev/null || true
```

두 Node 실행 명령어 모두 argv를 전달하도록 업데이트합니다:

```bash
env BRAINSTORM_DIR="$SESSION_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" BRAINSTORM_OWNER_PID="$OWNER_PID" node server.cjs "--brainstorm-server-id=$SERVER_ID" &
```

그리고:

```bash
nohup env BRAINSTORM_DIR="$SESSION_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" BRAINSTORM_OWNER_PID="$OWNER_PID" node server.cjs "--brainstorm-server-id=$SERVER_ID" > "$LOG_FILE" 2>&1 &
```

- [ ] **5단계: stop-server에서 인스턴스 ID 필수화**

`skills/brainstorming/scripts/stop-server.sh`에 다음을 추가합니다:

```bash
SERVER_ID_FILE="${STATE_DIR}/server-instance-id"
```

`is_brainstorm_server()`를 다음으로 교체합니다:

```bash
read_expected_server_id() {
  [[ -f "$SERVER_ID_FILE" ]] || return 1
  local id
  id="$(tr -d '\r\n' < "$SERVER_ID_FILE" 2>/dev/null || true)"
  [[ "$id" =~ ^[A-Za-z0-9_-]{32,64}$ ]] || return 1
  printf '%s\n' "$id"
}

command_line_for_pid() {
  local pid="$1"
  if [[ -r "/proc/$pid/cmdline" ]]; then
    tr '\0' '\n' < "/proc/$pid/cmdline" 2>/dev/null || true
    return 0
  fi
  ps -ww -p "$pid" -o command= 2>/dev/null || ps -f -p "$pid" 2>/dev/null | sed '1d' || true
}

command_has_server_id() {
  local pid="$1"
  local expected="$2"
  local expected_arg="--brainstorm-server-id=$expected"
  if [[ -r "/proc/$pid/cmdline" ]]; then
    local arg
    while IFS= read -r -d '' arg; do
      [[ "$arg" == "$expected_arg" ]] && return 0
    done < "/proc/$pid/cmdline"
    return 1
  fi
  local command_line
  command_line="$(command_line_for_pid "$pid")"
  [[ -n "$command_line" ]] || return 1
  case " $command_line " in
    *" $expected_arg "*) return 0 ;;
    *) return 1 ;;
  esac
}

is_brainstorm_server() {
  kill -0 "$1" 2>/dev/null || return 1
  local expected_id
  expected_id="$(read_expected_server_id)" || return 1
  command_has_server_id "$1" "$expected_id" || return 1
  return 0
}
```

오래된 PID 분기에서 두 메타데이터 파일을 모두 제거합니다:

```bash
    rm -f "$PID_FILE" "$SERVER_ID_FILE"
```

종료된 분기에서 정리 줄을 다음으로 변경합니다:

```bash
  rm -f "$PID_FILE" "$SERVER_ID_FILE" "${STATE_DIR}/server.log"
```

- [ ] **6단계: GREEN 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers
bash tests/brainstorm-server/stop-server.test.sh
```

예상: 일치하는 ID의 실제 서버는 중지되고 가장(impostor)은 생존하며, 모든 오래된 케이스는 `stale_pid`를 반환함.

- [ ] **7단계: 커밋**

실행:

```bash
git add tests/brainstorm-server/stop-server.test.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh
git commit -m "Harden companion stop ownership proof"
```

## 작업 4: 플랫폼 및 고정 포트 테스트 강화

**파일:**
- 수정: `tests/brainstorm-server/auth.test.js`
- 수정: `tests/brainstorm-server/start-server.test.sh`
- 수정: `tests/brainstorm-server/windows-lifecycle.test.sh`

- [ ] **1단계: auth 테스트에 고정 포트 가드 추가**

`tests/brainstorm-server/auth.test.js`에서 `waitForServer()` 다음에 다음 헬퍼를 추가합니다:

```js
function serverStartedMessage(out) {
  const line = out.trim().split('\n').find(l => l.includes('server-started'));
  assert(line, 'server-started JSON should be present');
  return JSON.parse(line);
}

function assertStartedOnExpectedPort(out) {
  const msg = serverStartedMessage(out);
  assert.strictEqual(
    msg.port,
    TEST_PORT,
    `auth.test.js expected fixed port ${TEST_PORT}, got ${msg.port}; fixed-port tests must not run through fallback`
  );
  return msg;
}
```

`const { stdout: initialStdout } = await waitForServer(server);` 다음에 다음을 추가합니다:

```js
  assertStartedOnExpectedPort(initialStdout);
```

- [ ] **2단계: auth 고정 포트 가드 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node auth.test.js
```

예상: 자유 포트 `3335`에서 auth 테스트가 통과하며, 폴백 발생 시 명확하게 실패함.

- [ ] **3단계: start-server ID argv 단정문 추가**

`tests/brainstorm-server/start-server.test.sh`에서 첫 번째 가짜 node 본문을 다음으로 변경합니다:

```bash
cat > "$TEST_DIR/fake-bin/node" <<'EOF'
#!/usr/bin/env bash
echo "CAPTURED_OWNER_PID=${BRAINSTORM_OWNER_PID:-__UNSET__}"
echo "CAPTURED_ARGV=$*"
exit 0
EOF
```

소유자 PID 단정문 다음에 다음을 추가합니다:

```bash
captured_argv=$(echo "$captured" | grep "CAPTURED_ARGV=" | head -1 | sed 's/CAPTURED_ARGV=//')
if echo "$captured_argv" | grep -Eq -- '--brainstorm-server-id=[A-Za-z0-9_-]{32,64}'; then
  pass "passes shell-safe server instance id argv"
else
  fail "passes shell-safe server instance id argv" \
       "expected --brainstorm-server-id=<safe id>, got: $captured_argv"
fi

server_id_file=$(find "$TEST_DIR/project/.superpowers/brainstorm" -name server-instance-id -print 2>/dev/null | head -1)
server_id_value=""
if [[ -n "$server_id_file" ]]; then
  server_id_value="$(tr -d '\r\n' < "$server_id_file")"
fi
if [[ "$server_id_value" =~ ^[A-Za-z0-9_-]{32,64}$ ]]; then
  pass "writes shell-safe server-instance-id state file"
else
  fail "writes shell-safe server-instance-id state file" \
       "expected valid id in state, got '$server_id_value'"
fi
```

- [ ] **4단계: Windows 라이프사이클 ID argv 단정문 추가**

`tests/brainstorm-server/windows-lifecycle.test.sh`에서 테스트 2의 가짜 node 본문을 다음으로 변경합니다:

```bash
cat > "$FAKE_NODE_DIR/node" <<'FAKENODE'
#!/usr/bin/env bash
echo "CAPTURED_OWNER_PID=${BRAINSTORM_OWNER_PID:-__UNSET__}"
echo "CAPTURED_ARGV=$*"
exit 0
FAKENODE
```

테스트 2의 소유자 PID 검사 다음에 다음을 추가합니다:

```bash
captured_argv=$(echo "$captured" | grep "CAPTURED_ARGV=" | head -1 | sed 's/CAPTURED_ARGV=//')
if echo "$captured_argv" | grep -Eq -- '--brainstorm-server-id=[A-Za-z0-9_-]{32,64}'; then
  pass "start-server.sh passes server instance id argv on Windows"
else
  fail "start-server.sh passes server instance id argv on Windows" \
       "Expected --brainstorm-server-id=<safe id>, output: $captured"
fi
```

테스트 6에서 직접 Node를 실행하기 전에 다음을 추가합니다:

```bash
STOP_TEST_ID="$(printf 'windowsstop%021d\n' "$RANDOM")"
printf '%s\n' "$STOP_TEST_ID" > "$TEST_DIR/stop-test/state/server-instance-id"
```

테스트 6의 직접 Node 실행을 다음으로 변경합니다:

```bash
  node "$SERVER_SCRIPT" "--brainstorm-server-id=$STOP_TEST_ID" > "$TEST_DIR/stop-test/.server.log" 2>&1 &
```

- [ ] **5단계: 플랫폼 테스트 검증**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers
bash tests/brainstorm-server/start-server.test.sh
```

예상: macOS에서 모든 start-server 셸 테스트 통과.

Windows 라이프사이클 테스트는 추후 작업 6의 일환으로 `ballmer`에서 실행합니다.

- [ ] **6단계: 커밋**

실행:

```bash
git add tests/brainstorm-server/auth.test.js tests/brainstorm-server/start-server.test.sh tests/brainstorm-server/windows-lifecycle.test.sh
git commit -m "Harden companion platform tests"
```

## 작업 5: 문서 및 PR 일관성

**파일:**
- 수정: `skills/brainstorming/visual-companion.md`
- 수정: `docs/superpowers/plans/2026-06-09-visual-companion-issues.md`
- 업데이트: `gh pr edit`를 통해 PR #1720 본문 업데이트

- [ ] **1단계: 플랫폼 시작 명령을 자동 열기 동작에 맞게 유지**

`skills/brainstorming/visual-companion.md`에서 사용자가 승인한 컴패니언 세션을 시작하는 플랫폼별 명령에 `--open`이 포함되도록 업데이트합니다:

```bash
scripts/start-server.sh --project-dir /path/to/project --open
```

```bash
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

의도적으로 자동 열기를 건너뛰는 원격 바인드 예시에는 `--open`을 추가하지 마세요.

- [ ] **2단계: 이슈 카탈로그 조치(disposition) 행 조율**

`docs/superpowers/plans/2026-06-09-visual-companion-issues.md`에서 A2, D1, D2, D3, D4에 대한 조치 행을 다음으로 교체합니다:

```markdown
| A2 | Host allowlist; browser WS Origin check | PRs #1110/#1553 | Host allowlist dropped; WS Origin check retained after auth for browser confused-deputy defense |
| D1 | Permanent opt-out of the companion | issue #892 | Deferred - not in PR #1720 |
| D2 | Free-text feedback from the browser | issue #957 | Deferred - not in PR #1720 |
| D3 | Auto-open the companion URL | PR #759 (#755) | Done in PR #1720 via `--open` |
| D4 | Light/dark contrast helpers in the frame | PR #1683 | Deferred - not in PR #1720 |
```

- [ ] **3단계: A2 세부 텍스트 조율**

A2 섹션의 마지막 문장을 다음으로 교체합니다:

```markdown
No `BRAINSTORM_ALLOWED_HOSTS` and no Host allowlist. The final implementation still checks browser WebSocket `Origin` after session auth so a cross-origin localhost tab cannot ride the companion cookie.
```

- [ ] **4단계: 타임아웃 및 기능 그룹화 텍스트 조율**

C1 섹션에서 다음을:

```markdown
- Raise the default (about 2h) and make it configurable:
```

다음으로 교체합니다:

```markdown
- Raise the default to 4 hours and make it configurable:
```

제안된 그룹화 섹션에서 4번 항목을 다음으로 교체합니다:

```markdown
4. **Deferred feature pass** - D1, D2, D4 are not part of PR #1720. D3 is shipped through the `--open` flow.
```

- [ ] **5단계: 문서 diff 검증**

실행:

```bash
git diff -- skills/brainstorming/visual-companion.md docs/superpowers/plans/2026-06-09-visual-companion-issues.md
```

예상: diff가 자동 열기 명령 일관성, 출시/지연 조치, WS Origin 어휘 및 4시간 타임아웃 진술만 업데이트함.

- [ ] **6단계: 커밋**

실행:

```bash
git add skills/brainstorming/visual-companion.md docs/superpowers/plans/2026-06-09-visual-companion-issues.md
git commit -m "Align visual companion docs with shipped scope"
```

## 작업 6: 전체 검증 및 증거 자료

**파일:**
- 소스 수정 요구 없음
- 업데이트: PR #1720 본문

- [ ] **1단계: 집중적인 macOS 검사 실행**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node server.test.js
node auth.test.js
node lifecycle.test.js
bash stop-server.test.sh
bash start-server.test.sh
```

예상: 집중 테스트 모두 통과; 심볼릭 링크 전용 테스트는 호스트 지원을 사용할 수 없을 때만 건너뜀으로 보고될 수 있음.

- [ ] **2단계: 전체 macOS 테스트 스위트 실행**

실행:

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
npm test
```

예상: 전체 brainstorm-server 테스트 스위트 통과.

- [ ] **3단계: 정적 검사 실행**

저장소 루트에서 실행:

```bash
git diff --check
node --check skills/brainstorming/scripts/server.cjs
node --check skills/brainstorming/scripts/helper.js
bash scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/start-server.test.sh tests/brainstorm-server/stop-server.test.sh tests/brainstorm-server/windows-lifecycle.test.sh
```

예상: 모든 명령 종료 코드 0.

- [ ] **4단계: ballmer에서 Windows 검증 실행**

`ballmer`에서 리베이스된 브랜치를 복사하거나 가져온 후 실행합니다:

```bash
cd superpowers
npm --prefix tests/brainstorm-server ci
npm --prefix tests/brainstorm-server test
bash tests/brainstorm-server/windows-lifecycle.test.sh
```

예상: 실행 가능한 전체 Windows 스위트 통과. Git Bash에 `lsof`가 없는 경우 lsof 전용 레거시 포트 교차 검사 테스트만 건너뛸 수 있으며, instance-id 중지 테스트는 여전히 통과해야 합니다.

- [ ] **5단계: PR diff 및 GitHub 상태 검증**

실행:

```bash
git diff --quiet origin/dev...HEAD -- evals
gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid
```

예상: 첫 번째 명령 종료 코드 0. 브랜치가 푸시된 후 PR JSON에 더 이상 `DIRTY`나 `CONFLICTING`이 보고되지 않음.

- [ ] **6단계: 외부 평가 증거 수집**

실행:

```bash
git -C /Users/drewritter/.codex/worktrees/59f6/superpowers-evals rev-parse HEAD
git -C /Users/drewritter/.codex/worktrees/59f6/superpowers-evals status --short --branch
```

평가 워크트리가 해당 경로에 없는 경우 `/Users/drewritter/prime-rad/superpowers-evals`에서 동일한 명령을 실행합니다.

이미 실행된 평가 증거에서 정확한 평가 시나리오 경로, 명령, 결과 아티팩트 경로 및 RED/GREEN 결과를 기록합니다. 이 PR #1720 diff에 eval 서브모듈이 포함되어 있다고 주장하지 마세요.

- [ ] **7단계: 최종 수동/브라우저 스모크 실행**

자동화된 테스트가 성공(green)으로 나오면 `--open`을 사용하여 컴패니언을 시작하고, 작은 화면을 푸시하고, 브라우저가 부트스트랩 후 순수 `/` URL에 도달하는지 확인하고, 상태가 Connected에 도달하는지 확인하고, 동일한 프로젝트 디렉터리로 서버를 중지 후 재시작하고, 열려 있는 탭이 재연결되는지 확인합니다. 정확한 명령어와 관찰된 결과를 기록합니다.

- [ ] **8단계: PR 본문 업데이트**

`/tmp/pr-1720-body.md`를 준비한 다음 본문에 아래 내용이 포함된 후 `gh pr edit 1720 --body-file /tmp/pr-1720-body.md`를 실행합니다:

- 모델, 하네스, 플러그인 및 사람 리뷰어 Drew
- 중복/관련 PR 검색 결과
- 이 PR diff에 `evals`가 없음을 나타내는 정확한 리베이스 후 노트
- 집중 RED/GREEN 증거 표
- macOS `npm test` 증거
- Windows `ballmer` 증거
- 수동/브라우저 스모크 증거
- 외부 eval 저장소 커밋, 시나리오 경로, 명령, 아티팩트 경로 및 결과

- [ ] **9단계: 브랜치 푸시**

실행:

```bash
git status --short --branch
git push origin brainstorming-companion
```

예상: 푸시 성공 및 PR #1720 업데이트.

- [ ] **10단계: 최종 PR 준비 상태 검사**

실행:

```bash
gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid,url
```

예상: PR이 푸시된 HEAD SHA를 가리키고, 병합 상태가 더 이상 충돌에 의해 차단되지 않으며, Drew에 대한 검사 상태가 기록됨.

## 자체 검토 체크리스트

- [ ] `docs/superpowers/specs/2026-06-11-visual-companion-final-hardening-fixup-design.md`의 모든 요구 사항이 위의 작업 중 하나에 매핑됩니다.
- [ ] 계획에 모호하거나 불완전한 단계가 포함되어 있지 않습니다.
- [ ] 작업 1, 2, 3에서는 프로덕션 수정 전에 테스트가 추가됩니다.
- [ ] 문서 작업에 지연된 기능이 추가되지 않습니다.
- [ ] 검증 작업에는 macOS, Windows, PR diff, PR 메타데이터, 외부 평가 증거 및 최종 수동/브라우저 스모크 테스트가 포함됩니다.
