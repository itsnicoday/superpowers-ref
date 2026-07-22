# 시각적 브레인스토밍 리팩토링 구현 계획

> **에이전트 작업자 참고:** 필수: 이 계획을 구현하려면 superpowers:subagent-driven-development (서브에이전트가 사용 가능한 경우) 또는 superpowers:executing-plans를 사용하세요. 추적을 위해 단계에 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** 시각적 브레인스토밍을 블로킹 TUI 피드백 모델에서 비블로킹 "브라우저 디스플레이, 터미널 명령어" 아키텍처로 리팩토링합니다.

**아키텍처:** 브라우저는 상호작용 가능한 디스플레이가 되고, 터미널은 대화 채널로 유지됩니다. 서버는 사용자 이벤트를 화면별 `.events` 파일에 작성하며, Claude는 다음 차례에 이를 읽습니다. `wait-for-feedback.sh` 및 모든 `TaskOutput` 블로킹을 제거합니다.

**기술 스택:** Node.js (Express, ws, chokidar), 바닐라 HTML/CSS/JS

**명세:** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 파일 맵

| 파일 | 작업 | 역할 |
|------|--------|---------------|
| `lib/brainstorm-server/index.js` | 수정 | 서버: `.events` 파일 작성 추가, 새 화면 시 초기화, `wrapInFrame` 교체 |
| `lib/brainstorm-server/frame-template.html` | 수정 | 템플릿: 피드백 푸터 제거, 콘텐츠 자리표시자 + 선택 표시기 추가 |
| `lib/brainstorm-server/helper.js` | 수정 | 클라이언트 JS: 전송/피드백 함수 제거, 클릭 캡처 + 표시기 업데이트로 축소 |
| `lib/brainstorm-server/wait-for-feedback.sh` | 삭제 | 더 이상 필요 없음 |
| `skills/brainstorming/visual-companion.md` | 수정 | 스킬 지침: 루프를 비블로킹 흐름으로 재작성 |
| `tests/brainstorm-server/server.test.js` | 수정 | 테스트: 새 템플릿 구조 및 helper.js API에 맞게 업데이트 |

---

## 청크 1: 서버, 템플릿, 클라이언트, 테스트, 스킬

### 작업 1: `frame-template.html` 업데이트

**파일:**
- 수정: `lib/brainstorm-server/frame-template.html`

- [ ] **단계 1: 피드백 푸터 HTML 제거**

feedback-footer div(227-233줄)를 선택 표시기 바로 교체합니다:

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above, then return to the terminal</span>
  </div>
```

또한 `#claude-content` 내부의 기본 콘텐츠(220-223줄)를 콘텐츠 자리표시자로 교체합니다:

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **단계 2: 피드백 푸터 CSS를 표시기 바 CSS로 교체**

`.feedback-footer`, `.feedback-footer label`, `.feedback-row` 및 `.feedback-footer` 내부의 textarea/button 스타일(82-112줄)을 제거합니다.

표시기 바 CSS를 추가합니다:

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **단계 3: 템플릿 렌더링 검증**

템플릿이 여전히 로드되는지 확인하기 위해 테스트 수트를 실행합니다:
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 테스트 1-5는 여전히 통과해야 합니다. 테스트 6-8은 실패할 수 있습니다 (이전 구조를 단정하므로 예상됨).

- [ ] **단계 4: 커밋**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### 작업 2: `index.js` 업데이트 — 콘텐츠 주입 및 `.events` 파일

**파일:**
- 수정: `lib/brainstorm-server/index.js`

- [ ] **단계 1: `.events` 파일 작성에 대한 실패하는 테스트 작성**

Test 4 영역 뒤의 `tests/brainstorm-server/server.test.js`에 `choice` 필드가 포함된 WebSocket 이벤트를 보내고 `.events` 파일이 작성되는지 검증하는 새 테스트를 추가합니다:

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **단계 2: 테스트를 실행하여 실패하는지 확인**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 새 테스트 실패 — `.events` 파일이 아직 존재하지 않음.

- [ ] **단계 3: 새 화면에서 `.events` 파일 초기화에 대한 실패하는 테스트 작성**

또 다른 테스트를 추가합니다:

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **단계 4: 테스트를 실행하여 실패하는지 확인**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 새 테스트 실패 — 화면 푸시 시 `.events`가 초기화되지 않음.

- [ ] **단계 5: `index.js`에 `.events` 파일 작성 구현**

WebSocket `message` 핸들러(`index.js` 74-77줄)의 `console.log` 뒤에 다음을 추가합니다:

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

