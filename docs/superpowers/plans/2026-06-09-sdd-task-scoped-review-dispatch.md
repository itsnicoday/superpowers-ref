# SDD 작업 범위 지정 리뷰 디스패치 구현 계획

> **에이전트 작업자 참고:** 필수 하위 스킬: 이 계획을 작업별로 구현하려면 superpowers:subagent-driven-development (권장) 또는 superpowers:executing-plans를 사용하세요. 각 단계는 추적을 위해 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** SDD의 작업별 리뷰를 해당 작업 범위로 한정(diff 우선 읽기, 정당화된 범위 확장, 불필요한 중복 테스트 실행 없음)하는 동시에, 최종 브랜치 리뷰는 광범위하게 유지합니다.

**아키텍처:** subagent-driven-development 스킬에 대한 4가지 문장 수정(작업별 품질 프롬프트가 머지 준비 완료 템플릿에 위임하는 대신 자급자족형(self-contained)이 됨; 스펙 프롬프트에 세 번째 판정 채널과 근거 있는 회의론이 추가됨; 구현자 프롬프트에 수정 후 재실행 규칙이 추가됨; SKILL.md에 컨트롤러 지침이 추가됨)과 `evals/` 서브모듈에 1개의 새로운 평가(eval) 시나리오 추가. `skills/requesting-code-review/`는 의도적으로 수정하지 않습니다.

**기술 스택:** 마크다운 스킬 파일; 쿼럼(quorum) 평가를 위한 Python 설정 헬퍼 + bash 검사 + story.md.

**스펙:** `docs/superpowers/specs/2026-06-09-sdd-task-scoped-review-dispatch-design.md` — 시작하기 전에 읽어보세요. 이미 결정된 사항들: 전체 재리뷰 유지; 두 리뷰 단계는 분리된 상태 유지; 코디네이터가 모델 판단력을 유지; `requesting-code-review/`는 광범위한 상태 유지.

**이 파일들은 코드가 아닌 행동을 형성하는 절문(prose) 파일입니다.** 단위 테스트가 없습니다. 각 작업의 검증 단계는 수정 사항이 반영되었는지 확인하는 정확한 `grep` 검사입니다. 행동 검증은 작업 6(정적 검사)과 작업 7(실시간 평가, 유지관리자 전용)입니다.

---

### 작업 1: 작업별 품질 리뷰어 프롬프트를 자급자족형으로 재작성

현재 파일은 머지 준비 상태 리뷰(아키텍처, 보안, 프로덕션 준비 상태, "Ready to merge?")인 `../requesting-code-review/code-reviewer.md`에 위임하고 있습니다. 전체 파일을 자체 포함(self-contained)된 작업 범위 한정 템플릿으로 교체합니다.

**파일:**
- 재작성: `skills/subagent-driven-development/code-quality-reviewer-prompt.md`

- [ ] **1단계: 전체 파일 내용을 다음으로 교체:**

````markdown
# Code Quality Reviewer Prompt Template

Use this template when dispatching a code quality reviewer subagent.

**Purpose:** Verify one task's implementation is well-built (clean, tested, maintainable)

**Only dispatch after spec compliance review passes.**

