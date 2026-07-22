# 테스트 안티 패턴 (Testing Anti-Patterns)

**다음 상황일 때 이 문서를 로드하세요:** 테스트를 작성하거나 변경할 때, 모의 객체(mock)를 추가할 때, 또는 프로덕션 코드에 테스트 전용 메서드를 추가하고 싶은 유혹이 들 때.

## 개요

테스트는 모의 객체(mock)의 동작이 아니라 실제 동작을 검증해야 합니다. mock은 격리를 위한 수단일 뿐, 테스트 대상 그 자체가 아닙니다.

**핵심 원칙:** mock이 수행하는 작업이 아니라 코드가 수행하는 작업을 테스트하세요.

**엄격한 TDD를 준수하면 이러한 안티 패턴을 방지할 수 있습니다.**

## 절대 법칙 (The Iron Laws)

```
1. 절대 mock의 동작을 테스트하지 말 것
2. 절대 프로덕션 클래스에 테스트 전용 메서드를 추가하지 말 것
3. 절대 의존성을 이해하지 않은 상태에서 mock을 사용하지 말 것
```

## 안티 패턴 1: Mock 동작 테스트하기

**위반 사례:**
```typescript
// ❌ BAD: mock이 존재하는지 테스트하는 경우
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**잘못된 이유:**
- 컴포넌트의 동작이 아니라 mock이 작동하는지 검증하고 있음
- mock이 존재할 때는 테스트가 통과하지만, 없을 때는 실패함
- 실제 동작에 대해 아무것도 알려주지 않음

**사람 파트너의 지적:** "우리가 지금 mock의 동작을 테스트하고 있는 건가요?"

**해결책:**
```typescript
// ✅ GOOD: 실제 컴포넌트를 테스트하거나 mock 처리하지 않음
test('renders sidebar', () => {
  render(<Page />);  // sidebar를 mock하지 않음
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 또는 격리를 위해 sidebar를 반드시 mock해야 하는 경우:
// mock 자체에 대해 단정(assert)하지 말고, sidebar가 존재하는 상황에서 Page의 동작을 테스트함
```

### 게이트 함수 (Gate Function)

```
mock 엘리먼트에 대해 단정(assert)하기 전에:
  질문: "내가 실제 컴포넌트의 동작을 테스트하고 있는가, 아니면 단지 mock의 존재 여부만 검증하고 있는가?"

  만약 mock의 존재 여부만 검증하고 있다면:
    중단 - 단정문(assertion)을 삭제하거나 컴포넌트의 mock을 해제하세요

  대신 실제 동작을 테스트하세요
```

## 안티 패턴 2: 프로덕션 코드 내 테스트 전용 메서드

**위반 사례:**
```typescript
// ❌ BAD: destroy()가 오직 테스트에서만 사용됨
class Session {
  async destroy() {  // 프로덕션 API처럼 보임!
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... cleanup
  }
}

// 테스트 코드 내에서
afterEach(() => session.destroy());
```

**잘못된 이유:**
- 프로덕션 클래스가 테스트 전용 코드로 오염됨
- 프로덕션 환경에서 실수로 호출될 경우 위험함
- YAGNI 및 관심사 분리(separation of concerns) 원칙 위반
- 객체의 수명 주기와 엔티티의 수명 주기를 혼동함

**해결책:**
```typescript
// ✅ GOOD: 테스트 유틸리티가 테스트 정리를 담당함
// Session에는 destroy()가 없음 - 프로덕션에서는 상태를 유지하지 않음(stateless)

// test-utils/ 내에서
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 테스트 코드 내에서
afterEach(() => cleanupSession(session));
```

### 게이트 함수 (Gate Function)

```
프로덕션 클래스에 메서드를 추가하기 전에:
  질문: "이 메서드가 오직 테스트에 의해서만 사용되는가?"

  만약 그렇다면:
    중단 - 메서드를 추가하지 마세요
    대신 테스트 유틸리티에 넣으세요

  질문: "이 클래스가 해당 리소스의 수명 주기를 소유하는가?"

  만약 아니라면:
    중단 - 이 메서드가 위치하기에 잘못된 클래스입니다
```

## 안티 패턴 3: 이해 없는 Mocking

**위반 사례:**
```typescript
// ❌ BAD: mock이 테스트 로직을 망가뜨림
test('detects duplicate server', () => {
  // mock으로 인해 테스트가 의존하는 설정 쓰기 작업이 차단됨!
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 예외를 던져야 하지만 그러지 않음!
});
```

**잘못된 이유:**
- mock 처리된 메서드가 테스트에 필요한 부작용(side effect, 설정 파일 쓰기)을 가지고 있었음
- "안전을 위해" 수행한 과도한 mocking이 실제 동작을 깨뜨림
- 테스트가 잘못된 이유로 통과하거나 원인을 알 수 없이 실패함

**해결책:**
```typescript
// ✅ GOOD: 올바른 수준에서의 mocking
test('detects duplicate server', () => {
  // 느린 부분만 mock 처리하고, 테스트에 필요한 동작은 유지함
  vi.mock('MCPServerManager'); // 느린 서버 시작 부분만 mock

  await addServer(config);  // 설정이 작성됨
  await addServer(config);  // 중복 감지됨 ✓
});
```

### 게이트 함수 (Gate Function)

```
어떤 메서드를 mock하기 전에:
  중단 - 아직 mock하지 마세요

  1. 질문: "실제 메서드가 어떤 부작용(side effect)을 가지는가?"
  2. 질문: "이 테스트가 해당 부작용 중 어느 하나라도 의존하는가?"
  3. 질문: "이 테스트에 필요한 것이 무엇인지 완전히 이해하고 있는가?"

  부작용에 의존하는 경우:
    더 낮은 수준(실제 느리거나 외부와 통신하는 연산)에서 mock하세요
    또는 필요한 동작을 보존하는 테스트 더블(test double)을 사용하세요
    테스트가 의존하는 상위 수준 메서드를 mock하지 마세요

  테스트가 무엇에 의존하는지 불확실한 경우:
    먼저 실제 구현으로 테스트를 실행해보세요
    실제로 무슨 일이 일어나야 하는지 관찰하세요
    그런 다음 올바른 수준에서 최소한의 mocking을 추가하세요

  Red flags:
    - "안전을 위해 이걸 mock해 두자"
    - "이건 느릴 수 있으니 mock하는 게 낫겠어"
    - 의존성 체인을 이해하지 못한 채 mocking하는 행위
```

## 안티 패턴 4: 불완전한 Mocks

**위반 사례:**
```typescript
// ❌ BAD: 부분 mock - 필요하다고 생각하는 필드만 생성
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 누락됨: 하위 코드가 사용하는 메타데이터
};

// 나중에: 코드가 response.metadata.requestId에 접근할 때 깨짐
```

**잘못된 이유:**
- **부분 mock은 구조적 가정을 감춤** - 당신이 알고 있는 필드만 mock했기 때문
- **하위 코드가 포함하지 않은 필드에 의존할 수 있음** - 조용한 실패 발생
- **테스트는 통과하지만 통합은 실패함** - mock은 불완전하고 실제 API는 완전하기 때문
- **거짓된 신뢰감** - 테스트가 실제 동작에 대해 아무것도 증명하지 못함

**절대 규칙:** 직전 테스트가 사용하는 필드만이 아니라, 실제로 존재하는 형태 그대로 **완전한(COMPLETE)** 데이터 구조를 mock하세요.

**해결책:**
```typescript
// ✅ GOOD: 실제 API의 완전성을 반영
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 실제 API가 반환하는 모든 필드 포함
};
```

### 게이트 함수 (Gate Function)

```
mock 응답을 만들기 전에:
  확인: "실제 API 응답에는 어떤 필드가 포함되어 있는가?"

  조치 사항:
    1. 문서/예시에서 실제 API 응답을 조사하세요
    2. 시스템이 하위에서 소비할 수 있는 모든 필드를 포함하세요
    3. mock이 실제 응답 스키마와 완전히 일치하는지 검증하세요

  중요:
    mock을 생성할 때는 전체 구조를 이해해야 합니다
    누락된 필드에 코드가 의존할 경우 부분 mock은 조용히 실패합니다

  불확실한 경우: 문서화된 모든 필드를 포함하세요
