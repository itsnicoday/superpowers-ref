# 문서 리뷰 시스템 구현 계획

> **에이전트 작업자 참고:** 필수: 이 계획을 구현하려면 superpowers:subagent-driven-development (하위 에이전트를 사용할 수 있는 경우) 또는 superpowers:executing-plans를 사용하세요.

**목표:** 브레인스토밍(brainstorming) 및 계획 작성(writing-plans) 스킬에 스펙 및 계획 문서 리뷰 루프를 추가합니다.

**아키텍처:** 각 스킬 디렉터리에 리뷰어 프롬프트 템플릿을 생성합니다. 문서 생성 후 리뷰 루프를 추가하도록 스킬 파일을 수정합니다. 리뷰어 디스패치를 위해 범용 하위 에이전트와 함께 Task 도구를 사용합니다.

**기술 스택:** 마크다운 스킬 파일, Task 도구를 통한 하위 에이전트 디스패치

**스펙:** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## 청크 1: 스펙 문서 리뷰어

이 청크는 브레인스토밍 스킬에 스펙 문서 리뷰어를 추가합니다.

### 작업 1: 스펙 문서 리뷰어 프롬프트 템플릿 생성

**파일:**
- 생성: `skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **1단계:** 리뷰어 프롬프트 템플릿 파일 생성

````markdown
# Spec Document Reviewer Prompt Template

Use this template when dispatching a spec document reviewer subagent.

**Purpose:** Verify the spec is complete, consistent, and ready for implementation planning.

**Dispatch after:** Spec document is written to docs/superpowers/specs/

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
````

- [ ] **2단계:** 파일이 올바르게 생성되었는지 확인

실행: `cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
예상: 헤더 및 목적 섹션 표시

- [ ] **3단계:** 커밋

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### 작업 2: 브레인스토밍 스킬에 리뷰 루프 추가

**파일:**
- 수정: `skills/brainstorming/SKILL.md`

- [ ] **1단계:** 현재 브레인스토밍 스킬 읽기

실행: `cat skills/brainstorming/SKILL.md`

- [ ] **2단계:** "After the Design" 다음에 리뷰 루프 섹션 추가

"After the Design" 섹션을 찾고 문서화 다음, 구현 전에 새로운 "Spec Review Loop" 섹션을 추가합니다:

```markdown
**Spec Review Loop:**
After writing the spec document:
1. Dispatch spec-document-reviewer subagent (see spec-document-reviewer-prompt.md)
2. If ❌ Issues Found:
   - Fix the issues in the spec document
   - Re-dispatch reviewer
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to implementation setup

**Review loop guidance:**
- Same agent that wrote the spec fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect
```

- [ ] **3단계:** 변경 사항 검증

실행: `grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
예상: 새로운 리뷰 루프 섹션 표시

- [ ] **4단계:** 커밋

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## 청크 2: 계획 문서 리뷰어

이 청크는 계획 작성(writing-plans) 스킬에 계획 문서 리뷰어를 추가합니다.

### 작업 3: 계획 문서 리뷰어 프롬프트 템플릿 생성

**파일:**
- 생성: `skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **1단계:** 리뷰어 프롬프트 템플릿 파일 생성

````markdown
# Plan Document Reviewer Prompt Template

Use this template when dispatching a plan document reviewer subagent.

**Purpose:** Verify the plan chunk is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** Each plan chunk is written

```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
````

- [ ] **2단계:** 파일이 생성되었는지 확인

실행: `cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
예상: 헤더 및 목적 섹션 표시

- [ ] **3단계:** 커밋

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### 작업 4: Writing-Plans 스킬에 리뷰 루프 추가

**파일:**
- 수정: `skills/writing-plans/SKILL.md`

- [ ] **1단계:** 현재 스킬 파일 읽기

실행: `cat skills/writing-plans/SKILL.md`

- [ ] **2단계:** 청크별 리뷰 섹션 추가

"Execution Handoff" 섹션 앞에 추가:

```markdown
## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.
```

- [ ] **3단계:** 체크박스를 사용하도록 작업 구문 예시 업데이트

작업 구조(Task Structure) 섹션을 변경하여 체크박스 구문을 보여줍니다:

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **4단계:** 리뷰 루프 섹션이 추가되었는지 검증

실행: `grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
예상: 새로운 리뷰 루프 섹션 표시

- [ ] **5단계:** 작업 구문 예시가 업데이트되었는지 검증

실행: `grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
예상: 체크박스 구문 `### Task N:` 표시

- [ ] **6단계:** 커밋

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## 청크 3: 계획 문서 헤더 업데이트

이 청크는 새 체크박스 구문 요구사항을 참조하도록 계획 문서 헤더 템플릿을 업데이트합니다.

### 작업 5: Writing-Plans 스킬의 계획 헤더 템플릿 업데이트

**파일:**
- 수정: `skills/writing-plans/SKILL.md`

- [ ] **1단계:** 현재 계획 헤더 템플릿 읽기

실행: `grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **2단계:** 체크박스 구문을 참조하도록 헤더 템플릿 업데이트

계획 헤더에는 작업과 단계가 체크박스 구문을 사용한다는 지침이 포함되어야 합니다. 헤더 주석을 업데이트합니다:

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **3단계:** 변경 사항 검증

실행: `grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
예상: 체크박스 구문 언급이 포함된 업데이트된 헤더 표시

- [ ] **4단계:** 커밋

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```