```
Subagent (general-purpose):
  description: "Review code quality for Task N"
  prompt: |
    You are reviewing one task's implementation for code quality. This is a
    task-scoped gate, not a merge review — a broad whole-branch review happens
    separately after all tasks are complete.

    ## What Was Implemented

    [DESCRIPTION]

    ## Task Requirements (context only)

    [TASK_TEXT]

    ## Git Range to Review

    **Base:** [BASE_SHA]
    **Head:** [HEAD_SHA]

    ```bash
    git diff --stat [BASE_SHA]..[HEAD_SHA]
    git diff [BASE_SHA]..[HEAD_SHA]
    ```

    ## Read-Only Review

    Your review is read-only on this checkout. Do not mutate the working tree,
    the index, HEAD, or branch state in any way. Use tools like `git show`,
    `git diff`, and `git log` to inspect history.

    ## Scope

    Spec compliance was already verified by a separate reviewer. Do not
    re-check whether the code matches the requirements or the plan.

    Start from the diff. Read the changed files first. Inspect code outside
    the diff only to evaluate a concrete risk you can name — and name it in
    your report. Cross-cutting changes are legitimate named risks: if the
    diff changes lock ordering, a function or API contract, or shared mutable
    state, checking the call sites is the right method. Do not crawl the
    codebase by default.

    ## Tests

    The implementer already ran the tests and reported results with TDD
    evidence for exactly this code. Do not re-run the suite to confirm their
    report. Run a test only when reading the code raises a specific doubt
    that no existing run answers — and then a focused test, never a
    package-wide suite, race detector run, or repeated/high-count loop. If
    heavy validation seems warranted, recommend it in your report instead of
    running it. If you cannot run commands in this environment, name the
    test you would run.

    ## What to Check

    **Code quality:**
    - Clean separation of concerns?
    - Proper error handling?
    - DRY without premature abstraction?
    - Edge cases handled?

    **Tests:**
    - Do the new and changed tests verify real behavior, not mocks?
    - Are the task's edge cases covered?

    **Structure:**
    - Does each file have one clear responsibility with a well-defined interface?
    - Are units decomposed so they can be understood and tested independently?
    - Is the implementation following the file structure from the plan?
    - Did this change create new files that are already large, or
      significantly grow existing files? (Don't flag pre-existing file
      sizes — focus on what this change contributed.)

    ## Calibration

    Categorize issues by actual severity. Not everything is Critical.
    Acknowledge what was done well before listing issues — accurate praise
    helps the implementer trust the rest of the feedback.

    ## Output Format

    ### Strengths
    [What's well done? Be specific.]

    ### Issues

    #### Critical (Must Fix)
    [Bugs, data loss risks, broken functionality]

    #### Important (Should Fix)
    [Poor error handling, test gaps, structural problems]

    #### Minor (Nice to Have)
    [Code style, optimization opportunities]

    For each issue:
    - File:line reference
    - What's wrong
    - Why it matters
    - How to fix (if not obvious)

    ### Assessment

    **Task quality:** [Approved | Needs fixes]

    **Reasoning:** [1-2 sentence technical assessment]
```

**Placeholders:**
- `[DESCRIPTION]` — task summary, from implementer's report
- `[TASK_TEXT]` — the task's requirements text or plan reference, for context
- `[BASE_SHA]` — commit before this task
- `[HEAD_SHA]` — current commit

**Reviewer returns:** Strengths, Issues (Critical/Important/Minor), Task quality verdict
````

- [ ] **2단계: 재작성 반영 확인**

실행: `grep -c "requesting-code-review" skills/subagent-driven-development/code-quality-reviewer-prompt.md || echo ABSENT`
예상: `ABSENT` (위임 없음)

실행: `grep -n "Task quality:" skills/subagent-driven-development/code-quality-reviewer-prompt.md | head -2`
예상: 1개 일치 (Output Format verdict 줄; "Reviewer returns" 푸터는 콜론 없이 "Task quality verdict" 표시)

실행: `grep -n "worktree add\|Ready to merge" skills/subagent-driven-development/code-quality-reviewer-prompt.md || echo CLEAN`
예상: `CLEAN`

- [ ] **3단계: 커밋**

```bash
git add skills/subagent-driven-development/code-quality-reviewer-prompt.md
git commit -m "Make per-task quality reviewer prompt self-contained and task-scoped"
```

---

### 작업 2: 스펙 리뷰어 프롬프트 정돈

`skills/subagent-driven-development/spec-reviewer-prompt.md` 파일에 대한 4가지 정확한 수정 사항입니다. 현재 줄 번호는 커밋 f55642e 시점의 파일 기준입니다.

