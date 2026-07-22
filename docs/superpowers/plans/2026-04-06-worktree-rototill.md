# 워크트리 재정비(Worktree Rototill) 구현 계획

> **에이전트 작업자 참고:** 필수 서브 스킬: 이 계획을 작업별로 구현하려면 superpowers:subagent-driven-development (권장) 또는 superpowers:executing-plans를 사용하세요. 추적을 위해 단계에 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** superpowers가 사용 가능한 경우 네이티브 하네스 워크트리 시스템에 위임하고, 사용 불가능한 경우 수동 git 워크트리로 폴백하도록 하며, 알려진 세 가지 마무리 버그를 수정합니다.

**아키텍처:** 두 개의 스킬 파일이 재작성되고(`using-git-worktrees`, `finishing-a-development-branch`), 세 개의 파일이 한 줄의 통합 업데이트를 받습니다(`executing-plans`, `subagent-driven-development`, `writing-plans`). 핵심 변경 사항은 감지 기능(`GIT_DIR != GIT_COMMON`) 및 네이티브 도구 우선 생성 경로를 추가하는 것입니다. 이 파일들은 애플리케이션 코드가 아닌 마크다운 스킬 지침 파일이며, "테스트"는 testing-skills-with-subagents TDD 프레임워크를 사용하는 에이전트 동작 테스트입니다.

**기술 스택:** 마크다운 (스킬 파일), bash (테스트 스크립트), Claude Code CLI (헤드리스 테스트용 `claude -p`)

**명세:** `docs/superpowers/specs/2026-04-06-worktree-rototill-design.md`

---

### 작업 1: GATE — 단계 1a (네이티브 도구 선호) TDD 검증

단계 1a는 전체 설계의 핵심 전제입니다. 에이전트가 `git worktree add`보다 네이티브 워크트리 도구를 선호하지 않으면 명세는 실패합니다. 스킬 파일을 수정하기 전에 이 내용을 가장 먼저 검증하세요.

**파일:**
- 생성: `tests/claude-code/test-worktree-native-preference.sh`
- 읽기: `skills/using-git-worktrees/SKILL.md` (현재 버전, RED 베이스라인용)
- 읽기: `tests/claude-code/test-helpers.sh` (`run_claude`, `assert_contains` 등용)
- 읽기: `skills/writing-skills/testing-skills-with-subagents.md` (TDD 프레임워크)

**이 작업은 게이트(gate)입니다.** 2회의 REFACTOR 반복 후에도 GREEN 단계가 실패하면 중단(STOP)하세요. 작업 2로 진행하지 마세요. 보고 후 생성 방식 재설계가 필요합니다.

- [ ] **단계 1: RED 베이스라인 테스트 스크립트 작성**

업데이트된 스킬 텍스트가 없는 경우와 있는 경우 모두에서 시나리오를 실행할 테스트 스크립트를 생성합니다. RED 단계는 현재 스킬(단계 1a가 없음)을 대상으로 실행됩니다.

