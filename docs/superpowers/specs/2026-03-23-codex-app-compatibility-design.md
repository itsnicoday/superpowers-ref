# Codex App 호환성: 워크트리 및 Finishing 스킬 적응

기존 Claude Code 또는 Codex CLI 동작을 깨뜨리지 않으면서 Codex App의 샌드박스 처리된 워크트리 환경에서 superpowers 스킬이 동작하도록 만듭니다.

**티켓:** PRI-823

## 동기

Codex App은 자체가 관리하는 git 워크트리 내에서 에이전트를 실행합니다 — detached HEAD 상태, `$CODEX_HOME/worktrees/` 아래 위치하며, `git checkout -b`, `git push`, 그리고 네트워크 접근을 차단하는 Seatbelt 샌드박스가 적용됩니다. 3개의 superpowers 스킬이 제약 없는 git 접근을 가정하고 있습니다: `using-git-worktrees`는 이름이 지정된 브랜치로 수동 워크트리를 생성하고, `finishing-a-development-branch`는 브랜치 이름으로 머지/푸시/PR을 수행하며, `subagent-driven-development`는 두 작업 모두를 필요로 합니다.

Codex CLI(오픈 소스 터미널 도구)는 이러한 충돌이 **없습니다** — 내장된 워크트리 관리 기능이 없습니다. 우리의 수동 워크트리 접근 방식은 거기서 격리 공백을 채워줍니다. 문제는 전적으로 Codex App에 국한됩니다.

## 실증적 조사 결과 (Empirical Findings)

2026-03-23에 Codex App에서 테스트됨:

| Operation | workspace-write sandbox | Full access sandbox |
|---|---|---|
| `git add` | Works | Works |
| `git commit` | Works | Works |
| `git checkout -b` | **Blocked** (`.git/refs/heads/` 작성 불가) | Works |
| `git push` | **Blocked** (네트워크 + `.git/refs/remotes/`) | Works |
| `gh pr create` | **Blocked** (네트워크) | Works |
| `git status/diff/log` | Works | Works |
|