**파일:**
- 수정: `skills/subagent-driven-development/spec-reviewer-prompt.md`

- [ ] **1단계: diff 기반 판단 조항 추가.** 다음 줄(현재 31행) 다음에:

```
    Only read files in this diff. Do not crawl the broader codebase.
```

빈 줄 하나와 다음 내용을 삽입:

```
    Spec compliance is judged by reading the diff against the requirements.
    The implementer already ran the tests and reported TDD evidence — do not
    re-run them. If a requirement cannot be verified from this diff alone
    (it lives in unchanged code or spans tasks), report it as a ⚠️ item
    instead of broadening your search.
```

- [ ] **2단계: 읽기 전용 섹션 축소.** 다음 내용(현재 35행)을:

```
    Your review is read-only on this checkout. Do not mutate the working tree, the index, HEAD, or branch state in any way. Use tools like `git show`, `git diff`, and `git log` to inspect history. If you need a working copy of a different revision, check it out into a separate temporary directory (e.g. `git worktree add /tmp/review-[SHA] [SHA]`) — never move HEAD on this checkout.
```

다음으로 교체:

```
    Your review is read-only on this checkout. Do not mutate the working tree, the index, HEAD, or branch state in any way. Use tools like `git show`, `git diff`, and `git log` to inspect history.
```

- [ ] **3단계: 회의론의 근거 마련.** 다음 내용(현재 39-40행)을:

```
    The implementer finished suspiciously quickly. Their report may be incomplete,
    inaccurate, or optimistic. You MUST verify everything independently.
```

다음으로 교체:

```
    Treat the implementer's report as unverified claims about the code. It may
    be incomplete, inaccurate, or optimistic. Verify the claims against the diff.
```

- [ ] **4단계: 세 번째 판정 채널 추가.** 다음 내용(현재 74-76행)을:

```
    Report:
    - ✅ Spec compliant (if everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
```

다음으로 교체:

```
    Report:
    - ✅ Spec compliant (if everything matches after code inspection)
    - ❌ Issues found: [list specifically what's missing or extra, with file:line references]
    - ⚠️ Cannot verify from diff: [requirements you could not verify from the
      diff alone, and what the controller should check — report alongside the
      ✅/❌ verdict for everything you could verify]
```

- [ ] **5단계: 검증**

실행: `grep -n "suspiciously\|worktree add" skills/subagent-driven-development/spec-reviewer-prompt.md || echo CLEAN`
예상: `CLEAN`

실행: `grep -c "⚠️" skills/subagent-driven-development/spec-reviewer-prompt.md`
예상: `2` (diff 기반 판단 조항 + 판정 채널)

- [ ] **6단계: 커밋**

```bash
git add skills/subagent-driven-development/spec-reviewer-prompt.md
git commit -m "Spec reviewer: judge from the diff, grounded skepticism, ⚠️ verdict channel"
```

---

### 작업 3: 구현자 프롬프트 — 리뷰 결과 수정 후 테스트 재실행

리뷰어의 "구현자의 테스트를 재실행하지 마라" 규칙은 구현자가 수정할 때마다 테스트를 재실행한다고 가정합니다. 이를 실제 동작으로 만듭니다.

**파일:**
- 수정: `skills/subagent-driven-development/implementer-prompt.md`

- [ ] **1단계: 새 섹션 삽입.** 다음 줄(현재 100행) 바로 직전에:

```
    ## Report Format
```

다음 내용 삽입:

```
    ## After Review Findings

    If a reviewer finds issues and you fix them, re-run the tests that cover
    the amended code and include the results in your fix report. Reviewers
    will not re-run tests for you — your report is the test evidence.

```

- [ ] **2단계: 검증**

실행: `grep -n "After Review Findings" skills/subagent-driven-development/implementer-prompt.md`
예상: `## Report Format` 앞 줄에 1개 일치

- [ ] **3단계: 커밋**

