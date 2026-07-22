# Codex App 호환성 구현 계획

> **에이전트 작업자 참고:** 필수 하위 스킬: 이 계획을 작업별로 구현하려면 superpowers:subagent-driven-development (권장) 또는 superpowers:executing-plans를 사용하세요. 각 단계는 추적을 위해 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** 기존 동작을 깨뜨리지 않으면서 Codex App의 샌드박스 처리된 워크트리 환경에서 `using-git-worktrees`, `finishing-a-development-branch` 및 관련 스킬이 잘 작동하도록 만듭니다.

**아키텍처:** 2개 스킬의 시작 부분에서 읽기 전용 환경 감지(`git-dir` 대 `git-common-dir`)를 수행합니다. 이미 연결된 워크트리에 있는 경우 생성을 건너뜁니다. 분리된 HEAD(detached HEAD) 상태인 경우 4가지 옵션 메뉴 대신 핸드오프 페이로드를 생성합니다. 샌드박스 폴백은 워크트리 생성 중 발생하는 권한 오류를 포착합니다.

**기술 스택:** Git, 마크다운 (스킬 파일은 지침 문서이며 실행 가능한 코드가 아님)

**스펙:** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 파일 구조

| 파일 | 책임 | 작업 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | 워크트리 생성 + 격리 | 0단계 감지 + 샌드박스 폴백 추가 |
| `skills/finishing-a-development-branch/SKILL.md` | 브랜치 마무리 워크플로 | 1.5단계 감지 + 정리 가드 추가 |
| `skills/subagent-driven-development/SKILL.md` | 하위 에이전트를 통한 계획 실행 | Integration 설명 업데이트 |
| `skills/executing-plans/SKILL.md` | 인라인 계획 실행 | Integration 설명 업데이트 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 플랫폼 참조 | 감지 + 마무리 문서 추가 |

---

### 작업 1: `using-git-worktrees`에 0단계 추가

**파일:**
- 수정: `skills/using-git-worktrees/SKILL.md:14-15` (Overview 다음, Directory Selection Process 앞에 삽입)

- [ ] **1단계: 현재 스킬 파일 읽기**

`skills/using-git-worktrees/SKILL.md` 전체를 읽습니다. 정확한 삽입 지점을 확인합니다: "Announce at start" 줄(14행) 다음 및 "## Directory Selection Process"(16행) 앞.

- [ ] **2단계: 0단계 섹션 삽입**

Overview 섹션과 "## Directory Selection Process" 사이에 다음 내용을 삽입합니다:

```markdown
## Step 0: Check if Already in an Isolated Workspace

Before creating a worktree, check if one already exists:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**If `GIT_DIR` differs from `GIT_COMMON`:** You are already inside a linked worktree (created by the Codex App, Claude Code's Agent tool, a previous skill run, or the user). Do NOT create another worktree. Instead:

1. Run project setup (auto-detect package manager as in "Run Project Setup" below)
2. Verify clean baseline (run tests as in "Verify Clean Baseline" below)
3. Report with branch state:
   - On a branch: "Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD: "Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

After reporting, STOP. Do not continue to Directory Selection or Creation Steps.

**If `GIT_DIR` equals `GIT_COMMON`:** Proceed with the full worktree creation flow below.

**Sandbox fallback:** If you proceed to Creation Steps but `git worktree add -b` fails with a permission error (e.g., "Operation not permitted"), treat this as a late-detected restricted environment. Fall back to the behavior above — run setup and baseline tests in the current directory, report accordingly, and STOP.
```

- [ ] **3단계: 삽입 검증**

파일을 다시 읽습니다. 확인 사항:
- 0단계가 Overview와 Directory Selection Process 사이에 표시됨
- 파일의 나머지 부분(Directory Selection, Safety Verification, Creation Steps 등)이 변경되지 않음
- 중복 섹션이나 깨진 마크다운이 없음

- [ ] **4단계: 커밋**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection (PRI-823)

Skip worktree creation when already in a linked worktree. Includes
sandbox fallback for permission errors on git worktree add."
```

---

### 작업 2: `using-git-worktrees` 통합 섹션 업데이트

**파일:**
- 수정: `skills/using-git-worktrees/SKILL.md:211-215` (Integration > Called by)

- [ ] **1단계: 3개의 "Called by" 항목 업데이트**

212-214행을 다음에서:

```markdown
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

다음으로 변경:

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **2단계: Integration 섹션 검증**

Integration 섹션을 읽습니다. 3개 항목이 모두 업데이트되었고, "Pairs with"는 변경되지 않았는지 확인합니다.

- [ ] **3단계: 커밋**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions (PRI-823)