```

## 안티 패턴 5: 나중에 떠올린 생각으로서의 통합 테스트

**위반 사례:**
```
✅ 구현 완료
❌ 작성된 테스트 없음
"테스트할 준비 완료"
```

**잘못된 이유:**
- 테스트는 구현의 일부이지 선택적 후속 작업이 아닙니다
- TDD를 적용했다면 이를 사전에 포착했을 것입니다
- 테스트 없이는 완료되었다고 주장할 수 없습니다

**해결책:**
```
TDD 사이클:
1. 실패하는 테스트 작성
2. 통과하도록 구현
3. 리팩터링
4. 그 후에 완료 선언
```

## Mock이 너무 복잡해질 때

**경고 신호:**
- mock 설정이 테스트 로직보다 늚
- 테스트를 통과시키기 위해 모든 것을 mock함
- 실제 컴포넌트에 있는 메서드가 mock에는 누락됨
- mock이 변경될 때 테스트가 깨짐

**사람 파트너의 질문:** "여기서 정말 mock을 사용해야 하나요?"

**고려할 점:** 실제 컴포넌트를 사용한 통합 테스트가 복잡한 mock보다 더 간단한 경우가 많습니다.

## TDD가 이러한 안티 패턴을 방지하는 이유

**TDD가 도움이 되는 이유:**
1. **테스트를 먼저 작성** → 실제로 무엇을 테스트하는지 진지하게 고민하게 함
2. **실패를 관찰** → 테스트가 mock이 아닌 실제 동작을 검증함을 확인함
3. **최소한의 구현** → 테스트 전용 메서드가 스며들지 않음
4. **실제 의존성 파악** → mock을 적용하기 전에 테스트에 진짜 필요한 것이 무엇인지 파악함

**만약 mock 동작을 테스트하고 있다면, TDD를 위반한 것입니다** - 실제 코드에 대해 테스트가 실패하는 것을 먼저 지켜보지 않고 mock을 추가한 것입니다.

## 빠른 참조 (Quick Reference)

| 안티 패턴 | 해결책 |
|--------------|-----|
| mock 엘리먼트에 대한 단정(assert) | 실제 컴포넌트를 테스트하거나 mock을 해제 |
| 프로덕션 내 테스트 전용 메서드 | 테스트 유틸리티로 이동 |
| 이해 없는 mocking | 의존성을 먼저 이해하고 최소한으로 mock |
| 불완전한 mocks | 실제 API를 완전히 반영 |
| 나중에 떠올린 테스트 | TDD 적용 - 테스트를 먼저 작성 |
| 과도하게 복잡한 mocks | 통합 테스트 고려 |

## Red Flags

- `*-mock` 테스트 ID를 확인하는 단정문
- 테스트 파일에서만 호출되는 메서드
- mock 설정이 테스트 전체의 50% 이상을 차지함
- mock을 제거하면 테스트가 실패함
- mock이 왜 필요한지 설명할 수 없음
- "그냥 안전을 위해" mock함

## 요약 (The Bottom Line)

**mock은 격리를 위한 도구일 뿐, 테스트할 대상이 아닙니다.**

TDD 수행 중 mock의 동작을 테스트하고 있음을 깨달았다면, 무언가 잘못된 것입니다.

해결책: 실제 동작을 테스트하거나, 애초에 왜 mock을 사용하고 있는지 스스로 질문해 보세요.