```bash
#!/usr/bin/env bash
# Test: Does the agent prefer native worktree tools (EnterWorktree) over git worktree add?
# Framework: RED-GREEN-REFACTOR per testing-skills-with-subagents.md
#
# RED:   Current skill has no native tool preference. Agent should use git worktree add.
# GREEN: Updated skill has Step 1a. Agent should use EnterWorktree on Claude Code.

set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# Pressure scenario: realistic implementation task where agent needs isolation
SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to implement a small feature (add a "version" field to package.json).
This should be done in an isolated workspace to protect the main branch.

You have the using-git-worktrees skill available. Set up the isolated workspace now.
Do NOT actually implement the feature — just set up the workspace and report what you did.

Respond with EXACTLY what tool/command you used to create the workspace.'

echo "=== Worktree Native Preference Test ==="
echo ""

# Phase selection
PHASE="${1:-red}"

if [ "$PHASE" = "red" ]; then
    echo "--- RED PHASE: Running WITHOUT Step 1a (current skill) ---"
    echo "Expected: Agent uses 'git worktree add' (no native tool awareness)"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # RED expectation: agent uses git worktree add (current behavior)
    if echo "$output" | grep -qi "EnterWorktree"; then
        echo "[UNEXPECTED] Agent used EnterWorktree WITHOUT Step 1a — skill may not be needed"
        echo "Investigate: is Claude Code's default behavior already correct?"
    else
        echo "[RED CONFIRMED] Agent did NOT use EnterWorktree"
        assert_contains "$output" "git worktree" "Agent used git worktree (expected in RED)"
    fi

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "green" ]; then
    echo "--- GREEN PHASE: Running WITH Step 1a (updated skill) ---"
    echo "Expected: Agent uses EnterWorktree instead of git worktree add"
    echo ""

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"

    output=$(run_claude "$SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # GREEN expectation: agent uses EnterWorktree
    assert_contains "$output" "EnterWorktree" "Agent used native EnterWorktree tool"
    assert_not_contains "$output" "git worktree add" "Agent did NOT fall back to git worktree add"

    cleanup_test_project "$test_dir"

elif [ "$PHASE" = "pressure" ]; then
    echo "--- PRESSURE PHASE: Multiple pressures to skip native tools ---"
    echo ""

    PRESSURE_SCENARIO='IMPORTANT: This is a real task. Choose and act.

You need to set up an isolated workspace URGENTLY. Production is impacted and
the fix needs to go in a worktree. You know git worktree add works reliably.
The .worktrees/ directory already exists and is gitignored.

You have the using-git-worktrees skill. Set up the workspace NOW.
Speed matters. Report EXACTLY what tool/command you used.'

    test_dir=$(create_test_project)
    cd "$test_dir"
    git init && git commit --allow-empty -m "init"
    mkdir -p .worktrees
    echo ".worktrees/" >> .gitignore

    output=$(run_claude "$PRESSURE_SCENARIO" 120)

    echo "Agent output:"
    echo "$output"
    echo ""

    # Should STILL use EnterWorktree even under pressure
    assert_contains "$output" "EnterWorktree" "Agent used native tool even under time pressure"
    assert_not_contains "$output" "git worktree add" "Agent resisted falling back to git despite pressure"

    cleanup_test_project "$test_dir"
fi

echo ""
echo "=== Test Complete ==="
```

- [ ] **단계 2: RED 단계 실행 — 현재 에이전트가 git worktree add를 사용하는지 확인**

실행: `cd tests/claude-code && bash test-worktree-native-preference.sh red`

예상 결과: `[RED CONFIRMED] Agent did NOT use EnterWorktree` — 현재 스킬에 네이티브 도구 선호가 없으므로 에이전트가 `git worktree add`를 사용함.

에이전트의 정확한 출력과 합리화 문구를 그대로 기록하세요. 이것이 스킬이 수정해야 하는 베이스라인 실패입니다.

- [ ] **단계 3: RED가 확인되면 진행. 단계 1a 스킬 텍스트 작성.**

변수를 격리하기 위해 단계 1a만 추가된 임시 테스트 버전의 스킬을 생성합니다. 기존 디렉터리 선택 프로세스 이전의 스킬 생성 지침 상단에 이 섹션을 추가합니다:

```markdown
## Step 1: Create Isolated Workspace

**You have two mechanisms. Try them in this order.**

### 1a. Native Worktree Tools (preferred)

If your platform provides a worktree or workspace-isolation tool, use it. You know your own toolkit — the skill does not need to name specific tools. Native tools handle directory placement, branch creation, and cleanup automatically.

After using a native tool, skip to Step 3 (Project Setup).

### 1b. Git Worktree Fallback

If no native tool is available, create a worktree manually using git.
```

- [ ] **단계 4: GREEN 단계 실행 — 에이전트가 이제 EnterWorktree를 사용하는지 확인**

실행: `cd tests/claude-code && bash test-worktree-native-preference.sh green`

예상 결과: `[PASS] Agent used native EnterWorktree tool`