Clarify that skill ensures a workspace exists, not that it always creates one."
```

---

### 작업 3: `finishing-a-development-branch`에 1.5단계 추가

**파일:**
- 수정: `skills/finishing-a-development-branch/SKILL.md:38` (1단계 다음, 2단계 앞에 삽입)

- [ ] **1단계: 현재 스킬 파일 읽기**

`skills/finishing-a-development-branch/SKILL.md` 전체를 읽습니다. 삽입 지점을 확인합니다: "**If tests pass:** Continue to Step 2."(38행) 다음 및 "### Step 2: Determine Base Branch"(40행) 앞.

- [ ] **2단계: 1.5단계 섹션 삽입**

1단계와 2단계 사이에 다음 내용을 삽입합니다:

```markdown
### Step 1.5: Detect Environment

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Path A — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` is empty (externally managed worktree, detached HEAD):**

First, ensure all work is staged and committed (`git add` + `git commit`).

Then present this to the user (do NOT present the 4-option menu):

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

Branch name: use ticket ID if available (e.g., `pri-823/codex-compat`), otherwise slugify the first 5 words of the plan title, otherwise omit. Avoid sensitive content in branch names.

Skip to Step 5 (cleanup is a no-op — see guard below).

**Path B — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` exists (externally managed worktree, named branch):**

Proceed to Step 2 and present the 4-option menu as normal.

**Path C — `GIT_DIR` equals `GIT_COMMON` (normal environment):**

Proceed to Step 2 and present the 4-option menu as normal.
```

- [ ] **3단계: 삽입 검증**

파일을 다시 읽습니다. 확인 사항:
- 1.5단계가 1단계와 2단계 사이에 표시됨
- 2~5단계는 변경되지 않음
- Path A 핸드오프에 커밋 SHA 및 데이터 손실 경고 포함
- Path B 및 C는 정 정상적으로 2단계로 진행됨

- [ ] **4단계: 커밋**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection (PRI-823)

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu. Includes commit SHA and data loss warning."
```

---

### 작업 4: `finishing-a-development-branch`에 5단계 정리 가드 추가

**파일:**
- 수정: `skills/finishing-a-development-branch/SKILL.md` (Step 5: Cleanup Worktree — 섹션 헤더로 찾기, 작업 3 이후 줄 번호가 이동됨)

- [ ] **1단계: 현재 5단계 섹션 읽기**

`skills/finishing-a-development-branch/SKILL.md`에서 "### Step 5: Cleanup Worktree" 섹션을 찾습니다(작업 3의 삽입 후 줄 번호가 이동됨). 현재 5단계는 다음과 같습니다:

```markdown
### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

- [ ] **2단계: 기존 로직 앞에 정리 가드 추가**

5단계 섹션을 다음으로 교체합니다:

```markdown
### Step 5: Cleanup Worktree

**First, check if worktree is externally managed:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

If `GIT_DIR` differs from `GIT_COMMON`: skip worktree removal — the host environment owns this workspace.

**Otherwise, for Options 1 and 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

참고: 원문에는 "For Options 1, 2, 4"라고 되어 있었으나 빠른 참조(Quick Reference) 테이블과 흔한 실수(Common Mistakes) 섹션에는 "Options 1 & 4 only"라고 나와 있습니다. 이 수정으로 5단계가 해당 섹션들과 일치하게 됩니다.

- [ ] **3단계: 교체 검증**

5단계를 읽습니다. 확인 사항:
- 정리 가드(재감지)가 처음에 표시됨
- 외부에서 관리되지 않는 워크트리에 대해 기존 제거 로직이 보존됨
- "Options 1 and 4"("1, 2, 4"가 아님)가 빠른 참조 및 흔한 실수와 일치함

- [ ] **4단계: 커밋**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard (PRI-823)

Re-detect externally managed worktree at cleanup time and skip removal.
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only, matching Quick Reference and Common Mistakes."
```

---

### 작업 5: `subagent-driven-development` 및 `executing-plans`의 Integration 항목 업데이트

**파일:**
- 수정: `skills/subagent-driven-development/SKILL.md:268`
- 수정: `skills/executing-plans/SKILL.md:68`

- [ ] **1단계: `subagent-driven-development` 업데이트**

268행을 다음에서:
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
다음으로 변경:
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **2단계: `executing-plans` 업데이트**

68행을 다음에서:
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
다음으로 변경:
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **3단계: 두 파일 검증**

`skills/subagent-driven-development/SKILL.md` 268행과 `skills/executing-plans/SKILL.md` 68행을 읽습니다. 두 파일 모두 "Ensures isolated workspace (creates one or verifies existing)"으로 기술되어 있는지 확인합니다.

- [ ] **4단계: 커밋**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): update worktree Integration descriptions (PRI-823)

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one."
```

---

### 작업 6: `codex-tools.md`에 환경 감지 문서 추가

**파일:**
- 수정: `skills/using-superpowers/references/codex-tools.md:25` (끝부분에 덧붙임)