chokidar `add` 핸들러(104-111줄)에 `.events` 초기화를 추가합니다:

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **단계 6: `wrapInFrame`을 주석 자리표시자 주입으로 교체**

`wrapInFrame` 함수(`index.js` 27-32줄)를 교체합니다:

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **단계 7: 모든 테스트 실행**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 새로운 `.events` 테스트 통과. 기존 테스트에는 이전 단정문으로 인한 실패가 여전히 있을 수 있음 (작업 4에서 수정됨).

- [ ] **단계 8: 커밋**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### 작업 3: `helper.js` 단순화

**파일:**
- 수정: `lib/brainstorm-server/helper.js`

- [ ] **단계 1: `sendToClaude` 함수 제거**

`sendToClaude` 함수(92-106줄)를 삭제합니다 — 함수 본문 및 페이지 인수(takeover) HTML.

- [ ] **단계 2: `window.send` 함수 제거**

`window.send` 함수(120-129줄)를 삭제합니다 — 제거된 Send 버튼과 연결되어 있었음.

- [ ] **단계 3: 폼 제출 및 입력 변경 핸들러 제거**

폼 제출 핸들러(57-71줄) 및 `inputTimeout` 변수를 포함한 입력 변경 핸들러(73-89줄)를 삭제합니다.

- [ ] **단계 4: `pageshow` 이벤트 리스너 제거**

이전에 추가한 `pageshow` 리스너를 삭제합니다 (더 이상 초기화할 textarea가 없음).

- [ ] **단계 5: 클릭 핸들러를 `[data-choice]`로만 축소**

클릭 핸들러(36-55줄)를 더 좁은 버전으로 교체합니다:

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **단계 6: 선택 클릭 시 표시기 바 업데이트 추가**

클릭 핸들러의 `sendEvent` 호출 뒤에 다음을 추가합니다:

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **단계 7: `window.brainstorm` API에서 `sendToClaude` 제거**

`window.brainstorm` 객체(132-136줄)를 업데이트하여 `sendToClaude`를 제거합니다:

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **단계 8: 테스트 실행**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **단계 9: 커밋**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### 작업 4: 새 구조에 맞게 테스트 업데이트

**파일:**
- 수정: `tests/brainstorm-server/server.test.js`

**참고:** 아래 줄 번호는 _원본_ 파일 기준입니다. 작업 2에서 파일 앞부분에 새 테스트를 삽입했으므로 실제 줄 번호는 이동됩니다. `console.log` 라벨(예: "Test 5:", "Test 6:")로 테스트를 찾으세요.

- [ ] **단계 1: Test 5 (전체 문서 단정문) 업데이트**

Test 5 단정문 `!fullRes.body.includes('feedback-footer')`를 찾습니다. 다음과 같이 변경합니다: 전체 문서에는 표시기 바도 없어야 함 (있는 그대로 제공됨):

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **단계 2: Test 6 (조각 래핑) 업데이트**

125줄: `feedback-footer` 단정문을 표시기 바 단정문으로 교체합니다:

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

또한 콘텐츠 자리표시자가 교체되었는지 검증합니다 (조각 콘텐츠는 나타나고, 자리표시자 주석은 나타나지 않음):

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **단계 3: Test 7 (helper.js API) 업데이트**

140-142줄: 새 API 표면을 반영하도록 단정문을 업데이트합니다:

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **단계 4: Test 8 (sendToClaude 테마 설정)을 표시기 바 테스트로 교체**

Test 8(145-149줄)을 교체합니다 — `sendToClaude`는 더 이상 존재하지 않습니다. 대신 표시기 바를 테스트합니다:

```javascript
    // Test 8: Indicator bar uses CSS variables (theme support)
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **단계 5: 전체 테스트 수트 실행**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 모든 테스트 통과.

- [ ] **단계 6: 커밋**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### 작업 5: `wait-for-feedback.sh` 삭제

**파일:**
- 삭제: `lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **단계 1: 다른 파일이 `wait-for-feedback.sh`를 임포트하거나 참조하지 않는지 검증**

