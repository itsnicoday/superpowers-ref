---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# Git Worktree 사용하기

## 개요

작업이 격리된 워크스페이스에서 수행되도록 합니다. 플랫폼의 네이티브 워크트리 도구를 우선적으로 사용하세요. 네이티브 도구를 사용할 수 없는 경우에만 수동 git worktree로 대체(fallback)합니다.

**핵심 원칙:** 기존 격리 상태를 먼저 감지하세요. 그런 다음 네이티브 도구를 사용하세요. 이후 git으로 대체하세요. 절대 하네스(harness)와 충돌하지 마세요.

**시작 시 알림:** "격리된 워크스페이스를 설정하기 위해 using-git-worktrees 스킬을 사용합니다."

## Step 0: 기존 격리 상태 감지

**무엇인가를 생성하기 전에, 이미 격리된 워크스페이스에 있는지 확인하세요.**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**서브모듈 보호:** `GIT_DIR != GIT_COMMON` 조건은 git 서브모듈 내부에서도 참(true)입니다. "이미 워크트리에 있음"으로 결론짓기 전에, 서브모듈에 있지 않은지 검증하세요:

```bash
# 이 명령이 경로를 반환하면 워크트리가 아니라 서브모듈에 있는 것입니다 — 일반 저장소로 취급하세요
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**`GIT_DIR != GIT_COMMON` 인 경우 (서브모듈이 아닌 경우):** 이미 연결된 워크트리에 있는 상태입니다. Step 2 (프로젝트 설정)로 건너뛰세요. 또 다른 워크트리를 생성하지 마세요.

브랜치 상태와 함께 보고하세요:
- 브랜치 위인 경우: "`<path>` 위치의 격리된 워크스페이스(브랜치: `<name>`)에 이미 존재합니다."
- Detached HEAD인 경우: "`<path>` 위치의 격리된 워크스페이스(detached HEAD, 외부 관리됨)에 이미 존재합니다. 완료 시점에 브랜치 생성이 필요합니다."

**`GIT_DIR == GIT_COMMON` 인 경우 (또는 서브모듈 내부인 경우):** 일반 저장소 체크아웃 상태입니다.

지시사항에 사용자의 워크트리 선호도가 이미 표시되어 있나요? 그렇지 않다면 워크트리를 생성하기 전에 동의를 구하세요:

> "격리된 워크스페이스(worktree)를 설정할까요? 현재 브랜치를 변경 사항으로부터 보호할 수 있습니다."

이미 선언된 선호도가 있다면 묻지 않고 이를 준수하세요. 사용자가 동의하지 않으면 현재 위치에서 작업하고 Step 2로 건너뛰세요.

## Step 1: 격리된 워크스페이스 생성

**두 가지 메커니즘이 있습니다. 다음 순서대로 시도하세요.**

### 1a. 네이티브 워크트리 도구 (권장)

사용자가 격리된 워크스페이스를 요청했습니다 (Step 0 동의). 워크트리를 생성하는 수단이 이미 제공되어 있나요? `EnterWorktree`, `WorktreeCreate`, `/worktree` 명령, 또는 `--worktree` 플래그와 같은 이름의 도구일 수 있습니다. 존재한다면 이를 사용하고 Step 2로 건너뛰세요.

네이티브 도구는 디렉터리 배치, 브랜치 생성 및 정리 작업을 자동으로 처리합니다. 네이티브 도구가 존재할 때 `git worktree add`를 사용하면 하네스가 인식하거나 관리할 수 없는 유령 상태가 생성됩니다.

사용 가능한 네이티브 워크트리 도구가 없는 경우에만 Step 1b로 진행하세요.

### 1b. Git Worktree 대체(Fallback)

**Step 1a가 적용되지 않는 경우에만 이 방법을 사용하세요** — 네이티브 워크트리 도구를 사용할 수 없는 상황입니다. git을 사용하여 워크트리를 수동으로 생성하세요.

#### 디렉터리 선택

다음 우선순위를 따르세요. 명시적인 사용자 선호도는 항상 관찰된 파일 시스템 상태보다 우선합니다.

1. **지시사항에 선언된 워크트리 디렉터리 선호도가 있는지 확인하세요.** 사용자가 이미 지정했다면 묻지 않고 해당 위치를 사용하세요.

2. **기존 프로젝트 로컬 워크트리 디렉터리가 있는지 확인하세요:**
   ```bash
   ls -d .worktrees 2>/dev/null     # 권장 (숨김 디렉터리)
   ls -d worktrees 2>/dev/null      # 대안
   ```
   발견되면 해당 디렉터리를 사용하세요. 둘 다 존재하면 `.worktrees`가 우선합니다.

3. **이용 가능한 다른 안내 지침이 없다면**, 프로젝트 루트의 `.worktrees/`를 기본값으로 사용하세요.

#### 안전 검증 (프로젝트 로컬 디렉터리 전용)

**워크트리를 생성하기 전에 디렉터리가 ignore 처리되었는지 반드시 검증해야 합니다:**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**ignore 처리되지 않은 경우:** `.gitignore`에 추가하고 변경 사항을 커밋한 후 진행하세요.

**중요한 이유:** 워크트리 내용이 실수로 저장소에 커밋되는 것을 방지합니다.

#### 워크트리 생성

```bash
# 선택한 위치에 따라 경로 결정
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**샌드박스 대체(Fallback):** `git worktree add`가 권한 오류(샌드박스 거부)로 실패하면, 샌드박스로 인해 워크트리 생성이 차단되었으며 대신 현재 디렉터리에서 작업함을 사용자에게 알리세요. 그런 다음 해당 위치에서 설정 및 베이스라인 테스트를 실행하세요.