실패 시: 에이전트의 정확한 출력과 합리화 문구를 기록하세요. 이는 REFACTOR 신호입니다 — 단계 1a 텍스트 수정이 필요합니다. 최대 2회의 REFACTOR 반복을 시도하세요. 2회 반복 후에도 여전히 실패하면 중단하고 보고하세요.

- [ ] **단계 5: PRESSURE 단계 실행 — 압박 상황에서도 에이전트가 폴백에 저항하는지 확인**

실행: `cd tests/claude-code && bash test-worktree-native-preference.sh pressure`

예상 결과: `[PASS] Agent used native tool even under time pressure`

실패 시: 합리화 문구를 그대로 기록하세요. 단계 1a 텍스트에 명시적인 대응책을 추가하세요(예: 레드 플래그 항목: "플랫폼이 네이티브 워크트리 도구를 제공할 때는 절대로 git worktree add를 사용하지 마십시오"). 재실행하세요.

- [ ] **단계 6: 테스트 스크립트 커밋**

```bash
git add tests/claude-code/test-worktree-native-preference.sh
git commit -m "test: add RED/GREEN validation for native worktree preference (PRI-974)

Gate test for Step 1a — validates agents prefer EnterWorktree over
git worktree add on Claude Code. Must pass before skill rewrite."
```

---

### 작업 2: `using-git-worktrees` SKILL.md 재작성

생성 스킬 전체 재작성. 기존 파일을 완전히 교체합니다.

**파일:**
- 수정: `skills/using-git-worktrees/SKILL.md` (전체 재작성, 219줄 → ~210줄)

**의존성:** 작업 1 GREEN 통과.

- [ ] **단계 1: 완성된 새 SKILL.md 작성**

`skills/using-git-worktrees/SKILL.md`의 전체 내용을 다음으로 교체합니다:

```markdown
---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - ensures an isolated workspace exists via native tools or git worktree fallback
---

# Using Git Worktrees

## Overview

Ensure work happens in an isolated workspace. Prefer your platform's native worktree tools. Fall back to manual git worktrees only when no native tool is available.

**Core principle:** Detect existing isolation first. Then use native tools. Then fall back to git. Never fight the harness.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Step 0: Detect Existing Isolation

**Before creating anything, check if you are already in an isolated workspace.**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Submodule guard:** `GIT_DIR != GIT_COMMON` is also true inside git submodules. Before concluding "already in a worktree," verify you are not in a submodule:

```bash
# If this returns a path, you're in a submodule, not a worktree — proceed to Step 1
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**If `GIT_DIR != GIT_COMMON` (and not a submodule):** You are already in a linked worktree. Skip to Step 3 (Project Setup). Do NOT create another worktree.

Report with branch state:
- On a branch: "Already in isolated workspace at `<path>` on branch `<name>`."
- Detached HEAD: "Already in isolated workspace at `<path>` (detached HEAD, externally managed). Branch creation needed at finish time."

**If `GIT_DIR == GIT_COMMON` (or in a submodule):** You are in a normal repo checkout.

Has the user already indicated their worktree preference in your instructions? If not, ask for consent before creating a worktree:

> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

Honor any existing declared preference without asking. If the user declines consent, work in place and skip to Step 3.

## Step 1: Create Isolated Workspace

**You have two mechanisms. Try them in this order.**

### 1a. Native Worktree Tools (preferred)

If your platform provides a worktree or workspace-isolation tool, use it. You know your own toolkit — the skill does not need to name specific tools. Native tools handle directory placement, branch creation, and cleanup automatically.

After using a native tool, skip to Step 3 (Project Setup).

### 1b. Git Worktree Fallback

If no native tool is available, create a worktree manually using git.

#### Directory Selection

Follow this priority order:

1. **Check your instructions for a worktree directory preference.** If specified, use it without asking.

2. **Check existing project-local directories:**
   ```bash
   ls -d .worktrees 2>/dev/null     # Preferred (hidden)
   ls -d worktrees 2>/dev/null      # Alternative
   ```
   If found, use that directory. If both exist, `.worktrees` wins.

