# 심층 방어(Defense-in-Depth) 검증

## 개요

유효하지 않은 데이터로 인한 버그를 고칠 때, 한 곳에 검증을 추가하는 것으로 충분하다고 생각하기 쉽습니다. 그러나 단일 검증은 다른 코드 경로, 리팩터링 또는 모의 객체(mock)에 의해 우회될 수 있습니다.

**핵심 원칙:** 데이터가 거쳐가는 모든 레이어(layer)에서 검증하세요. 버그가 구조적으로 불가능하도록 만드세요.

## 여러 레이어가 필요한 이유

단일 검증: "버그를 고쳤다"
다중 레이어: "버그를 불가능하게 만들었다"

서로 다른 레이어가 서로 다른 케이스를 잡아냅니다:
- 진입점 검증(Entry validation)은 대부분의 버그를 포착함
- 비즈니스 로직(Business logic)은 엣지 케이스를 포착함
- 환경 보호 장치(Environment guards)는 컨텍스트별 위험을 방지함
- 디버그 로그(Debug logging)는 다른 레이어가 실패했을 때 원인 분석을 도움

## 4가지 레이어

### Layer 1: 진입점 검증 (Entry Point Validation)
**목적:** API 경계에서 명백히 유효하지 않은 입력을 거부함

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error(`workingDirectory does not exist: ${workingDirectory}`);
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error(`workingDirectory is not a directory: ${workingDirectory}`);
  }
  // ... 진행
}
```

### Layer 2: 비즈니스 로직 검증 (Business Logic Validation)
**목적:** 해당 연산에 데이터가 적합한지 확인함

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... 진행
}
```

### Layer 3: 환경 보호 장치 (Environment Guards)
**목적:** 특정 컨텍스트에서 위험한 연산이 실행되는 것을 방지함

```typescript
async function gitInit(directory: string) {
  // 테스트 중에는 임시 디렉터리 외부에서의 git init을 거부함
  if (process.env.NODE_ENV === 'test') {
    const normalized = normalize(resolve(directory));
    const tmpDir = normalize(resolve(tmpdir()));

    if (!normalized.startsWith(tmpDir)) {
      throw new Error(
        `Refusing git init outside temp dir during tests: ${directory}`
      );
    }
  }
  // ... 진행
}
```

### Layer 4: 디버그 로깅 (Debug Instrumentation)
**목적:** 원인 분석을 위해 컨텍스트 정보를 캡처함

```typescript
async function gitInit(directory: string) {
  const stack = new Error().stack;
  logger.debug('About to git init', {
    directory,
    cwd: process.cwd(),
    stack,
  });
  // ... 진행
}
```

## 패턴 적용하기

버그를 발견했을 때:

1. **데이터 흐름 추적** - 잘못된 값이 어디서 시작하여 어디서 사용되는가?
2. **모든 체크포인트 맵핑** - 데이터가 거쳐가는 모든 지점 나열
3. **각 레이어별 검증 추가** - 진입점, 비즈니스, 환경, 디버그
4. **각 레이어 검증 테스트** - Layer 1을 우회해보고 Layer 2가 잡아내는지 확인

## 실무 세션 예시

버그: 빈 `projectDir`로 인해 소스 코드 디렉터리에서 `git init`이 실행됨

**데이터 흐름:**
1. 테스트 셋업 → 빈 문자열
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `process.cwd()`에서 `git init` 실행

**추가된 4가지 레이어:**
- Layer 1: `Project.create()`에서 비어 있지 않음/존재함/쓰기 가능함 검증
- Layer 2: `WorkspaceManager`에서 projectDir이 비어 있지 않음을 검증
- Layer 3: `WorktreeManager`에서 테스트 중 tmpdir 외부 git init 거부
- Layer 4: git init 전 스택 트레이스 로깅

**결과:** 1847개 테스트 모두 통과, 버그 재현 불가능

## 핵심 인사이트

4개 레이어 모두가 필요했습니다. 테스트를 진행하는 동안 각 레이어는 다른 레이어가 놓친 버그를 잡아냈습니다:
- 서로 다른 코드 경로가 진입점 검증을 우회함
- 모의 객체(mock)가 비즈니스 로직 검사를 우회함
- 서로 다른 플랫폼의 엣지 케이스에 환경 보호 장치가 필요했음
- 디버그 로깅이 구조적 오용 패턴을 식별해 냄

**단 하나의 검증 지점에 멈추지 마세요.** 모든 레이어에 검사를 추가하세요.
