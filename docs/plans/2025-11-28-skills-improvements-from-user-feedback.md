# 사용자 피드백을 통한 스킬 개선 사항

**날짜:** 2025-11-28
**상태:** 초안
**출처:** 실제 개발 시나리오에서 superpowers를 사용한 두 개의 Claude 인스턴스

---

## 요약 (Executive Summary)

두 개의 Claude 인스턴스가 실제 개발 세션에서 얻은 상세한 피드백을 제공했습니다. 그들의 피드백은 스킬을 준수했음에도 불구하고 방지할 수 있었던 버그가 배포되도록 만든 현재 스킬의 **체계적인 공백**을 드러냅니다.

**핵심 인사이트:** 이것은 단순한 해결책 제안이 아니라 문제 보고서입니다. 문제는 실제 존재하며, 해결책은 신중한 평가가 필요합니다.

**주요 주제:**
1. **검증 공백** - 작업 성공은 검증하지만 의도한 결과가 달성되었는지는 검증하지 않음
2. **프로세스 위생** - 백그라운드 프로세스가 누적되어 서브에이전트 간에 간섭을 일으킴
3. **컨텍스트 최적화** - 서브에이전트에게 불필요한 정보가 너무 많이 전달됨
4. **자아 성찰 누락** - 작업 인계 전에 자신의 작업을 비판적으로 되돌아보는 프롬프트 부재
5. **모의 객체(Mock) 안전성** - Mock이 감지되지 않은 채 인터페이스에서 이탈할 수 있음
6. **스킬 활성화** - 스킬이 존재하지만 읽히거나 사용되지 않음

---

## 식별된 문제점들

### Problem 1: 설정 변경 검증 공백

**발생한 상황:**
- 서브에이전트가 "OpenAI 통합"을 테스트함
- `OPENAI_API_KEY` 환경 변수를 설정함
- 상태 코드 200 응답을 받음
- "OpenAI 통합 정상 작동함"으로 보고함
- **그러나** 응답 내용에 `"model": "claude-sonnet-4-20250514"`가 포함되어 있었으며 - 실제로는 Anthropic을 사용 중이었음

**근본 원인:**
`verification-before-completion`이 작업 성공 여부는 확인하지만 결과가 의도한 설정 변경을 반영하는지는 확인하지 않음.

**영향:** 높음 - 통합 테스트에 대한 잘못된 신뢰, 버그가 프로덕션에 배포됨

**대표적 실패 패턴:**
- LLM 제공자 전환 → 상태 코드 200만 확인하고 모델 이름은 확인하지 않음
- 기능 플래그(feature flag) 활성화 → 오류가 없는 것만 확인하고 기능 활성화 여부는 확인하지 않음
- 환경 변경 → 배포 성공만 확인하고 환경 변수는 확인하지 않음

---

### Problem 2: 백그라운드 프로세스 누적

**발생한 상황:**
- 세션 동안 여러 서브에이전트가 디스패치됨
- 각 서브에이전트가 백그라운드 서버 프로세스를 시작함
- 프로세스가 누적됨 (4개 이상의 서버 실행 중)
- 오래된(stale) 프로세스가 여전히 포트에 바인딩되어 있음
- 이후 E2E 테스트가 잘못된 설정을 가진 오래된 서버에 요청을 보냄
- 혼란스럽고/잘못된 테스트 결과 발생

**근본 원인:**
서브에이전트는 상태를 유지하지 않음(stateless) - 이전 서브에이전트의 프로세스를 알지 못함. 정리 프로토콜 부재.

**영향:** 중간-높음 - 테스트가 잘못된 서버에 접근, 가짜 성공/실패 발생, 디버깅 혼란

---

### Problem 3: 서브에이전트 프롬프트의 컨텍스트 비대화

**발생한 상황:**
- 표준 접근 방식: 서브에이전트에게 전체 계획 파일을 읽도록 전달
- 실험: 작업 + 패턴 + 파일 + 검증 명령어만 전달
- 결과: 더 빠르고, 집중도가 높으며, 단 한 번 만에 완료하는 경우가 더 많아짐

**근본 원인:**
서브에이전트가 관련 없는 계획 섹션에 토큰과 주의력을 낭비함.