## Step 2: 프로젝트 설정

적절한 설정을 자동 감지하여 실행하세요:

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## Step 3: 깨끗한 베이스라인 검증

워크스페이스가 깨끗하게 시작되는지 확인하기 위해 테스트를 실행하세요:

```bash
# 프로젝트에 적합한 명령 사용
npm test / cargo test / pytest / go test ./...
```

**테스트가 실패한 경우:** 실패 내역을 보고하고, 진행할지 조사할지 물어보세요.

**테스트가 통과한 경우:** 준비 완료를 보고하세요.

### 보고

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 빠른 참조 (Quick Reference)

| 상황 | 조치 |
|-----------|--------|
| 이미 연결된 워크트리에 있음 | 생성 건너뛰기 (Step 0) |
| 서브모듈 내부 | 일반 저장소로 취급 (Step 0 보호 조건) |
| 네이티브 워크트리 도구 사용 가능 | 해당 도구 사용 (Step 1a) |
| 네이티브 도구 없음 | Git worktree 대체 사용 (Step 1b) |
| `.worktrees/` 존재함 | 사용 (ignore 검증) |
| `worktrees/` 존재함 | 사용 (ignore 검증) |
| 둘 다 존재함 | `.worktrees/` 사용 |
| 둘 다 존재하지 않음 | 지시사항 파일 확인 후 `.worktrees/` 기본 사용 |
| 디렉터리가 ignore되지 않음 | .gitignore에 추가 + 커밋 |
| 생성 시 권한 오류 발생 | 샌드박스 대체, 현재 위치에서 작업 |
| 베이스라인 중 테스트 실패 | 실패 보고 + 문의 |
| package.json/Cargo.toml 없음 | 의존성 설치 건너뛰기 |

## 흔한 실수 (Common Mistakes)

### 하네스와 충돌하기

- **문제:** 플랫폼이 이미 격리 환경을 제공함에도 `git worktree add`를 사용하는 경우
- **해결책:** Step 0에서 기존 격리를 감지합니다. Step 1a는 네이티브 도구에 양보합니다.

### 감지 단계 건너뛰기

- **문제:** 기존 워크트리 내부에서 중첩 워크트리를 생성하는 경우
- **해결책:** 무언가를 생성하기 전에 항상 Step 0을 실행하세요.

### Ignore 검증 건너뛰기

- **문제:** 워크트리 내용이 추적되어 git status가 지저분해짐
- **해결책:** 프로젝트 로컬 워크트리를 생성하기 전에 항상 `git check-ignore`를 사용하세요.

### 디렉터리 위치 임의 지정

- **문제:** 불일치를 유발하고 프로젝트 컨벤션을 위반함
- **해결책:** 우선순위를 따르세요: 명시적 지시사항 > 기존 프로젝트 로컬 디렉터리 > 기본값

### 실패하는 테스트를 무시하고 진행하기

- **문제:** 새로운 버그와 기존 문제를 구별할 수 없음
- **해결책:** 실패 내역을 보고하고, 진행에 대한 명시적 허가를 받으세요.

## Red Flags

**절대 하지 말 것:**
- Step 0에서 기존 격리가 감지되었을 때 워크트리 생성
- 네이티브 워크트리 도구(예: `EnterWorktree`)가 존재함에도 `git worktree add` 사용. 이것이 가장 흔한 실수입니다 — 도구가 있다면 해당 도구를 사용하세요.
- Step 1b의 git 명령으로 바로 점프하여 Step 1a를 건너뛰기
- ignore 처리 여부를 검증하지 않고 워크트리 생성 (프로젝트 로컬 기준)
- 베이스라인 테스트 검증 건너뛰기
- 묻지 않고 실패하는 테스트를 두고 계속 진행하기

**항상 할 것:**
- Step 0 감지를 먼저 실행
- git 대체(fallback)보다 네이티브 도구를 우선 사용
- 디렉터리 우선순위 따르기: 명시적 지시사항 > 기존 프로젝트 로컬 디렉터리 > 기본값
- 프로젝트 로컬 디렉터리가 ignore 되었는지 검증
- 프로젝트 설정 자동 감지 및 실행
- 깨끗한 테스트 베이스라인 검증