3. **Default to `.worktrees/`.**

#### Safety Verification (project-local directories only)

**MUST verify directory is ignored before creating worktree:**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**If NOT ignored:** Add to .gitignore, commit the change, then proceed.

**Why critical:** Prevents accidentally committing worktree contents to repository.

#### Create the Worktree

```bash
# Determine path based on chosen location
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

#### Hooks Awareness

Git worktrees do not inherit the parent repo's hooks directory. After creating the worktree, symlink hooks from the main repo if they exist:

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

This prevents pre-commit checks, linters, and other hooks from silently stopping when work moves to a worktree.

**Sandbox fallback:** If `git worktree add` fails with a permission error (sandbox denial), treat this as a restricted environment. Skip creation, run setup and baseline tests in the current directory, report accordingly.

## Step 3: Project Setup

Auto-detect and run appropriate setup:

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

## Step 4: Verify Clean Baseline

Run tests to ensure workspace starts clean:

```bash
# Use project-appropriate command
npm test / cargo test / pytest / go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### Report

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Already in linked worktree | Skip creation (Step 0) |
| In a submodule | Treat as normal repo (Step 0 guard) |
| Native worktree tool available | Use it (Step 1a) |
| No native tool | Git worktree fallback (Step 1b) |
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check instruction file, then default `.worktrees/` |
| Directory not ignored | Add to .gitignore + commit |
| Permission error on create | Sandbox fallback, work in place |
| Tests fail during baseline | Report failures + ask |
| No package.json/Cargo.toml | Skip dependency install |

## Common Mistakes

### Fighting the harness

- **Problem:** Using `git worktree add` when the platform already provides isolation
- **Fix:** Step 0 detects existing isolation. Step 1a defers to native tools.

### Skipping detection

- **Problem:** Creating a nested worktree inside an existing one
- **Fix:** Always run Step 0 before creating anything

### Skipping ignore verification

- **Problem:** Worktree contents get tracked, pollute git status
- **Fix:** Always use `git check-ignore` before creating project-local worktree

### Assuming directory location

- **Problem:** Creates inconsistency, violates project conventions
- **Fix:** Follow priority: existing > instruction file > default

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

## Red Flags

**Never:**
- Create a worktree when Step 0 detects existing isolation
- Use git commands when a native worktree tool is available
- Create worktree without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking

**Always:**
- Run Step 0 detection first
- Prefer native tools over git fallback
- Follow directory priority: existing > instruction file > default
- Verify directory is ignored for project-local
- Auto-detect and run project setup
- Verify clean test baseline
- Symlink hooks after creating worktree via 1b

## Integration

**Called by:**
- **subagent-driven-development** - Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - Ensures isolated workspace (creates one or verifies existing)
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
```

- [ ] **단계 2: 파일이 올바르게 읽히는지 검증**

실행: `wc -l skills/using-git-worktrees/SKILL.md`

예상 결과: 약 200-220줄. 마크다운 서식 오류가 있는지 스캔하세요.

- [ ] **단계 3: 커밋**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat: rewrite using-git-worktrees with detect-and-defer (PRI-974)

Step 0: GIT_DIR != GIT_COMMON detection (skip if already isolated)
Step 0 consent: opt-in prompt before creating worktree (#991)
Step 1a: native tool preference (short, first, declarative)
Step 1b: git worktree fallback with project-local directory policy
Submodule guard prevents false detection
Platform-neutral instruction file references (#1049)"
```

---

### 작업 3: `finishing-a-development-branch` SKILL.md 재작성

마무리 스킬 전체 재작성. 환경 감지 추가, 세 가지 버그 수정, 출처(provenance) 기반 정리 추가.

**파일:**
- 수정: `skills/finishing-a-development-branch/SKILL.md` (전체 재작성, 201줄 → ~220줄)

- [ ] **단계 1: 완성된 새 SKILL.md 작성**

`skills/finishing-a-development-branch/SKILL.md`의 전체 내용을 다음으로 교체합니다:

```markdown
---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion of development work by presenting structured options for merge, PR, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Detect environment → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Detect Environment

**Determine workspace state before presenting options:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

This determines which menu to show and how cleanup works:

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (normal repo) | Standard 4 options | No worktree to clean up |
| `GIT_DIR != GIT_COMMON`, named branch | Standard 4 options | Provenance-based (see Step 6) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | Reduced 3 options (no merge) | No cleanup (externally managed) |

### Step 3: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 4: Present Options

**Normal repo and named-branch worktree — present exactly these 4 options:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD — present exactly these 3 options:**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**Don't add explanation** - keep options concise.

### Step 5: Execute Choice

#### Option 1: Merge Locally

```bash
# Get main repo root for CWD safety
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Merge first — verify success before removing anything
git checkout <base-branch>
git pull
git merge <feature-branch>

# Verify tests on merged result
<test command>

# Only after merge succeeds: remove worktree, then delete branch
# (See Step 6 for worktree cleanup)
git branch -d <feature-branch>
```

Then: Cleanup worktree (Step 6)

#### Option 2: Push and Create PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**Do NOT clean up worktree** — user needs it alive to iterate on PR feedback.

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

Then: Cleanup worktree (Step 6), then force-delete branch:
```bash
git branch -D <feature-branch>
```

### Step 6: Cleanup Workspace

**Only runs for Options 1 and 4.** Options 2 and 3 always preserve the worktree.

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**If `GIT_DIR == GIT_COMMON`:** Normal repo, no worktree to clean up. Done.

**If worktree path is under `.worktrees/` or `worktrees/`:** Superpowers created this worktree — we own cleanup.

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # Self-healing: clean up any stale registrations
```

**Otherwise:** The host environment (harness) owns this workspace. Do NOT remove it. If your platform provides a workspace-exit tool, use it. Otherwise, leave the workspace in place.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | yes | - | - | yes |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | yes (force) |

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Open-ended questions**
- **Problem:** "What should I do next?" is ambiguous
- **Fix:** Present exactly 4 structured options (or 3 for detached HEAD)

**Cleaning up worktree for Option 2**
- **Problem:** Remove worktree user needs for PR iteration
- **Fix:** Only cleanup for Options 1 and 4

**Deleting branch before removing worktree**
- **Problem:** `git branch -d` fails because worktree still references the branch
- **Fix:** Merge first, remove worktree, then delete branch

**Running git worktree remove from inside the worktree**
- **Problem:** Command fails silently when CWD is inside the worktree being removed
- **Fix:** Always `cd` to main repo root before `git worktree remove`

**Cleaning up harness-owned worktrees**
- **Problem:** Removing a worktree the harness created causes phantom state
- **Fix:** Only clean up worktrees under `.worktrees/` or `worktrees/`

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request
- Remove a worktree before confirming merge success
- Clean up worktrees you didn't create (provenance check)
- Run `git worktree remove` from inside the worktree

**Always:**
- Verify tests before offering options
- Detect environment before presenting menu
- Present exactly 4 options (or 3 for detached HEAD)
- Get typed confirmation for Option 4
- Clean up worktree for Options 1 & 4 only
- `cd` to main repo root before worktree removal
- Run `git worktree prune` after removal

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill
```

- [ ] **단계 2: 파일이 올바르게 읽히는지 검증**

실행: `wc -l skills/finishing-a-development-branch/SKILL.md`

예상 결과: 약 210-230줄.

- [ ] **단계 3: 커밋**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat: rewrite finishing-a-development-branch with detect-and-defer (PRI-974)

Step 2: environment detection (GIT_DIR != GIT_COMMON) before presenting menu
Detached HEAD: reduced 3-option menu (no merge from detached HEAD)
Provenance-based cleanup: .worktrees/ = ours, anything else = hands off
Bug #940: Option 2 no longer cleans up worktree
Bug #999: merge -> verify -> remove worktree -> delete branch
Bug #238: cd to main repo root before git worktree remove
Stale worktree pruning after removal (git worktree prune)"
```

---

### 작업 4: 통합 업데이트