**영향:** 중간 - 실행 속도 저하, 더 많은 시도 실패

**효과가 있었던 방식:**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### Problem 4: 인계 전 자아 성찰 부재

**발생한 상황:**
- 자아 성찰 프롬프트 추가: "새로운 시각으로 당신의 작업을 살펴보세요 - 개선할 점은 무엇인가요?"
- Task 5의 구현자가 테스트 실패의 원인이 테스트 버그가 아니라 구현 버그임을 밝혀냄
- 99행 `strings.Join(metadata.Entrypoint, " ")`이 유효하지 않은 Docker 구문을 생성함을 추적함
- 자아 성찰이 없었다면 근본 원인 분석 없이 단순히 "테스트 실패"로 보고했을 것임

**근본 원인:**
구현자가 완료 보고를 하기 전에 자연스럽게 한 걸음 물러서서 자신의 작업을 비판적으로 검토하지 않음.

**영향:** 중간 - 구현자가 잡아낼 수 있었던 버그가 리뷰어에게 전달됨

---

### Problem 5: Mock-Interface 이탈 (Drift)

**발생한 상황:**
```typescript
// Interface defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) defines cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // Wrong!
  })),
}));
```
- 테스트 통과함
- 런타임 크래시 발생: "adapter.cleanup is not a function"

**근본 원인:**
Mock이 인터페이스 정의가 아니라 버그가 있는 코드가 호출하는 방식에 따라 작성됨. TypeScript는 잘못된 메서드 이름을 가진 인라인 Mock을 잡아내지 못함.

**영향:** 높음 - 테스트가 잘못된 신뢰감을 주고 런타임 크래시 발생

**testing-anti-patterns가 이를 방지하지 못한 이유:**
이 스킬은 Mock 동작 테스트 및 이해 없는 Mock 작성을 다루지만, "구현이 아닌 인터페이스로부터 Mock 도출"이라는 특정 패턴은 다루지 않음.

---

### Problem 6: 코드 리뷰어 파일 접근 문제

**발생한 상황:**
- 코드 리뷰어 서브에이전트 디스패치됨
- 테스트 파일을 찾을 수 없음: "저장소에 파일이 존재하지 않는 것 같습니다"
- 파일은 실제로 존재함
- 리뷰어가 명시적으로 먼저 파일을 읽어야 한다는 점을 알지 못함

**근본 원인:**
리뷰어 프롬프트에 명시적인 파일 읽기 지침이 포함되어 있지 않음.

**영향:** 낮음-중간 - 리뷰 실패 또는 불완전한 리뷰

---

### Problem 7: 수정 워크플로 지연

**발생한 상황:**
- 구현자가 자아 성찰 중 버그를 식별함
- 구현자가 해결책을 알고 있음
- 현재 워크플로: 보고 → 내가 수정자 디스패치 → 수정자가 수정 → 내가 검증
- 불필요한 왕복으로 인해 가치 창출 없이 지연만 추가됨

**근본 원인:**
구현자가 이미 진단했음에도 구현자와 수정자 역할 사이의 경직된 분리.

**영향:** 낮음 - 지연이 발생하지만 정확성 문제는 없음

---

### Problem 8: 스킬이 읽히지 않음

**발생한 상황:**
- `testing-anti-patterns` 스킬이 존재함
- 인간이나 서브에이전트 모두 테스트 작성 전에 이를 읽지 않음
- 읽었다면 일부 문제를 방지할 수 있었음 (전부는 아님 - Problem 5 참조)

**근본 원인:**
서브에이전트가 관련 스킬을 읽도록 하는 강제 조항이 없음. 스킬 읽기를 포함하는 프롬프트가 없음.

**영향:** 중간 - 사용되지 않으면 스킬 투자 낭비

---

## 제안된 개선 사항

### 1. verification-before-completion: 설정 변경 검증 추가

**새 섹션 추가:**