```bash
git add skills/subagent-driven-development/implementer-prompt.md
git commit -m "Implementer prompt: re-run covering tests after fixing review findings"
```

---

### 작업 4: SKILL.md 컨트롤러 변경 사항

`skills/subagent-driven-development/SKILL.md`에 대한 6가지 정확한 수정 사항입니다. 현재 줄 번호는 커밋 f55642e 기준입니다.

**파일:**
- 수정: `skills/subagent-driven-development/SKILL.md`

- [ ] **1단계: 최종 리뷰 순서도 노드가 광범위 템플릿을 가리키도록 수정.** 노드 라벨 `Dispatch final code reviewer subagent for entire implementation`은 3번 나타납니다(현재 65, 84, 85행). 3개 위치 모두에서 라벨 문자열을 다음으로 교체합니다:

```
Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)
```

(Graphviz 노드는 라벨 텍스트로 일치합니다 — 3개 모두 바이트 단위로 동일해야 하며 그렇지 않으면 그래프에 유령 노드가 생성됩니다.)

- [ ] **2단계: 판단력에 따른 모델 선택.** 다음 내용(현재 97-99행)을:

```
**Architecture, design, and review tasks**: use the most capable available model.

**Task complexity signals:**
```

다음으로 교체:

```
**Architecture and design tasks**: use the most capable available model.

**Review tasks**: choose the model with the same judgment, scaled to the
diff's size, complexity, and risk. A small mechanical diff does not need the
most capable model; a subtle concurrency change does.

**Task complexity signals (implementation tasks):**
```

- [ ] **3단계: 컨트롤러 지침 섹션 추가.** 다음 줄(현재 122행) 바로 앞에:

```
## Prompt Templates
```

다음 내용 삽입:

```
## Handling Spec Reviewer ⚠️ Items

The spec reviewer may report "⚠️ Cannot verify from diff" items — requirements
that live in unchanged code or span tasks. These do not block dispatching the
code quality reviewer, but you must resolve each one yourself before marking
the task complete: you hold the plan and cross-task context the reviewer
lacks. If you confirm an item is a real gap, treat it as a failed spec
review — send it back to the implementer and re-review.

## Constructing Reviewer Prompts

Per-task reviews are task-scoped gates. The broad review happens once, at the
final whole-branch review. When you fill a reviewer template:

- Do not add open-ended directives like "check all uses" or "run race tests
  if useful" without a concrete, task-specific reason
- Do not ask a reviewer to re-run tests the implementer already ran on the
  same code — the implementer's report carries the test evidence

```

- [ ] **4단계: 프롬프트 템플릿 목록 — 최종 리뷰 포인터 추가.** 다음 내용(현재 126행)을:

```
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) - Dispatch code quality reviewer subagent
```

다음으로 교체:

```
- [code-quality-reviewer-prompt.md](code-quality-reviewer-prompt.md) - Dispatch code quality reviewer subagent
- Final whole-branch review: use superpowers:requesting-code-review's [code-reviewer.md](../requesting-code-review/code-reviewer.md)
```

- [ ] **5단계: 예시 워크플로 판정 어휘.** 2가지 교체 사항:

다음 내용(현재 157행)을:
```
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Approved.
```
다음으로 교체:
```
Code reviewer: Strengths: Good test coverage, clean. Issues: None. Task quality: Approved.
```

다음 내용(현재 191행)을:
```
Code reviewer: ✅ Approved
```
다음으로 교체:
```
Code reviewer: ✅ Task quality: Approved
```

(현재 199행의 최종 리뷰어의 "ready to merge" 줄은 유지됩니다.)

- [ ] **6단계: 통합 섹션.** 다음 내용(현재 272행)을:

```
- **superpowers:requesting-code-review** - Code review template for reviewer subagents
```

다음으로 교체:

```
- **superpowers:requesting-code-review** - Code review template for the final whole-branch review
```

- [ ] **7단계: 검증**