using-git-worktrees를 참조하는 세 파일에 대한 한 줄 변경 사항.

**파일:**
- 수정: `skills/executing-plans/SKILL.md:68`
- 수정: `skills/subagent-driven-development/SKILL.md:268`
- 수정: `skills/writing-plans/SKILL.md:16`

- [ ] **단계 1: executing-plans 통합 줄 업데이트**

`skills/executing-plans/SKILL.md`에서 68번째 줄을 다음에서:

```markdown
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

다음으로 변경:

```markdown
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **단계 2: subagent-driven-development 통합 줄 업데이트**

`skills/subagent-driven-development/SKILL.md`에서 268번째 줄을 다음에서:

```markdown
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```

다음으로 변경:

```markdown
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **단계 3: writing-plans 컨텍스트 줄 업데이트**

`skills/writing-plans/SKILL.md`에서 16번째 줄을 다음에서:

```markdown
**Context:** This should be run in a dedicated worktree (created by brainstorming skill).
```

다음으로 변경:

```markdown
**Context:** If working in an isolated worktree, it should have been created via the using-git-worktrees skill at execution time.
```

- [ ] **단계 4: 세 파일 모두 커밋**

```bash
git add skills/executing-plans/SKILL.md skills/subagent-driven-development/SKILL.md skills/writing-plans/SKILL.md
git commit -m "fix: update worktree integration references across skills (PRI-974)

Remove REQUIRED language from executing-plans and subagent-driven-development.
Consent and detection now live inside using-git-worktrees itself.
Fix stale 'created by brainstorming' claim in writing-plans."
```

---

### 작업 5: 엔드투엔드 검증

재작성된 전체 스킬이 함께 잘 작동하는지 검증합니다. 기존 테스트 수트 실행 및 수동 검증을 진행합니다.

**파일:**
- 읽기: `tests/claude-code/run-skill-tests.sh`
- 읽기: `skills/using-git-worktrees/SKILL.md` (최종 상태 검증)
- 읽기: `skills/finishing-a-development-branch/SKILL.md` (최종 상태 검증)

- [ ] **단계 1: 기존 테스트 수트 실행**

실행: `cd tests/claude-code && bash run-skill-tests.sh`

예상 결과: 모든 기존 테스트 통과. 실패하는 항목이 있으면 조사하세요 — 통합 변경 사항(작업 4)이 콘텐츠 단정문을 깨뜨렸을 수 있습니다.

- [ ] **단계 2: 단계 1a GREEN 테스트 재실행**

실행: `cd tests/claude-code && bash test-worktree-native-preference.sh green`

예상 결과: 통과(PASS) — 에이전트가 최종 스킬 텍스트(작업 1의 최소 단계 1a 추가뿐만 아니라)로도 여전히 EnterWorktree를 사용함.

- [ ] **단계 3: 수동 검증 — 재작성된 두 스킬 모두 처음부터 끝까지 읽기**

`skills/using-git-worktrees/SKILL.md` 및 `skills/finishing-a-development-branch/SKILL.md` 전체를 읽으세요. 다음을 확인합니다:

1. 이전 동작에 대한 참조 없음(하드코딩된 `CLAUDE.md`, 대화형 디렉터리 프롬프트, "REQUIRED" 언어)
2. 각 파일 내 단계 번호 지정이 일관됨
3. 빠른 참조(Quick Reference) 표가 본문과 일치함
4. 통합 섹션 상호 참조가 올바름
5. 마크다운 서식 문제 없음

- [ ] **단계 4: git status가 깨끗한지 검증**

실행: `git status`

예상 결과: 깨끗한 작업 트리. 작업 1-4 전반의 모든 변경 사항이 커밋됨.

- [ ] **단계 5: 수정이 필요한 경우 최종 커밋**

수동 검증에서 문제를 발견한 경우 수정하고 커밋하세요:

```bash
git add -A
git commit -m "fix: address review findings in worktree skill rewrite (PRI-974)"
```

문제가 발견되지 않으면 이 단계를 건너뜁니다.
