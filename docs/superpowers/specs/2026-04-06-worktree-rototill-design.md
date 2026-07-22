# Worktree Rototill: 감지 및 위임 (Detect-and-Defer)

**날짜:** 2026-04-06  
**상태:** 초안 (Draft)  
**티켓:** PRI-974  
**포함 항목:** PRI-823 (Codex App 호환성)

## 문제점

Superpowers는 워크트리 관리에 대해 주관적(opinionated)입니다 — 특정 경로(`.worktrees/<branch>`), 특정 명령어(`git worktree add`), 특정 정리 작업(`git worktree remove`). 반면 Claude Code, Codex App, Gemini CLI, Cursor는 모두 자체 경로, 라이프사이클 관리, 정리를 갖춘 네이티브 워크트리 지원을 제공합니다.

이로 인해 세 가지 오류 모드가 발생합니다:

1. **중복** — Claude Code에서 스킬이 `EnterWorktree`/`ExitWorktree`가 이미 수행하는 작업을 중복 실행함
2. **충돌** — Codex App에서 스킬이 이미 관리 중인 워크트리 내부에서 또 다른 워크트리를 생성하려 시도함
3. **유령 상태 (Phantom state)** — `.worktrees/`에 생성된 스킬 기반 워크트리는 하네스에서 보이지 않음; `.claude/worktrees/`에 생성된 하네스 기반 워크트리는 스킬에서 보이지 않음

네이티브 지원이 없는 하네스(Codex CLI, OpenCode, Copilot 단독 실행)의 경우, superpowers가 실질적인 공백을 채워줍니다. 스킬이 완전히 사라져서는 안 되며 — 네이티브 지원이 존재할 때 방해되지 않도록 물러나야(get out of the way) 합니다.

## 목표