```markdown
## Verifying Configuration Changes

When testing changes to configuration, providers, feature flags, or environment:

**Don't just verify the operation succeeded. Verify the output reflects the intended change.**

### Common Failure Pattern

Operation succeeds because *some* valid config exists, but it's not the config you intended to test.

### Examples

| Change | Insufficient | Required |
|--------|-------------|----------|
| Switch LLM provider | Status 200 | Response contains expected model name |
| Enable feature flag | No errors | Feature behavior actually active |
| Change environment | Deploy succeeds | Logs/vars reference new environment |
| Set credentials | Auth succeeds | Authenticated user/context is correct |

### Gate Function

```
BEFORE claiming configuration change works:

1. IDENTIFY: What should be DIFFERENT after this change?
2. LOCATE: Where is that difference observable?
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: Command that shows the observable difference
4. VERIFY: Output contains expected difference
5. ONLY THEN: Claim configuration change works

Red flags:
  - "Request succeeded" without checking content
  - Checking status code but not response body
  - Verifying no errors but not positive confirmation
```

**이 방식이 효과적인 이유:**
단순히 작업 성공이 아닌, **의도(INTENT)**에 대한 검증을 강제합니다.

---

### 2. subagent-driven-development: E2E 테스트를 위한 프로세스 위생 추가

**새 섹션 추가:**

```markdown
## Process Hygiene for E2E Tests

When dispatching subagents that start services (servers, databases, message queues):

### Problem

Subagents are stateless - they don't know about processes started by previous subagents. Background processes persist and can interfere with later tests.

### Solution

**Before dispatching E2E test subagent, include cleanup in prompt:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- Stale processes serve requests with wrong config
- Port conflicts cause silent failures
- Process accumulation slows system
- Confusing test results (hitting wrong server)
```

**트레이드오프 분석:**
- 프롬프트에 기본 코드가 추가됨
- 그러나 매우 혼란스러운 디버깅 상황을 방지함
- E2E 테스트 서브에이전트에는 충분히 가치가 있음

---

### 3. subagent-driven-development: 경량 컨텍스트(Lean Context) 옵션 추가

**Step 2 수정: 서브에이전트로 작업 실행**

**수정 전:**
```
Read that task carefully from [plan-file].
```

**수정 후:**
```
## Context Approaches