- [ ] **1단계: 현재 파일 읽기**

`skills/using-superpowers/references/codex-tools.md` 전체를 읽습니다. multi_agent 섹션 다음인 25-26행에서 끝나는지 확인합니다.

- [ ] **2단계: 2개의 새 섹션 덧붙이기**

파일 끝에 다음 내용을 추가합니다:

```markdown

## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.

## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

- [ ] **3단계: 추가 사항 검증**

전체 파일을 읽습니다. 확인 사항:
- 기존 내용 뒤에 2개의 새 섹션이 표시됨
- Bash 코드 블록이 올바르게 렌더링됨 (이스케이프되지 않음)
- 0단계 및 1.5단계에 대한 교차 참조가 존재함

- [ ] **4단계: 커밋**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs (PRI-823)

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt."
```

---

### 작업 7: 자동화된 테스트 — 환경 감지

**파일:**
- 생성: `tests/codex-app-compat/test-environment-detection.sh`

- [ ] **1단계: 테스트 디렉터리 생성**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **2단계: 감지 테스트 스크립트 작성**

`tests/codex-app-compat/test-environment-detection.sh` 생성:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Test environment detection logic from PRI-823
# Tests the git-dir vs git-common-dir comparison used by
# using-git-worktrees Step 0 and finishing-a-development-branch Step 1.5

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# Helper: run detection and return "linked" or "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== Test 1: Normal repo detection ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Normal repo detected as normal"
else
  log_fail "Normal repo detected as '$result' (expected 'normal')"
fi

echo "=== Test 2: Linked worktree detection ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree detected as linked"
else
  log_fail "Linked worktree detected as '$result' (expected 'linked')"
fi

echo "=== Test 3: Detached HEAD detection ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch is empty"
else
  log_fail "Detached HEAD: branch is '$branch' (expected empty)"
fi

echo "=== Test 4: Linked worktree + detached HEAD (Codex App simulation) ==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App simulation: linked + detached HEAD"
else
  log_fail "Codex App simulation: result='$result', branch='$branch'"
fi

echo "=== Test 5: Cleanup guard — linked worktree should NOT remove ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Cleanup guard: linked worktree correctly detected (would skip removal)"
else
  log_fail "Cleanup guard: expected 'linked', got '$result'"
fi

echo "=== Test 6: Cleanup guard — main repo SHOULD remove ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Cleanup guard: main repo correctly detected (would proceed with removal)"
else
  log_fail "Cleanup guard: expected 'normal', got '$result'"
fi

# Cleanup worktree before temp dir removal
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== Results: $PASS passed, $FAIL failed ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **3단계: 실행 권한 부여 및 실행**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

예상 출력: 6 passed, 0 failed.

- [ ] **4단계: 커밋**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat (PRI-823)

Tests git-dir vs git-common-dir comparison in normal repo, linked
worktree, detached HEAD, and cleanup guard scenarios."
```

---

### 작업 8: 최종 검증

**파일:**
- 읽기: 수정된 스킬 파일 5개 전체

- [ ] **1단계: 자동화된 감지 테스트 실행**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

예상: 6 passed, 0 failed.

- [ ] **2단계: 수정된 각 파일 읽기 및 변경 사항 확인**

각 파일을 처음부터 끝까지 읽습니다:
- `skills/using-git-worktrees/SKILL.md` — 0단계 존재, 나머지는 미변경
- `skills/finishing-a-development-branch/SKILL.md` — 1.5단계 존재, 정리 가드 존재, 나머지는 미변경
- `skills/subagent-driven-development/SKILL.md` — 268행 업데이트됨
- `skills/executing-plans/SKILL.md` — 68행 업데이트됨
- `skills/using-superpowers/references/codex-tools.md` — 끝에 2개 새 섹션 추가됨

- [ ] **3단계: 의도치 않은 변경 사항이 없는지 확인**

```bash
git diff --stat HEAD~7
```

정확히 6개 파일이 변경(스킬 파일 5개 + 테스트 파일 1개)된 것으로 나타나야 합니다. 다른 파일은 수정되지 않습니다.

- [ ] **4단계: 기존 테스트 스위트 실행**

테스트 러너가 존재하는 경우:
```bash
# 스킬 트리거링 테스트 실행
# 참고: tests/skill-triggering/은 2026-05-06에 drill 시나리오로 이전되었습니다.
# evals/scenarios/triggering-*.yaml을 참조하세요. 아래 참조는 과거 기록 아티팩트입니다.
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "Skill triggering tests not available in this environment"

# SDD 통합 테스트 실행
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "SDD integration test not available in this environment"
```

참고: 이 테스트에는 `--dangerously-skip-permissions` 옵션이 설정된 Claude Code가 필요합니다. 사용할 수 없는 경우 회귀 테스트를 수동으로 실행하도록 문서화하세요.