1. 네이티브 하네스 워크트리 시스템이 존재하는 경우 이에 위임
2. 워크트리 지원이 부족한 하네스를 위한 계속적인 지원 제공
3. finishing-a-development-branch 스킬의 알려진 버그 3개 수정 (#940, #999, #238)
4. 워크트리 생성을 의무 사항이 아닌 선택 사항(opt-in)으로 변경 (#991)
5. 하드코딩된 `CLAUDE.md` 참조를 플랫폼 중립적 언어로 교체 (#1049)

## 비목표 (Non-Goals)

- 워크트리별 환경 컨벤션 (`.worktree-env.sh`, 포트 오프셋 지정) — Phase 4
- 경로 강제를 위한 PreToolUse 훅 — Phase 4
- 다중 저장소(Multi-repo) 워크트리 문서화 — Phase 4
- 워크트리를 위한 브레인스토밍 체크리스트 변경사항 — Phase 4
- `.superpowers-session.json` 메타데이터 추적 (PR #997의 흥미로운 아이디어이나 v1에서는 불필요)
- 워크트리로의 훅 심볼릭 링크 생성 (PR #965 아이디어, 별도 관련 사항)

## 설계 원칙

### 하네스가 아닌 상태 감지

하네스를 식별하기 위해 환경 변수를 탐지하는 대신 `GIT_DIR != GIT_COMMON`을 사용하여 "이미 워크트리 내부에 있는가?"를 판별합니다. 이는 신뢰할 수 있는 안정적인 git 프리미티브(git 2.5 이후, 2015년)이며, 모든 하네스에 걸쳐 보편적으로 작동하고 새로운 하네스가 나타나도 관리가 필요 없습니다.

### 선언적 의도, 지시적 폴백 (Declarative intent, prescriptive fallback)

스킬은 목표를 설명하고("격리된 작업 공간에서 작업이 진행되도록 보장") 가능할 때 네이티브 도구에 위임합니다. 네이티브 워크트리 지원이 없는 하네스에 대한 폴백으로서만 구체적인 git 명령어를 지시합니다. Step 1a가 먼저 나오며 네이티브 도구를 명시적으로 지칭합니다 (`EnterWorktree`, `WorktreeCreate`, `/worktree`, `--worktree`); Step 1b는 git 폴백과 함께 두 번째로 나옵니다. 원래 스펙은 Step 1a를 추상적으로 유지했으나("당신의 도구 세트를 알고 있을 것입니다"), TDD 검증 결과 Step 1a가 너무 모호할 경우 에이전트들이 Step 1b의 구체적인 명령어에 고정되는 현상이 입증되었습니다. 선호도를 신뢰할 수 있게 만들기 위해 명시적인 도구 지정과 동의-권한부여 가교(consent-authorization bridge)가 필요했습니다.

### 출처 기반 소유권 (Provenance-based ownership)

워크트리를 만든 주체가 정리 작업을 소유합니다. 하네스가 생성한 경우 superpowers는 이를 건드리지 않습니다. Superpowers가 (git 폴백을 통해) 생성한 경우 superpowers가 이를 정리합니다. 휴리스틱: 워크트리가 `.worktrees/` 또는 `worktrees/` 아래에 존재하면 superpowers가 소유합니다. 그 외의 모든 경우(`.claude/worktrees/`, `~/.codex/worktrees/`, `.gemini/worktrees/`, 또는 이전 사용자 글로벌 Superpowers 경로)는 하네스 또는 사용자의 소유이므로 건드리지 않고 남겨둡니다.

## 설계

### 1. `using-git-worktrees` SKILL.md 재작성

스킬은 생성 전에 세 개의 새로운 단계를 얻고 생성 흐름을 단순화합니다.

#### Step 0: 기존 격리 상태 감지

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

세 가지 결과:

| Condition | Meaning | Action |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 일반 저장소 체크아웃 | Step 0.5로 진행 |
| `GIT_DIR != GIT_COMMON`, 이름이 지정된 브랜치 | 이미 연결된 워크트리 내부 | Step 3(프로젝트 설정)으로 건너뜀. 보고: "Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`, detached HEAD | 외부에서 관리되는 워크트리 (예: Codex App 샌드박스) | Step 3으로 건너뜀. 보고: "Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

Step 0은 누가 워크트리를 만들었는지, 어떤 하네스가 실행 중인지 신경 쓰지 않습니다. 출처와 관계없이 워크트리는 워크트리입니다.

**서브모듈 가드:** `GIT_DIR != GIT_COMMON`은 git 서브모듈 내부에서도 참입니다. "이미 워크트리 내부"라고 결론 내리기 전에 서브모듈 내부가 아닌지 확인합니다:

```bash
# 이 명령어가 경로를 반환하면 워크트리가 아닌 서브모듈 내부임
git rev-parse --show-superproject-working-tree 2>/dev/null
```

서브모듈 내부인 경우 `GIT_DIR == GIT_COMMON`으로 취급합니다 (Step 0.5로 진행).

#### Step 0.5: 동의 구하기 (Consent)

Step 0에서 기존 격리 상태가 발견되지 않으면 (`GIT_DIR == GIT_COMMON`), 생성 전 확인합니다:

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

'예'인 경우 Step 1로 진행합니다. '아니오'인 경우 해당 위치에서 작업합니다 — 워크트리 없이 Step 3으로 건너뜀.

이 단계는 Step 0에서 기존 격리 상태가 감지된 경우 완전히 건너뜁니다 (이미 존재하는 것에 대해 물어볼 이유가 없음).

#### Step 1a: 네이티브 도구 (선호됨)

> 사용자가 격리된 작업 공간을 요청했습니다 (Step 0 동의). 이용 가능한 도구를 확인하십시오 — `EnterWorktree`, `WorktreeCreate`, `/worktree` 명령어, 또는 `--worktree` 플래그를 가지고 있습니까? 만약 그렇다면(YES): 사용자의 워크트리 생성 동의는 해당 도구를 사용할 수 있는 권한 부여입니다. 지금 해당 도구를 사용하고 Step 3으로 건너뛰십시오.

네이티브 도구를 사용한 후 Step 3(프로젝트 설정)으로 건너뜁니다.

**설계 참고사항 — TDD 수정:** 원래 스펙은 의도적으로 짧고 추상적인 Step 1a를 사용했습니다 ("당신은 당신의 도구 세트를 알고 있습니다 — 스킬이 특정 도구를 지칭할 필요가 없습니다"). TDD 검증 결과 이 방식은 실패했습니다: 에스컬레이션된 에이전트들이 Step 1b의 구체적인 git 명령어에 고정되어 추상적인 지침을 무시했습니다 (2/6 통과율). 세 가지 변경사항으로 이를 해결했습니다 (GREEN 및 PRESSURE 테스트 전반에 걸쳐 50/50 통과율):

1. **명시적 도구 지칭** — `EnterWorktree`, `WorktreeCreate`, `/worktree`, `--worktree`를 이름으로 나열함으로써 결정 방식을 해석("네이티브 도구가 있는가?")에서 사실 조회("내 도구 목록에 `EnterWorktree`가 있는가?")로 전환합니다. 해당 도구가 없는 플랫폼의 에이전트는 간단히 확인하고 아무것도 발견하지 못한 채 Step 1b로 넘어갑니다. 오탐은 관찰되지 않았습니다.
2. **동의 가교 (Consent bridge)** — "사용자의 워크트리 생성 동의는 해당 도구를 사용할 수 있는 권한 부여입니다"라는 문구는 `EnterWorktree`의 도구 수준 가드레일("사용자가 명시적으로 요청할 때만")을 직접적으로 다룹니다. 도구 설명이 스킬 지침보다 우선하므로 (Claude Code #29950), 스킬은 사용자의 동의를 도구가 요구하는 권한 부여로 프레임화해야 합니다.
3. **Red Flag 항목** — Red Flags 섹션에 구체적인 안티 패턴을 명시 ("네이티브 워크트리 도구가 있음에도 `git worktree add`를 사용하는 것 — 이것이 1위 실수입니다").

파일 분할(별도 스킬로 Step 1b 분리)도 테스트되었으나 불필요한 것으로 입증되었습니다. 고정 문제는 git 명령어의 물리적 분리가 아닌 Step 1a 텍스트의 품질을 통해 해결됩니다. 전체 240줄 스킬(모든 git 명령어가 표시됨)을 포함한 대조 테스트가 20/20 통과했습니다.

#### Step 1b: Git Worktree 폴백

네이티브 도구를 사용할 수 없을 때 수동으로 워크트리를 생성합니다.

**디렉토리 선택** (우선순위 순서):
1. 프로젝트의 에이전트 지시사항 파일 (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules 또는 이와 동등한 파일)에서 워크트리 디렉토리 선호사항을 확인합니다.
2. 기존 `.worktrees/` 또는 `worktrees/` 디렉토리가 있는지 확인합니다 — 발견되면 사용합니다. 둘 다 존재하면 `.worktrees/`가 우선합니다.
3. 기본값으로 `.worktrees/`를 사용합니다.

대화형 디렉토리 선택 프롬프트는 제공하지 않습니다. 이전 사용자 글로벌 Superpowers 워크트리 경로는 감지되거나 제안되지 않습니다; 새로운 수동 워크트리는 사용자가 다른 위치를 명시적으로 지정하지 않는 한 프로젝트 로컬에 위치합니다.

**안전성 검증** (프로젝트 로컬 디렉토리만 해당):

```bash
git check-ignore -q .worktrees 2>/dev/null
```

무시 설정이 되어 있지 않다면, 진행하기 전에 `.gitignore`에 추가하고 커밋합니다.

**생성:**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**훅 인식 (Hooks awareness):** Git 워크트리는 상위 저장소의 훅 디렉토리를 상속받지 않습니다. 1b를 통해 워크트리를 생성한 후, 메인 저장소에 훅 디렉토리가 존재하면 이를 심볼릭 링크로 연결합니다:

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

이를 통해 작업이 워크트리로 이동할 때 pre-commit 검사, 린터 및 기타 훅이 무음으로 중단되는 것을 방지합니다. (PR #965의 아이디어.)

**샌드박스 폴백:** `git worktree add`가 권한 오류로 실패하면 제한된 환경으로 취급합니다. 생성을 건너뛰고 현재 디렉토리에서 작업하며 Step 3으로 진행합니다.

**단계 번호 참고사항:** 현재 스킬은 Step 1-4를 평탄한 목록으로 가지고 있습니다. 이 재설계는 0, 0.5, 1a, 1b, 3, 4를 사용합니다. Step 2는 없습니다 — 이는 이제 1a/1b 구조로 분할된 이전의 단일체 "Create Isolated Workspace" 단계였습니다. 구현 시 명확하게 번호를 다시 매기거나(예: 0 → "Step 0: Detect", 0.5 → Step 0의 흐름 내, 1a/1b → "Step 1", 3 → "Step 2", 4 → "Step 3") 메모와 함께 현재 번호를 유지할 수 있습니다. 구현자의 선택 사항입니다.

#### Steps 3-4: 프로젝트 설정 및 베이스라인 테스트 (변경 없음)

어떤 경로로 작업 공간이 생성되었든 관계없이 (Step 0 기존 감지, Step 1a 네이티브 도구, Step 1b git 폴백, 또는 워크트리 없음), 실행은 하나로 수렴합니다:

- **Step 3:** 프로젝트 설정 자동 감지 및 실행 (`npm install`, `cargo build`, `pip install`, `go mod download` 등)
- **Step 4:** 테스트 수트 실행. 테스트가 실패하면 실패 사항을 보고하고 진행 여부를 확인.

### 2. `finishing-a-development-branch` SKILL.md 재작성

finishing 스킬은 환경 감지 기능을 얻고 3개의 버그를 수정합니다.

#### Step 1: 테스트 검증 (변경 없음)

프로젝트의 테스트 수트를 실행합니다. 테스트가 실패하면 중단합니다. 완료 옵션을 제공하지 마십시오.

#### Step 1.5: 환경 감지 (신규)

생성 시의 Step 0과 동일한 감지를 재실행합니다:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

세 가지 경로:

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (일반 저장소) | 표준 4개 옵션 | 정리할 워크트리 없음 |
| `GIT_DIR != GIT_COMMON`, 이름이 지정된 브랜치 | 표준 4개 옵션 | 출처 기반 정리 (Step 5 참조) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | 축소된 메뉴: 새 브랜치로 푸시 + PR, 있는 그대로 유지, 폐기 | 머지 옵션 없음 (detached HEAD에서는 머지 불가) |

#### Step 2: 베이스 브랜치 결정 (변경 없음)

#### Step 3: 옵션 제시

**일반 저장소 및 이름이 지정된 브랜치 워크트리:**

1. 로컬에서 `<base-branch>`로 머지
2. 푸시 후 Pull Request 생성
3. 브랜치를 있는 그대로 유지 (나중에 처리)
4. 작업 폐기

**Detached HEAD:**

1. 새 브랜치로 푸시 후 Pull Request 생성
2. 있는 그대로 유지 (나중에 처리)
3. 작업 폐기

#### Step 4: 선택 실행

**옵션 1 (로컬 머지):**

```bash
# CWD 안전성을 위해 메인 저장소 루트 가져오기 (Bug #238 수정)
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 먼저 머지하고 성공 여부를 검증한 후 항목 제거
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# 머지가 성공한 후에만: 워크트리 제거 후 브랜치 삭제 (Bug #999 수정)
git worktree remove "$WORKTREE_PATH"  # superpowers가 소유한 경우에만
git branch -d <feature-branch>
```

순서가 매우 중요합니다: 머지 → 검증 → 워크트리 제거 → 브랜치 삭제. 이전 스킬은 워크트리를 제거하기 전에 브랜치를 삭제했습니다 (워크트리가 여전히 브랜치를 참조하므로 실패함). 워크트리를 먼저 제거하는 단순 수정 방식도 잘못되었습니다 — 그 후 머지가 실패하면 작업 디렉토리가 사라지고 변경사항이 유실됩니다.

**옵션 2 (PR 생성):**

브랜치 푸시, PR 생성. 워크트리를 정리하지 마십시오 — 사용자는 PR 반복 작업을 위해 이것이 필요합니다. (Bug #940 수정: 모순되는 "Then: Cleanup worktree" 문구 제거.)

**옵션 3 (있는 그대로 유지):** 아무 작업도 하지 않음.

**옵션 4 (폐기):** 직접 입력하는 "discard" 확인이 필요함. 그 후 워크트리 제거(superpowers가 소유한 경우), 브랜치 강제 삭제.

#### Step 5: 정리 (업데이트됨)

```
if GIT_DIR == GIT_COMMON:
    # 일반 저장소, 정리할 워크트리 없음
    done

if worktree path is under .worktrees/ or worktrees/:
    # Superpowers가 생성함 — 당사가 정리를 소유함
    cd to main repo root       # Bug #238 수정
    git worktree remove <path>

else:
    # 하네스가 생성함 — 손대지 않음
    # 플랫폼이 workspace-exit 도구를 제공하면 사용
    # 그렇지 않다면 워크트리를 그대로 남겨둠
```

정리 작업은 옵션 1과 4에 대해서만 실행됩니다. 옵션 2와 3은 항상 워크트리를 보존합니다. (Bug #940 수정.)

**오래된 워크트리 정리 (Stale worktree pruning):** 모든 `git worktree remove` 실행 후 자가 치유 단계로서 `git worktree prune`을 실행합니다. 워크트리 디렉토리는 대역 외에서(예: 하네스 정리, 수동 `rm`, 또는 `.claude/` 정리로 인해) 삭제될 수 있어, 혼란스러운 오류를 유발하는 오래된 등록 정보를 남깁니다. 한 줄로 무음으로 진행되는 부식을 방지합니다. (PR #1072의 아이디어.)

### 3. 통합 업데이트

#### `subagent-driven-development` 및 `executing-plans`

둘 다 현재 통합 섹션에서 `using-git-worktrees`를 필수(REQUIRED) 항목으로 나열하고 있습니다. 다음과 같이 변경합니다:

> `using-git-worktrees` — 격리된 작업 공간 보장 (생성하거나 기존 상태 검증)

스킬 자체가 이제 동의(Step 0.5) 및 감지(Step 0)를 처리하므로, 호출하는 스킬들이 별도로 게이팅하거나 프롬프트를 띄울 필요가 없습니다.

#### `writing-plans`

"전용 워크트리(브레인스토밍 스킬에 의해 생성됨)에서 실행되어야 함"이라는 오래된 주장을 제거합니다. 브레인스토밍은 설계 스킬이며 워크트리를 생성하지 않습니다. 워크트리 확인은 `using-git-worktrees`를 통해 실행 시점에 일어납니다.

### 4. 플랫폼 중립적 지시사항 파일 참조

워크트리 관련 스킬에서 하드코딩된 `CLAUDE.md` 지칭을 모두 다음으로 교체합니다:

> "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"

이는 Step 1b의 디렉토리 선호사항 확인에 적용됩니다.

## 번들로 수정된 버그 (Bug Fixes)

| Bug | Problem | Fix | Location |
|-----|---------|-----|----------|
| #940 | 옵션 2 문구는 "Then: Cleanup worktree (Step 5)"라고 하지만 퀵 레퍼런스는 유지하라고 함. Step 5는 "For Options 1, 2, 4"라고 하지만 Common Mistakes는 "Options 1 and 4 only"라고 함. | 옵션 2에서 정리를 제거. Step 5는 옵션 1과 4에만 적용됨. | finishing SKILL.md |
| #999 | 옵션 1이 워크트리를 제거하기 전에 브랜치를 삭제함. 워크트리가 여전히 브랜치를 참조하므로 `git branch -d`가 실패할 수 있음. | 순서 변경: 머지 → 테스트 검증 → 워크트리 제거 → 브랜치 삭제. 무언가가 제거되기 전에 머지가 성공해야 함. | finishing SKILL.md |
| #238 | CWD가 제거되는 워크트리 내부에 있으면 `git worktree remove`가 무음으로 실패함. | CWD 가드 추가: `git worktree remove` 전 메인 저장소 루트로 `cd`. | finishing SKILL.md |

## 해결된 이슈 목록

| Issue | Resolution |
|-------|-----------|
| #940 | 직접 수정 (Bug #940) |
| #991 | Step 0.5에서 선택적 동의 |
| #918 | Step 0 감지 + Step 1.5 finishing 감지 |
| #1009 | Step 1a로 해결됨 — 에이전트가 네이티브 경로에 생성하는 네이티브 도구(예: `EnterWorktree`)를 사용함. Step 1a 작동에 의존함; 리스크 참조. |
| #999 | 직접 수정 (Bug #999) |
| #238 | 직접 수정 (Bug #238) |
| #1049 | 플랫폼 중립적 지시사항 파일 참조 |
| #279 | 감지 및 위임으로 해결됨 — 재정의하지 않으므로 네이티브 경로가 존중됨 |
| #574 | **연기됨.** 이 스펙의 어떤 내용도 버그가 존재하는 브레인스토밍 스킬을 건드리지 않음. 전체 수정(브레인스토밍 체크리스트에 워크트리 단계 추가)은 Phase 4임. |

## 리스크

### Step 1a는 비중 있는 하중을 받는 가설임 — 해결됨

Step 1a — 에이전트가 git 폴백보다 네이티브 워크트리 도구를 선호한다는 점 — 는 전체 설계가 입각한 기반입니다. 네이티브 지원이 있는 하네스에서 에이전트가 Step 1a를 무시하고 Step 1b로 빠지면 감지 및 위임은 완전히 실패합니다.

**상태:** 이 리스크는 구현 중에 현실화되었습니다. 원래의 추상적인 Step 1a("당신은 당사의 도구 세트를 알고 있습니다")는 Claude Code에서 2/6으로 실패했습니다. TDD 게이트는 설계대로 작동했습니다 — 스킬 파일이 수정되기 전에 실패를 포착하여 결함이 있는 릴리스를 방지했습니다. 세 번의 REFACTOR 반복을 통해 근본 원인을 식별하고 (구체적 명령어에 대한 에이전트 고정, 스킬 지침을 재정의하는 도구 설명 가드레일) GREEN 및 PRESSURE 테스트 전반에 걸쳐 50/50으로 검증된 수정사항을 도출했습니다. 자세한 내용은 위의 Step 1a 설계 참고사항을 참조하십시오.

**교차 플랫폼 검증:**

2026-04-06 기준, Claude Code는 에이전트가 세션 중간에 호출할 수 있는 워크트리 도구(`EnterWorktree`)를 가진 유일한 하네스입니다. 다른 모든 하네스는 에이전트가 시작되기 전에 워크트리를 생성하거나(Codex App, Gemini CLI, Cursor) 네이티브 워크트리 지원이 없습니다(Codex CLI, OpenCode). Step 1a는 미래 호환성을 갖춥니다: 다른 하네스가 에이전트 호출 가능한 워크트리 도구를 추가할 때, 에이전트는 나열된 예시 항목들과 이를 매칭하고 스킬 변경 없이 사용할 것입니다.

| Harness | Current worktree model | Skill mechanism | Tested |
|---------|----------------------|-----------------|--------|
| Claude Code | 에이전트 호출 가능 `EnterWorktree` | Step 1a | 50/50 (GREEN + PRESSURE) |
| Codex CLI | 네이티브 도구 없음 (쉘 전용) | Step 1b git 폴백 | 6/6 (`codex exec`) |
| Gemini CLI | 실행 시 `--worktree` 플래그, 에이전트 도구 없음 | 플래그 포함 실행 시 Step 0, 아니면 Step 1b | Step 0: 1/1, Step 1b: 1/1 (`gemini -p`) |
| Cursor Agent | 사용자 대상 `/worktree`, 에이전트 도구 없음 | 사용자가 활성화한 경우 Step 0, 아니면 Step 1b | Step 0: 1/1, Step 1b: 1/1 (`cursor-agent -p`) |
| Codex App | 플랫폼 관리, detached HEAD, 에이전트 도구 없음 | Step 0 기존 상태 감지 | 1/1 시뮬레이션됨 |
| OpenCode | 감지 전용 (`ctx.worktree`), 에이전트 도구 없음 | Step 1b git 폴백 | 미테스트 (CLI 접근 권한 없음) |

**잔여 리스크:**
1. Anthropic이 `EnterWorktree`의 도구 설명을 더 제한적으로 변경하면(예: "스킬 지침에 기반하여 사용하지 마라"), 동의 가교가 깨집니다. 도구 설명이 스킬 기반 호출을 수용하도록 요청하는 이슈를 제기할 필요가 있습니다.
2. 다른 하네스가 에이전트 호출 가능한 워크트리 도구를 추가할 때, Step 1a의 목록에 없는 이름을 사용할 수 있습니다. 새로운 도구가 등장함에 따라 목록을 업데이트해야 합니다. 일반적인 표현("a worktree or workspace-isolation tool")이 어느 정도 미래 커버리지를 제공합니다.

### 출처 휴리스틱

`.worktrees/` 또는 `worktrees/` = 당사 것, 그 외 = 손대지 않음 휴리스틱은 모든 현재 하네스에 대해 잘 작동합니다. 미래의 하네스가 해당 프로젝트 로컬 디렉토리 중 하나를 컨벤션으로 채택하면 오탐이 발생할 수 있습니다 (superpowers가 하네스 소유 워크트리를 정리하려고 시도함). 마찬가지로, 사용자가 superpowers 없이 수동으로 `git worktree add .worktrees/experiment`를 실행하면 당사가 소유권을 잘못 주장하게 됩니다. 둘 다 리스크가 낮습니다 — 모든 하네스는 브랜딩된 경로를 사용하며, 수동 `.worktrees/` 생성은 드뭅니다 — 그러나 유의할 가치가 있습니다.

### Detached HEAD 마무리

detached HEAD 워크트리에 대한 축소된 메뉴(머지 옵션 없음)는 Codex App의 샌드박스 모델에 적합합니다. 사용자가 다른 이유로 detached HEAD 상태에 있다면, 축소된 메뉴는 여전히 유효합니다 — 먼저 브랜치를 생성하지 않고는 detached HEAD에서 실제로 머지할 수 없기 때문입니다.

## 구현 참고사항

두 스킬 파일 모두 구현 중 업데이트가 필요한 핵심 단계 이외의 섹션을 포함하고 있습니다:

- **프론트매터** (`name`, `description`): 감지 및 위임 동작을 반영하도록 업데이트
- **Quick Reference 테이블**: 새로운 단계 구조 및 버그 수정 사항에 맞게 재작성
- **Common Mistakes 섹션**: 이전 동작을 참조하는 항목 업데이트 또는 제거 (예: "CLAUDE.md 검사 건너뛰기"는 이제 틀림)
- **Red Flags 섹션**: 새로운 우선순위를 반영하도록 업데이트 (예: "Step 0이 기존 격리를 감지했을 때 절대 워크트리를 생성하지 마라")
- **Integration 섹션**: 스킬 간 교차 참조 업데이트

스펙은 *무엇이 변경되는지*를 설명합니다; 구현 플랜은 이러한 보조 섹션에 대한 정확한 편집 내용을 지정할 것입니다.

## 향후 작업 (본 스펙에 포함되지 않음)

- **Phase 3 잔여:** `$TMPDIR` 디렉토리 옵션 (#666), 캐싱 및 환경 상속을 위한 설정 문서화 (#299)
- **Phase 4:** 경로 강제를 위한 PreToolUse 훅 (#1040), 워크트리별 환경 컨벤션 (#597), 브레인스토밍 체크리스트 워크트리 단계 (#574), 다중 저장소 문서화 (#710)
