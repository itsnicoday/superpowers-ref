---
name: finishing-a-development-branch
description: 구현이 완료되고 모든 테스트가 통과했을 때, 작업 통합 방식을 결정하기 위해 사용 - 머지, PR 또는 정리 옵션을 제시하여 개발 작업 완료를 안내함
---

# 개발 브랜치 완료하기 (Finishing a Development Branch)

## 개요 (Overview)

명확한 옵션을 제시하고 선택한 워크플로우를 처리하여 개발 작업의 완료를 안내합니다.

**핵심 원칙:** 테스트 검증 → 환경 감지 → 옵션 제시 → 선택 사항 실행 → 정리.

**시작 시 안내:** "개발 작업을 완료하기 위해 finishing-a-development-branch 스킬을 사용합니다."

## 프로세스 (The Process)

### 1단계: 테스트 검증 (Verify Tests)

**옵션을 제시하기 전에 테스트 통과 여부를 검증하세요:**

```bash
# 프로젝트의 테스트 수트 실행
npm test / cargo test / pytest / go test ./...
```

**테스트가 실패한 경우:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

중단하세요. 2단계로 진행하지 마세요.

**테스트가 통과한 경우:** 2단계로 진행하세요.

### 2단계: 환경 감지 (Detect Environment)

**옵션을 제시하기 전에 워크스페이스 상태를 확인하세요:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

이 상태에 따라 표시할 메뉴와 정리 방식이 결정됩니다:

| 상태 (State) | 메뉴 (Menu) | 정리 (Cleanup) |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (일반 저장소) | 표준 4가지 옵션 | 정리할 worktree 없음 |
| `GIT_DIR != GIT_COMMON`, 이름이 지정된 브랜치 | 표준 4가지 옵션 | 출처 기반 정리 (6단계 참조) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | 축소된 3가지 옵션 (머지 없음) | 정리 없음 (외부에서 관리됨) |

### 3단계: 베이스 브랜치 결정 (Determine Base Branch)

```bash
# 일반적인 베이스 브랜치 시도
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

또는 확인 질문: "이 브랜치는 main에서 분기되었습니다 - 맞습니까?"

### 4단계: 옵션 제시 (Present Options)

**일반 저장소 및 이름이 지정된 브랜치 worktree — 정확히 다음 4가지 옵션을 제시하세요:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD — 정확히 다음 3가지 옵션을 제시하세요:**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**추가 설명을 덧붙이지 마세요** - 옵션을 간결하게 유지하세요.

### 5단계: 선택 사항 실행 (Execute Choice)

#### 옵션 1: 로컬 머지 (Merge Locally)

```bash
# CWD 안전성을 위해 메인 저장소 루트 가져오기
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 먼저 머지 수행 — 무엇이든 삭제하기 전에 성공 여부 검증
git checkout <base-branch>
git pull
git merge <feature-branch>

# 머지된 결과에 대해 테스트 검증
<test command>

# 머지가 성공한 후에만: worktree 정리(6단계), 그 후 브랜치 삭제
```

그 후: worktree 정리 (6단계), 그리고 브랜치 삭제:

```bash
git branch -d <feature-branch>
```

#### 옵션 2: Push 및 PR 생성 (Push and Create PR)

```bash
# 브랜치 푸시
git push -u origin <feature-branch>
```

**worktree를 정리하지 마세요** — 사용자가 PR 피드백을 반복 반영할 수 있도록 유지해야 합니다.

#### 옵션 3: 그대로 유지 (Keep As-Is)

보고: "Keeping branch <name>. Worktree preserved at <path>."

**worktree를 정리하지 마세요.**

#### 옵션 4: 폐기 (Discard)

**먼저 확인을 받으세요:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

정확한 입력을 기다리세요.

확인된 경우:
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

그 후: worktree 정리 (6단계), 그리고 브랜치 강제 삭제:
```bash
git branch -D <feature-branch>
```

### 6단계: 워크스페이스 정리 (Cleanup Workspace)

**옵션 1과 4에 대해서만 실행됩니다.** 옵션 2와 3은 항상 worktree를 보존합니다.

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**`GIT_DIR == GIT_COMMON`인 경우:** 일반 저장소이므로 정리할 worktree가 없습니다. 완료.

**worktree 경로가 `.worktrees/` 또는 `worktrees/` 하위에 있는 경우:** Superpowers가 이 worktree를 생성했으므로 정리를 수행합니다.

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 자체 복구: 오래된 등록 정보 정리
```

**그 외의 경우:** 호스트 환경(harness)이 이 워크스페이스를 소유합니다. 제거하지 마세요. 플랫폼에서 워크스페이스 종료 툴을 제공하는 경우 이를 사용하세요. 그렇지 않다면 워크스페이스를 그대로 두세요.

## 빠른 참조 (Quick Reference)

| 옵션 (Option) | 머지 (Merge) | 푸시 (Push) | Worktree 보존 | 브랜치 정리 (Cleanup Branch) |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | yes | - | - | yes |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | yes (force) |

## 자주 하는 실수 (Common Mistakes)

**테스트 검증 건너뛰기**
- **문제:** 깨진 코드를 머지하거나 실패하는 PR 생성
- **해결책:** 옵션을 제공하기 전에 항상 테스트를 검증하세요.

**개방형 질문**
- **문제:** "다음에 무엇을 해야 하나요?"라는 질문은 모호합니다.
- **해결책:** 정확히 4가지 구조화된 옵션(detached HEAD의 경우 3가지)을 제시하세요.

**옵션 2에서 worktree 정리하기**
- **문제:** 사용자가 PR 피드백 반영에 필요한 worktree를 제거함
- **해결책:** 옵션 1과 4에 대해서만 정리를 수행하세요.

**worktree 제거 전에 브랜치 삭제하기**
- **문제:** worktree가 여전히 브랜치를 참조하고 있어 `git branch -d`가 실패함
- **해결책:** 먼저 머지하고, worktree를 제거한 다음, 브랜치를 삭제하세요.

**worktree 내부에서 git worktree remove 실행하기**
- **문제:** CWD가 제거 대상 worktree 내부일 때 명령이 조용히 실패함
- **해결책:** `git worktree remove` 실행 전에 항상 메인 저장소 루트로 `cd` 하세요.

**하네스(harness) 소유의 worktree 정리하기**
- **문제:** 하네스가 생성한 worktree를 제거하면 유령 상태(phantom state)가 발생함
- **해결책:** `.worktrees/` 또는 `worktrees/` 하위의 worktree만 정리하세요.

**폐기 시 확인 절차 없음**
- **문제:** 실수로 작업을 삭제함
- **해결책:** "discard" 입력을 통해 명시적 확인을 요청하세요.

## 주의 사항 (Red Flags)

**절대 금지:**
- 실패하는 테스트가 있는 상태로 진행하기
- 결과에 대한 테스트 검증 없이 머지하기
- 확인 없이 작업 삭제하기
- 명시적 요청 없이 force-push하기
- 머지 성공 확인 전에 worktree 제거하기
- 자신이 생성하지 않은 worktree 정리하기 (출처 확인 필요)
- worktree 내부에서 `git worktree remove` 실행하기

**항상 이행:**
- 옵션을 제시하기 전에 테스트 검증하기
- 메뉴를 표시하기 전에 환경 감지하기
- 정확히 4가지 옵션(detached HEAD의 경우 3가지) 제시하기
- 옵션 4에 대해 입력 확인 받기
- 옵션 1과 4에 대해서만 worktree 정리하기
- worktree 제거 전에 메인 저장소 루트로 `cd` 하기
- 제거 후 `git worktree prune` 실행하기