추가 조사 결과:
- `spawn_agent` 서브에이전트는 상위 스레드의 파일시스템을 **공유함** (마커 파일 테스트를 통해 확인됨)
- 어떤 브랜치에서 워크트리가 시작되었든 관계없이 App 헤더에 "Create branch" 버튼이 나타남
- App의 네이티브 finishing 흐름: Create branch → Commit 모달 → Commit and push / Commit and create PR
- `network_access = true` 설정이 macOS에서 무음으로 오작동함 (이슈 #10390)

## 설계: 읽기 전용 환경 감지

세 가지 읽기 전용 git 명령어는 부작용 없이 환경을 감지합니다:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

도출되는 두 가지 신호:

- **IN_LINKED_WORKTREE:** `GIT_DIR != GIT_COMMON` — 에이전트가 다른 무언가(Codex App, Claude Code Agent 도구, 이전 스킬 실행, 또는 사용자)에 의해 생성된 워크트리 내부에 있음
- **ON_DETACHED_HEAD:** `BRANCH`가 비어 있음 — 이름이 지정된 브랜치가 존재하지 않음

`show-toplevel`을 검사하는 대신 `git-dir != git-common-dir`을 사용하는 이유:
- 일반 저장소에서는 둘 다 동일한 `.git` 디렉토리로 해석됨
- 연결된 워크트리에서는 `git-dir`이 `.git/worktrees/<name>`인 반면 `git-common-dir`은 `.git`임
- 서브모듈에서는 둘 다 동일함 — `show-toplevel`이 유발할 오탐을 방지함
- `cd && pwd -P`를 통해 해석하면 상대 경로 문제(일반 저장소에서 `git-common-dir`은 상대적인 `.git`을 반환하지만 워크트리에서는 절대 경로를 반환함) 및 심볼릭 링크(macOS `/tmp` → `/private/tmp`)를 다룰 수 있음

### 의사결정 매트릭스 (Decision Matrix)

| Linked Worktree? | Detached HEAD? | Environment | Action |
|---|---|---|---|
| No | No | Claude Code / Codex CLI / 일반 git | 전체 스킬 동작 (변경 없음) |
| Yes | Yes | Codex App 워크트리 (workspace-write) | 워크트리 생성 건너뜀; 완료 시 핸드오프 페이로드 출력 |
| Yes | No | Codex App (Full access) 또는 수동 워크트리 | 워크트리 생성 건너뜀; 전체 finishing 흐름 수행 |
| No | Yes | 특이 케이스 (수동 detached HEAD) | 워크트리를 정상 생성; 완료 시 경고 |

## 변경 사항

### 1. `using-git-worktrees/SKILL.md` — Step 0 추가 (~12줄)

"Overview"와 "Directory Selection Process" 사이의 새로운 섹션:

**Step 0: 이미 격리된 작업 공간에 있는지 확인**

감지 명령어를 실행합니다. 만약 `GIT_DIR != GIT_COMMON`이면 워크트리 생성을 완전히 건너뜁니다. 대신:
1. Creation Steps 아래의 "Run Project Setup" 서브섹션으로 이동 — `npm install` 등은 멱등성을 가지므로 안전을 위해 실행할 가치가 있음
2. 그 후 "Verify Clean Baseline" 진행 — 테스트 실행
3. 브랜치 상태와 함께 보고:
   - 브랜치 상에 있음: "Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD 상태: "Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

만약 `GIT_DIR == GIT_COMMON`이면 전체 워크트리 생성 흐름을 계속 진행합니다 (변경 없음).

안전성 검증(.gitignore 검사)은 Step 0이 실행될 때 건너뜁니다 — 외부에서 생성된 워크트리에는 해당하지 않음.

Integration 섹션의 "Called by" 항목들을 업데이트합니다. 각 서브섹션의 설명을 컨텍스트별 텍스트에서 다음과 같이 변경합니다: "Ensures isolated workspace (creates one or verifies existing)". 예를 들어 `subagent-driven-development` 항목은 "REQUIRED: Set up isolated workspace before starting"에서 "REQUIRED: Ensures isolated workspace (creates one or verifies existing)"로 변경됩니다.

**샌드박스 폴백:** `GIT_DIR == GIT_COMMON`이고 스킬이 Creation Steps로 진행했지만, `git worktree add -b`가 권한 오류(예: Seatbelt 샌드박스 거부)로 실패하면 이를 늦게 감지된 제한적 환경으로 취급합니다. Step 0의 "이미 작업 공간에 있음" 동작으로 폴백합니다 — 생성을 건너뛰고, 현재 디렉토리에서 설정 및 베이스라인 테스트를 실행하며, 그에 맞게 보고합니다.

Step 0에서 보고한 후 중단합니다. Directory Selection이나 Creation Steps로 계속 진행하지 마십시오.

**그 외 모든 항목 변경 없음:** Directory Selection, Safety Verification, Creation Steps, Project Setup, Baseline Tests, Quick Reference, Common Mistakes, Red Flags.

### 2. `finishing-a-development-branch/SKILL.md` — Step 1.5 + 정리 가드 추가 (~20줄)

**Step 1.5: 환경 감지** (Step 1 "Verify Tests" 뒤, Step 2 "Determine Base Branch" 전)

감지 명령어를 실행합니다. 세 가지 경로:

- **Path A**는 Step 2와 3을 완전히 건너뜁니다 (베이스 브랜치나 옵션이 필요 없음).
- **Path B 및 C**는 정상적으로 Step 2 (Determine Base Branch) 및 Step 3 (Present Options)를 통해 진행됩니다.

**Path A — 외부에서 관리되는 워크트리 + detached HEAD** (`GIT_DIR != GIT_COMMON` 그리고 `BRANCH`가 비어 있음):

먼저, 모든 작업이 스테이징되고 커밋되었는지 확인합니다 (`git add` + `git commit`). Codex App의 finishing 제어 기능은 커밋된 작업에 대해 작동합니다.

그 후 사용자에게 다음을 제시합니다 (4개 옵션 메뉴를 제시하지 마십시오):

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

브랜치 이름 도출: 가능한 경우 티켓 ID를 사용(예: `pri-823/codex-compat`), 그렇지 않으면 플랜 제목의 첫 5개 단어를 슬러그화, 그것도 안 되면 제안을 생략합니다. 브랜치 이름에 민감한 내용(취약점 설명, 고객 이름)이 포함되지 않도록 피합니다.

Step 5로 건너뜁니다 (외부에서 관리되는 워크트리의 경우 정리는 아무 작업도 하지 않음).

**Path B — 외부에서 관리되는 워크트리 + 이름이 지정된 브랜치** (`GIT_DIR != GIT_COMMON` 그리고 `BRANCH`가 존재함):

정상적으로 4개 옵션 메뉴를 제시합니다. (Step 5 정리 가드가 외부에서 관리되는 상태를 독립적으로 재감지할 것입니다.)

**Path C — 일반 환경** (`GIT_DIR == GIT_COMMON`):

오늘날과 동일하게 4개 옵션 메뉴를 제시합니다 (변경 없음).

**Step 5 정리 가드:**

정리 시점에 `GIT_DIR` 대 `GIT_COMMON` 감지를 재실행합니다 (이전 스킬 출력에 의존하지 마십시오 — finishing 스킬은 다른 세션에서 실행될 수 있음). 만약 `GIT_DIR != GIT_COMMON`이면 `git worktree remove`를 건너뜁니다 — 호스트 환경이 이 작업 공간을 소유합니다.

그렇지 않으면 오늘날과 같이 검사하고 제거합니다. 참고: 기존 Step 5 텍스트는 "For Options 1, 2, 4"라고 되어 있지만 Quick Reference 테이블 및 Common Mistakes 섹션은 "Options 1 & 4 only"라고 되어 있습니다. 새 가드는 기존 로직 전에 추가되며 어떤 옵션이 정리를 트리거하는지 변경하지 않습니다.

**그 외 모든 항목 변경 없음:** Options 1-4 로직, Quick Reference, Common Mistakes, Red Flags.

### 3. `subagent-driven-development/SKILL.md` 및 `executing-plans/SKILL.md` — 각각 1줄 편집

두 스킬 모두 동일한 Integration 섹션 줄을 가집니다. 다음에서:
```
- superpowers:using-git-worktrees - REQUIRED: Set up isolated workspace before starting
```
다음으로 변경:
```
- superpowers:using-git-worktrees - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

**그 외 모든 항목 변경 없음:** 디스패치/리뷰 루프, 프롬프트 템플릿, 모델 선택, 상태 처리, red flags.

### 4. `codex-tools.md` — 환경 감지 문서 추가 (~15줄)

끝부분에 두 개의 새로운 섹션 추가:

**환경 감지 (Environment Detection):**

```markdown
## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.
```

**Codex App Finishing:**

```markdown
## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

## 변경되지 않는 사항

- `implementer-prompt.md`, `spec-reviewer-prompt.md`, `code-quality-reviewer-prompt.md` — 서브에이전트 프롬프트 건드리지 않음
- `executing-plans/SKILL.md` — 1줄의 Integration 설명만 변경됨 (`subagent-driven-development`와 동일); 모든 런타임 동작은 변경 없음
- `dispatching-parallel-agents/SKILL.md` — 워크트리나 finishing 작업 없음
- `.codex/INSTALL.md` — 설치 프로세스 변경 없음
- 4개 옵션의 finishing 메뉴 — Claude Code 및 Codex CLI를 위해 정확히 보존됨
- 전체 워크트리 생성 흐름 — 비워크트리 환경을 위해 정확히 보존됨
- 서브에이전트 디스패치/리뷰/반복 루프 — 변경 없음 (파일시스템 공유 확인됨)

## 범위 요약

| File | Change |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12줄 (Step 0) |
| `skills/finishing-a-development-branch/SKILL.md` | +20줄 (Step 1.5 + 정리 가드) |
| `skills/subagent-driven-development/SKILL.md` | 1줄 편집 |
| `skills/executing-plans/SKILL.md` | 1줄 편집 |
| `skills/using-superpowers/references/codex-tools.md` | +15줄 |

5개 파일에 걸쳐 약 50줄 추가/변경됨. 신규 파일 0개. 파괴적 변경 0개.

## 향후 고려 사항

세 번째 스킬이 동일한 감지 패턴을 필요로 하는 경우, 이를 공유 `references/environment-detection.md` 파일로 추출합니다 (접근 방식 B). 지금은 불필요 — 2개 스킬만 사용함.

## 테스트 계획

### 자동화 테스트 (구현 후 Claude Code에서 실행)

1. 일반 저장소 감지 — IN_LINKED_WORKTREE=false 어설션
2. 연결된 워크트리 감지 — 테스트 워크트리 `git worktree add`, IN_LINKED_WORKTREE=true 어설션
3. Detached HEAD 감지 — `git checkout --detach`, ON_DETACHED_HEAD=true 어설션
4. Finishing 스킬 핸드오프 출력 — 제한된 환경에서 핸드오프 메시지(4개 옵션 메뉴 아님) 검증
5. **Step 5 정리 가드** — 연결된 워크트리 생성 (`git worktree add /tmp/test-cleanup -b test-cleanup`), 내부로 `cd`, Step 5 정리 감지 (`GIT_DIR` 대 `GIT_COMMON`) 실행, `git worktree remove`를 호출하지 **않음을** 어설션. 그 후 메인 저장소로 `cd`하여 돌아와 동일 감지 실행, `git worktree remove`를 호출할 것임을 어설션. 완료 후 테스트 워크트리 정리.

### 수동 Codex App 테스트 (5개 테스트)

1. Worktree 스레드에서의 감지 (workspace-write) — GIT_DIR != GIT_COMMON, 비어 있는 브랜치 검증
2. Worktree 스레드에서의 감지 (Full access) — 동일한 감지, 다른 샌드박스 동작
3. Finishing 스킬 핸드오프 포맷 — 에이전트가 4개 옵션 메뉴가 아닌 핸드오프 페이로드를 출력하는지 검증
4. 전체 라이프사이클 — 감지 → 커밋 → finishing 감지 → 올바른 동작 → 정리
5. **Local 스레드에서의 샌드박스 폴백** — Codex App **Local 스레드** (workspace-write 샌드박스) 시작. 프롬프트: "Use the superpowers skill `using-git-worktrees` to set up an isolated workspace for implementing a small change." 사전 검사: `git checkout -b test-sandbox-check`가 `Operation not permitted`로 실패해야 함. 예상 동작: 스킬이 `GIT_DIR == GIT_COMMON`(일반 저장소)을 감지하고, `git worktree add -b`를 시도하다가, Seatbelt 거부에 걸려, Step 0 "이미 작업 공간에 있음" 동작으로 폴백함 — 설정을 실행하고, 베이스라인 테스트를 진행하며, 현재 디렉토리에서 준비됨을 보고함. 통과 기준: 에이전트가 모호한 에러 메시지 없이 우아하게 복구됨. 실패 기준: 에이전트가 날것의 Seatbelt 에러를 출력하거나, 재시도하거나, 혼란스러운 출력과 함께 포기함.

### 회귀 테스트

- 기존 Claude Code 스킬 트리거 테스트가 여전히 통과함
- 기존 subagent-driven-development 통합 테스트가 여전히 통과함
- 일반 Claude Code 세션: 전체 워크트리 생성 + 4개 옵션 finishing이 여전히 동작함