코드베이스를 검색합니다:
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```
예상되는 참조: `visual-companion.md`(작업 6에서 재작성됨) 및 히스토리성 릴리스 노트(있는 그대로 둠)뿐이어야 함.

- [ ] **단계 2: 파일 삭제**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **단계 3: 아무것도 깨지지 않는지 확인하기 위해 테스트 실행**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 모든 테스트 통과 (이 파일을 참조하는 테스트가 없음).

- [ ] **단계 4: 커밋**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### 작업 6: `visual-companion.md` 재작성

**파일:**
- 수정: `skills/brainstorming/visual-companion.md`

- [ ] **단계 1: "작동 방식" 설명 업데이트 (18줄)**

피드백을 "JSON으로" 수신한다는 문장을 다음으로 교체합니다:

```markdown
The server watches a directory for HTML files and serves the newest one to the browser. You write HTML content, the user sees it in their browser and can click to select options. Selections are recorded to a `.events` file that you read on your next turn.
```

- [ ] **단계 2: 조각 설명 업데이트 (20줄)**

프레임 템플릿이 제공하는 기능 설명에서 "피드백 푸터"를 제거합니다:

```markdown
**Content fragments vs full documents:** If your HTML file starts with `<!DOCTYPE` or `<html`, the server serves it as-is (just injects the helper script). Otherwise, the server automatically wraps your content in the frame template — adding the header, CSS theme, selection indicator, and all interactive infrastructure. **Write content fragments by default.** Only write full documents when you need complete control over the page.
```

- [ ] **단계 3: "루프" 섹션 재작성 (36-61줄)**

"루프" 섹션 전체를 다음으로 교체합니다:

```markdown
## The Loop

1. **Write HTML** to a new file in `screen_dir`:
   - Use semantic filenames: `platform.html`, `visual-style.html`, `layout.html`
   - **Never reuse filenames** — each screen gets a fresh file
   - Use Write tool — **never use cat/heredoc** (dumps noise into terminal)
   - Server automatically serves the newest file

2. **Tell user what to expect and end your turn:**
   - Remind them of the URL (every step, not just first)
   - Give a brief text summary of what's on screen (e.g., "Showing 3 layout options for the homepage")
   - Ask them to respond in the terminal: "Take a look and let me know what you think. Click to select an option if you'd like."

3. **On your next turn** — after the user responds in the terminal:
   - Read `$SCREEN_DIR/.events` if it exists — this contains the user's browser interactions (clicks, selections) as JSON lines
   - Merge with the user's terminal text to get the full picture
   - The terminal message is the primary feedback; `.events` provides structured interaction data

4. **Iterate or advance** — if feedback changes current screen, write a new file (e.g., `layout-v2.html`). Only move to the next question when the current step is validated.

5. Repeat until done.
```

- [ ] **단계 4: "사용자 피드백 형식" 섹션 교체 (165-174줄)**

다음으로 교체합니다:

```markdown
## Browser Events Format

When the user clicks options in the browser, their interactions are recorded to `$SCREEN_DIR/.events` (one JSON object per line). The file is cleared automatically when you push a new screen.

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

The full event stream shows the user's exploration path — they may click multiple options before settling. The last `choice` event is typically the final selection, but the pattern of clicks can reveal hesitation or preferences worth asking about.

If `.events` doesn't exist, the user didn't interact with the browser — use only their terminal text.
```

- [ ] **단계 5: "콘텐츠 조각 작성" 설명 업데이트 (65줄)**

"피드백 푸터" 참조 제거:

```markdown
Write just the content that goes inside the page. The server wraps it in the frame template automatically (header, theme CSS, selection indicator, and all interactive infrastructure).
```

- [ ] **단계 6: 참조(Reference) 섹션 업데이트 (200-203줄)**

"JS API"에 관한 helper.js 참조 설명을 제거합니다 — API가 이제 최소화되었습니다. 경로 참조는 유지합니다:

```markdown
## Reference

- Frame template (CSS reference): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script (client-side): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **단계 7: 커밋**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### 작업 7: 최종 검증

- [ ] **단계 1: 전체 테스트 수트 실행**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
예상 결과: 모든 테스트 통과.

- [ ] **단계 2: 수동 스모크 테스트**

서버를 수동으로 시작하고 흐름이 엔드투엔드로 작동하는지 검증합니다:

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

테스트 조각을 작성하고, 브라우저에서 열고, 옵션을 클릭한 뒤, `.events` 파일이 작성되는지 및 표시기 바가 업데이트되는지 검증합니다. 그 후 서버를 중지합니다:

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **단계 3: 오래된 참조가 남아있지 않은지 검증**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

예상 결과: 릴리스 노트 및 명세/계획 문서(히스토리성) 외에는 검색 결과가 없어야 함.

- [ ] **단계 4: 정리가 필요한 경우 최종 커밋**

```bash
git status
# 추적되지 않은/수정된 파일을 검토하고, 필요에 따라 특정 파일을 스테이징한 후 깨끗하면 커밋합니다
```