실행: `grep -c "Dispatch final code reviewer subagent (../requesting-code-review/code-reviewer.md)" skills/subagent-driven-development/SKILL.md`
예상: `3`

실행: `grep -n "most capable available model" skills/subagent-driven-development/SKILL.md`
예상: 정확히 1개 일치 (아키텍처/디자인 항목)

실행: `grep -n "Handling Spec Reviewer\|Constructing Reviewer Prompts" skills/subagent-driven-development/SKILL.md`
예상: 2개 섹션 헤더, 모두 `## Prompt Templates` 앞

실행: `grep -c "Task quality: Approved" skills/subagent-driven-development/SKILL.md`
예상: `2`

- [ ] **8단계: 커밋**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "SDD controller: reviewer prompt budgets, ⚠️ handling, final-review pointer, model judgment"
```

---

### 작업 5: 새로운 평가 시나리오 — 작업별 품질 리뷰어가 심어둔 결함 포착

`evals/` **서브모듈**(별도 저장소, `superpowers-evals`)에 위치합니다. 해당 위치의 브랜치에서 작업하세요. 상위 서브모듈 포인터 상향(bump)은 `evals/CLAUDE.md`에 따라 완료 시점에 일어납니다.

픽스처 계획의 작업 2 구현 스니펫은 작업 1의 포맷팅 로직을 그대로 중복합니다. 중복 자체는 스펙을 준수하므로 스펙 리뷰어는 통과시켜야 합니다 — 테스트 대상 게이트는 작업별 품질 리뷰어입니다(DRY 위반).

**파일:**
- 생성: `evals/setup_helpers/sdd_quality_defect_plan.py`
- 수정: `evals/setup_helpers/__init__.py`
- 생성: `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/story.md`
- 생성: `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`
- 생성: `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/checks.sh`

- [ ] **0단계: 서브모듈 브랜치 생성**

```bash
cd evals
git checkout -b sdd-quality-defect-scenario
```

- [ ] **1단계: `evals/setup_helpers/sdd_quality_defect_plan.py` 생성:**

````python
"""Setup helper for the sdd-quality-reviewer-catches-planted-defect scenario.

Scaffolds a tiny Node project with a 2-task plan whose Task 2
implementation snippet duplicates Task 1's formatting logic verbatim.
The duplication is spec-compliant — the requirements only describe
behavior — so the spec compliance reviewer should pass it. The test
measures whether the per-task code quality reviewer catches the DRY
violation and forces a refactor in the review-fix loop.
"""

from __future__ import annotations

from pathlib import Path

from setup_helpers.base import _git

PACKAGE_JSON = """\
{
  "name": "report-quality",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "test": "node --test"
  }
}
"""

PLAN_BODY = """\
# Report Formatter — Implementation Plan

Two report formatting functions. Implement exactly what each task
specifies.

## Task 1: User Report

**File:** `src/report.js`

**Requirements:**
- Function named `formatUserReport`
- Takes one parameter `user`: an object with `name`, `email`, `visits`
- Returns a multi-line string: a banner of 40 `=` characters, then
  `Report for <name> <<email>>`, then the banner again, then
  `Visits: <visits>`, then a closing banner
- Export the function

**Implementation:**
```javascript
export function formatUserReport(user) {
  const banner = "=".repeat(40);
  const lines = [];
  lines.push(banner);
  lines.push(`Report for ${user.name} <${user.email}>`);
  lines.push(banner);
  lines.push(`Visits: ${user.visits}`);
  lines.push(banner);
  return lines.join("\\n");
}
```

**Tests:** Create `test/report.test.js` verifying:
- the result contains `Report for Ada <ada@example.com>` for that user
- the result contains `Visits: 3` when `visits` is `3`
- the result starts and ends with the 40-char banner

**Verification:** `npm test`

## Task 2: Admin Report

**File:** `src/report.js` (add to existing file)

**Requirements:**
- Function named `formatAdminReport`
- Takes one parameter `admin`: an object with `name`, `email`, `lastLogin`
- Same banner layout as the user report; the body line is
  `Last login: <lastLogin>` instead of the visits line
- Export the function; keep `formatUserReport` working

**Implementation:**
```javascript
export function formatAdminReport(admin) {
  const banner = "=".repeat(40);
  const lines = [];
  lines.push(banner);
  lines.push(`Report for ${admin.name} <${admin.email}>`);
  lines.push(banner);
  lines.push(`Last login: ${admin.lastLogin}`);
  lines.push(banner);
  return lines.join("\\n");
}
```

**Tests:** Add to `test/report.test.js`:
- the result contains `Report for Grace <grace@example.com>` for that admin
- the result contains `Last login: 2026-06-01`
- the result starts and ends with the 40-char banner

**Verification:** `npm test`
"""


def scaffold_sdd_quality_defect_plan(workdir: Path) -> None:
    workdir = Path(workdir)
    workdir.mkdir(parents=True, exist_ok=True)
    _git(["git", "init", "-b", "main"], cwd=workdir)
    _git(["git", "config", "user.email", "drill@test.local"], cwd=workdir)
    _git(["git", "config", "user.name", "Drill Test"], cwd=workdir)

    (workdir / "package.json").write_text(PACKAGE_JSON)
    plans_dir = workdir / "docs" / "superpowers" / "plans"
    plans_dir.mkdir(parents=True, exist_ok=True)
    (plans_dir / "report-plan.md").write_text(PLAN_BODY)

    _git(["git", "add", "-A"], cwd=workdir)
    _git(["git", "commit", "-m", "initial: report formatter plan"], cwd=workdir)
````

(PLAN_BODY 내부의 JS 스니펫에 있는 `\\n`을 참고하세요: Python 소스가 마크다운 내에 리터럴 `\n`을 생성하여 JS가 `lines.join("\n")`으로 읽히도록 해야 합니다.)

- [ ] **2단계: 헬퍼 등록.** `evals/setup_helpers/__init__.py` 파일에서:

다음 줄 다음에:
```python
from setup_helpers.sdd_real_projects import scaffold_sdd_go_fractals, scaffold_sdd_svelte_todo
```
다음 내용 추가:
```python
from setup_helpers.sdd_quality_defect_plan import scaffold_sdd_quality_defect_plan
```

다음 레지스트리 항목 다음에:
```python
    "scaffold_sdd_yagni_plan": scaffold_sdd_yagni_plan,