**Full Plan (default):**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
Use when task is standalone and pattern-based:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern (add similar test, implement similar feature)
- Task is self-contained (doesn't need context from other tasks)
- Pattern reference is sufficient (e.g., "follow TestE2E_FeatureOptionValidation")

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**예시:**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**이 방식이 효과적인 이유:**
토큰 사용량을 줄이고 집중도를 높이며 적절한 경우 더 빠른 완료가 가능해집니다.

---

### 4. subagent-driven-development: 자아 성찰 단계 추가

**Step 2 수정: 서브에이전트로 작업 실행**

**프롬프트 템플릿에 추가:**

```
When done, BEFORE reporting back:

Take a step back and review your work with fresh eyes.

Ask yourself:
- Does this actually solve the task as specified?
- Are there edge cases I didn't consider?
- Did I follow the pattern correctly?
- If tests are failing, what's the ROOT CAUSE (implementation bug vs test bug)?
- What could be better about this implementation?

If you identify issues during this reflection, fix them now.

Then report:
- What you implemented
- Self-reflection findings (if any)
- Test results
- Files changed
```

**이 방식이 효과적인 이유:**
구현자가 인계 전에 스스로 발견할 수 있는 버그를 잡습니다. 문서화된 사례: 자아 성찰을 통해 entrypoint 버그를 식별함.

**트레이드오프:**
작업당 ~30초가 추가되지만 리뷰 전에 문제를 미리 발견할 수 있습니다.

---

### 5. requesting-code-review: 명시적 파일 읽기 추가

**code-reviewer 템플릿 수정:**

**시작 부분에 추가:**

```markdown
## Files to Review

BEFORE analyzing, read these files:

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file.

If you cannot find a file:
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code.
```

**이 방식이 효과적인 이유:**
명시적인 지침을 통해 "파일을 찾을 수 없음" 문제를 방지합니다.

---

### 6. testing-anti-patterns: Mock-Interface 이탈 안티패턴 추가

**새로운 Anti-Pattern 6 추가:**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock encodes the bug into the test
- TypeScript can't catch inline mocks with wrong method names
- Test passes because both code and mock are wrong
- Runtime crashes when real object is used

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
BEFORE writing any mock:

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**Detection:**

When you see runtime error "X is not a function" and tests pass:
1. Check if X is mocked
2. Compare mock methods to interface methods
3. Look for method name mismatches
```

**이 방식이 효과적인 이유:**
피드백에서 보고된 실패 패턴을 직접적으로 해결합니다.

---

### 7. subagent-driven-development: 테스트 서브에이전트에 대한 스킬 읽기 의무화

**테스트가 포함된 작업 시 프롬프트 템플릿에 추가:**

```markdown
BEFORE writing any tests:

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional. Tests that violate anti-patterns will be rejected in review.
```

**이 방식이 효과적인 이유:**
스킬이 단순히 존재하는 것에 그치지 않고 실제로 사용되도록 보장합니다.

**트레이드오프:**
각 작업에 시간이 추가되지만, 전체 버그 클래스를 방지합니다.

---

### 8. subagent-driven-development: 구현자가 자가 식별한 문제를 직접 수정하도록 허용

**Step 2 수정:**

**현재:**
```
Subagent reports back with summary of work.
```

**제안:**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**이 방식이 효과적인 이유:**
구현자가 이미 해결책을 알고 있을 때 지연 시간을 줄여줍니다. 문서화된 사례: entrypoint 버그에 대한 불필요한 왕복 1회를 절약할 수 있었음.

**트레이드오프:**
프롬프트가 약간 더 복잡해지지만 전체적인 처리 속도가 빨라집니다.

---

## 구현 계획

### Phase 1: 고영향, 저위험 (먼저 진행)

1. **verification-before-completion: 설정 변경 검증**
   - 명확한 추가 사항이며 기존 내용을 변경하지 않음
   - 높은 영향도의 문제(테스트에 대한 잘못된 신뢰) 해결
   - 파일: `skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns: Mock-Interface 이탈**
   - 새로운 안티패턴 추가, 기존 내용 수정 없음
   - 높은 영향도의 문제(런타임 크래시) 해결
   - 파일: `skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review: 명시적 파일 읽기**
   - 템플릿에 구체적 추가
   - 구체적인 문제(리뷰어가 파일을 찾지 못함) 해결
   - 파일: `skills/requesting-code-review/SKILL.md`

### Phase 2: 중간 정도의 변경 사항 (신중히 테스트)

4. **subagent-driven-development: 프로세스 위생**
   - 새 섹션 추가, 워크플로 변경 없음
   - 중간-높은 영향(테스트 신뢰성) 해결
   - 파일: `skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development: 자아 성찰**
   - 프롬프트 템플릿 변경 (상대적으로 높은 위험)
   - 그러나 버그를 잡는 것으로 문서화됨
   - 파일: `skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development: 스킬 읽기 의무화**
   - 프롬프트 오버헤드 추가
   - 그러나 스킬이 실제로 사용되도록 보장함
   - 파일: `skills/subagent-driven-development/SKILL.md`

### Phase 3: 최적화 (먼저 검증)

7. **subagent-driven-development: 경량 컨텍스트 옵션**
   - 복잡성 추가 (두 가지 접근 방식)
   - 혼란을 일으키지 않는지 검증 필요
   - 파일: `skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development: 구현자의 직접 수정 허용**
   - 워크플로 변경 (상대적으로 높은 위험)
   - 버그 수정이 아닌 최적화
   - 파일: `skills/subagent-driven-development/SKILL.md`

---

## 열린 질문들 (Open Questions)

1. **경량 컨텍스트 접근 방식:**
   - 패턴 기반 작업의 기본값으로 지정해야 할까요?
   - 어떤 접근 방식을 사용할지 어떻게 결정할까요?
   - 너무 지나치게 줄여서 중요한 컨텍스트를 놓칠 위험은 없을까요?

2. **자아 성찰:**
   - 이것이 간단한 작업을 크게 지연시키지는 않을까요?
   - 복잡한 작업에만 적용해야 할까요?
   - 판에 박힌 일상적 작업이 되어버리는 "성찰 피로"를 어떻게 방지할까요?

3. **프로세스 위생:**
   - 이것이 subagent-driven-development에 있어야 할까요, 아니면 별도의 스킬이어야 할까요?
   - E2E 테스트 이외의 다른 워크플로에도 적용될까요?
   - 프로세스가 계속 유지되어야 하는 경우(dev 서버)는 어떻게 처리할까요?

4. **스킬 읽기 강제:**
   - 모든 서브에이전트가 관련 스킬을 읽도록 의무화해야 할까요?
   - 프롬프트가 너무 길어지지 않도록 어떻게 관리할까요?
   - 과도한 문서화로 인해 초점을 잃을 위험은 없을까요?

---

## 성공 지표

이러한 개선 사항이 효과가 있는지 어떻게 알 수 있을까요?

1. **설정 검증:**
   - "테스트는 통과했지만 잘못된 설정이 사용됨" 사례 0건
   - Jesse가 "그것은 당신이 생각하는 것을 실제로 테스트하는 것이 아닙니다"라고 말하는 경우 0건

2. **프로세스 위생:**
   - "테스트가 잘못된 서버에 접근함" 사례 0건
   - E2E 테스트 실행 중 포트 충돌 오류 0건

3. **Mock-Interface 이탈:**
   - "테스트는 통과하지만 누락된 메서드로 인해 런타임 크래시 발생" 사례 0건
   - Mock과 인터페이스 간 메서드 이름 불일치 0건

4. **자아 성찰:**
   - 측정 가능: 구현자 보고서에 자아 성찰 결과가 포함되는가?
   - 질적: 코드 리뷰 단계로 넘어가는 버그가 줄어드는가?

5. **스킬 읽기:**
   - 서브에이전트 보고서가 스킬 겟 게이트 함수(gate functions)를 참조함
   - 코드 리뷰에서 안티패턴 위반 사례가 감소함

---

## 위험 및 완화 방안

### Risk: 프롬프트 비대화
**Problem:** 이 모든 요구 사항을 추가하면 프롬프트가 너무 압도적이 됨
**Mitigation:**
- 단계적 구현 (모든 것을 한 번에 추가하지 않음)
- 일부 추가 사항은 조건부로 설정 (E2E 위생은 E2E 테스트에만 적용)
- 다양한 작업 유형에 대한 템플릿 고려

### Risk: 분석 마비 (Analysis Paralysis)
**Problem:** 과도한 성찰/검증으로 인해 실행이 지연됨
**Mitigation:**
- 게이트 함수를 빠르게 유지 (분 단위가 아닌 초 단위)
- 경량 컨텍스트를 초기에는 선택 사항(opt-in)으로 설정
- 작업 완료 시간 모니터링

### Risk: 거짓된 안전감
**Problem:** 체크리스트를 따르는 것이 올바름을 보장하지는 않음
**Mitigation:**
- 게이트 함수는 최소한의 기준이지 최대 기준이 아님을 강조
- 스킬에 "판단력 사용" 언어 유지
- 스킬이 모든 실패가 아닌 일반적인 실패를 잡는다는 점을 문서화

### Risk: 스킬 분산/모순 (Skill Divergence)
**Problem:** 서로 다른 스킬이 상충되는 조언을 제공함
**Mitigation:**
- 일관성을 위해 모든 스킬의 변경 사항 검토
- 스킬 간 상호작용 방식 문서화 (통합 섹션)
- 배포 전 실제 시나리오로 테스트

---

## 추천 사항

**즉시 Phase 1 진행:**
- verification-before-completion: 설정 변경 검증
- testing-anti-patterns: Mock-Interface 이탈
- requesting-code-review: 명시적 파일 읽기

**최종 결정 전 Jesse와 Phase 2 테스트:**
- 자아 성찰의 영향에 대한 피드백 구하기
- 프로세스 위생 접근 방식 검증
- 스킬 읽기 요구 사항이 오버헤드 대비 가치가 있는지 확인

**검증을 거칠 때까지 Phase 3 보류:**
- 경량 컨텍스트는 실제 테스트 필요
- 구현자 직접 수정 워크플로 변경은 신중한 평가 필요

이러한 변경 사항은 스킬 악화 위험을 최소화하면서 사용자가 문서화한 실제 문제를 해결합니다.