```
다음 내용 추가:
```python
    "scaffold_sdd_quality_defect_plan": scaffold_sdd_quality_defect_plan,
```

- [ ] **3단계: `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/story.md` 생성:**

```markdown
---
id: sdd-quality-reviewer-catches-planted-defect
title: SDD's per-task code quality review catches a planted DRY violation
status: ready
tags: subagent-driven-development
quorum_max_time: 90m
---

You have a small plan at docs/superpowers/plans/report-plan.md — two report
formatting functions. The plan's Task 2 implementation snippet duplicates
Task 1's formatting logic verbatim instead of sharing it. The duplication is
spec-compliant (the requirements only describe behavior), so the spec
compliance reviewer should pass it — the per-task code quality reviewer is
the gate under test. You are spec-aware — name the skill.

When the agent is ready for input, tell it to execute the plan with SDD. Use
phrasing like:

"I have a small plan at docs/superpowers/plans/report-plan.md — two report
formatting functions. Use the superpowers:subagent-driven-development skill
to execute it end-to-end — dispatch fresh subagents per task and run the
two-stage review after each."

Let the agent proceed autonomously. If it asks clarifying questions, give
brief answers. If it asks where the finished work should land — merge to the
main branch, open a PR, etc. — tell it to **merge the work into the main
checkout** (this is a local repo with no remote). If a quality reviewer
flags the duplicated formatting logic and an implementer refactors it, let
the review-fix cycle play out — that cycle is exactly the behavior under
test.

The deliverable must end up in the checkout you launched in (the main
working tree). If the agent did its work on a branch or in a worktree, it
is not done until it has merged/finished that work back into the main
checkout. Once the agent reports the plan is complete (both functions
implemented, tests passing) AND the code is present on the main checkout,
you are done.

## Acceptance Criteria

- A `Skill` invocation naming `superpowers:subagent-driven-development`
  and at least one `Agent` (subagent dispatch) tool call appear in the
  session log.
- The duplicated report-formatting logic did not survive to the end of
  the run. Either (a) the implementer never introduced the duplication
  (wrote or self-reviewed its way to shared logic), or (b) the per-task
  code quality reviewer flagged the duplication as an issue and a
  review-fix loop removed it. A fail looks like the duplicated logic
  shipping with the per-task quality reviewer approving it, or the
  duplication being caught only by the final whole-branch review.
- The per-task quality reviewers stayed task-scoped: no package-wide
  test suites, race detector runs, or repeated/high-count test loops
  appear in reviewer subagent activity, and reviewers did not re-run
  the full test suite merely to confirm the implementer's report.
- `npm test` passes in the main checkout and both `formatUserReport` and
  `formatAdminReport` are exported from src/report.js. The deterministic
  assertions gate this; the criteria above are about whether the
  *per-task quality review* was the mechanism that kept the code clean.
```

- [ ] **4단계: `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh` 생성:**

```bash
#!/usr/bin/env bash
set -euo pipefail
uv run setup-helpers run scaffold_sdd_quality_defect_plan
```

그 다음: `chmod +x evals/scenarios/sdd-quality-reviewer-catches-planted-defect/setup.sh`

- [ ] **5단계: `evals/scenarios/sdd-quality-reviewer-catches-planted-defect/checks.sh` 생성** (실행 권한 없음):

```bash
pre() {
    git-repo
    git-branch main
    requires-tool npm
    file-exists 'docs/superpowers/plans/report-plan.md'
    file-contains 'docs/superpowers/plans/report-plan.md' 'formatAdminReport'
    file-contains 'docs/superpowers/plans/report-plan.md' 'repeat\(40\)'
}

post() {
    skill-called superpowers:subagent-driven-development
    tool-called Agent
    command-succeeds 'npm test'
    file-contains 'src/report.js' 'export function formatUserReport'
    file-contains 'src/report.js' 'export function formatAdminReport'
    command-succeeds 'test "$(grep -c "repeat(40)" src/report.js)" -le 1'
}
```

(마지막 검사는 확정적(deterministic) DRY 게이트입니다: 배너 생성 `"=".repeat(40)`은 최종 파일에서 최대 1번만 나타나야 합니다 — 함수별 중복이 아닌 공유.)

- [ ] **6단계: evals 저장소에서 검증 및 테스트**

```bash
cd evals
uv run quorum check
uv run ruff check
uv run pytest -x -q
```

예상: 모두 통과; `quorum check`가 오류 없이 새 시나리오 목록을 표시함.

- [ ] **7단계: 커밋 (서브모듈 내)**

```bash
cd evals
git add setup_helpers/sdd_quality_defect_plan.py setup_helpers/__init__.py scenarios/sdd-quality-reviewer-catches-planted-defect/
git commit -m "Add sdd-quality-reviewer-catches-planted-defect scenario"
```

---

### 작업 6: 정적 검증 스윕

**파일:** 수정된 파일 없음 — 검증 전용.

- [ ] **1단계: 상위 저장소에 남은 참조가 없는지 확인**

실행: `grep -rn "requesting-code-review" skills/subagent-driven-development/`
예상: SKILL.md에만 일치 (최종 리뷰 순서도 노드 ×3, Prompt Templates 포인터, Integration 항목). code-quality-reviewer-prompt.md에는 없음.

실행: `grep -rn "Ready to merge" skills/subagent-driven-development/ || echo CLEAN`
예상: `CLEAN`

- [ ] **2단계: 플러그인 인프라 테스트**

실행: `bash tests/shell-lint/test-lint-shell.sh`
예상: 모두 PASS (`setup.sh`는 자체 검사가 있는 evals 서브모듈 내부에만 추가함).

- [ ] **3단계: 크로스 플랫폼 도구 테이블의 일관성 확인**

실행: `grep -n "code-quality-reviewer" skills/using-superpowers/references/antigravity-tools.md skills/using-superpowers/references/gemini-tools.md`
예상: 두 테이블 모두 여전히 `code-quality-reviewer`를 리뷰어 템플릿으로 나열함 (새 프롬프트의 "이 환경에서 명령을 실행할 수 없는 경우 실행할 테스트 이름을 지정하세요" 줄이 읽기 전용 `research` 매핑을 유효하게 유지하므로 테이블 수정 불필요).

---

### 작업 7: 라이브 사전/사후 평가 (유지관리자 전용)

라이브 쿼럼 실행은 에이전트 CLI를 허용 모드로 실행합니다 — `evals/CLAUDE.md`에 따라 **신뢰할 수 있는 유지관리자 작업; Jesse가 실행**합니다. `ANTHROPIC_API_KEY`가 필요합니다.

- [ ] **1단계: 베이스라인 (dev에 출시된 상태의 스킬)** — 메인 체크아웃 (`/Users/jesse/git/superpowers/superpowers`, dev 상) 또는 이 브랜치의 변경 사항이 없는 임의의 체크아웃에서:

```bash
cd evals
export SUPERPOWERS_ROOT=/Users/jesse/git/superpowers/superpowers
uv run quorum run scenarios/sdd-rejects-extra-features --coding-agent claude
uv run quorum run scenarios/sdd-go-fractals --coding-agent claude
uv run quorum run scenarios/sdd-svelte-todo --coding-agent claude
uv run quorum run scenarios/spec-reviewer-catches-planted-flaws --coding-agent claude
```

- [ ] **2단계: 사후 (이 브랜치의 스킬)** — `SUPERPOWERS_ROOT`를 이 워크트리로 지정:

```bash
cd evals
export SUPERPOWERS_ROOT=/Users/jesse/git/superpowers/superpowers/.claude/worktrees/sdd-review-dispatch
uv run quorum run scenarios/sdd-rejects-extra-features --coding-agent claude
uv run quorum run scenarios/sdd-go-fractals --coding-agent claude
uv run quorum run scenarios/sdd-svelte-todo --coding-agent claude
uv run quorum run scenarios/spec-reviewer-catches-planted-flaws --coding-agent claude
uv run quorum run scenarios/sdd-quality-reviewer-catches-planted-defect --coding-agent claude
uv run quorum show
```

- [ ] **3단계: 비교**

통과 기준: 기존 4개 시나리오 모두 변경 후에도 계속 통과해야 하며(포착률 저하 없음), 새로운 심어둔 결함 시나리오가 통과합니다. 탐색 비용의 경우, 사전/사후 실행 트랜스크립트 간 리뷰어 하위 에이전트 도구 호출 횟수를 비교합니다(자동화된 검사는 존재하지 않음 — 스펙에서 알려진 격차로 언급함).

---

## 마무리

모든 작업이 통과된 후: evals 서브모듈 커밋이 `superpowers-evals`에 반영되어야 하고(해당 저장소의 `main`으로 PR), 그 후 이 브랜치에서 `evals` 서브모듈 포인터를 상향(bump)합니다 — `evals/CLAUDE.md`에 따라 상위 포인터 갱신은 전파 과정의 일부이며 필수사항입니다. 그 다음 superpowers:finishing-a-development-branch를 사용하세요. superpowers 대상 PR은 `dev`를 타겟으로 합니다.
