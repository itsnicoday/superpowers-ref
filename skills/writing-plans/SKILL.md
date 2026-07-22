---
name: writing-plans
description: 코드를 건드리기 전에 여러 단계로 이루어진 태스크에 대한 명세서나 요구사항이 있을 때 사용합니다.
---

# 계획 작성하기 (Writing Plans)

## 개요 (Overview)

엔지니어가 우리 코드베이스에 대한 컨텍스트가 전혀 없고 취향이 의심스럽다고 가정하여 포괄적인 구현 계획을 작성하세요. 엔지니어가 알아야 할 모든 것(각 태스크에서 어떤 파일을 건드려야 하는지, 코드, 테스트, 확인해야 할 문서, 테스트 방법)을 문서화하세요. 전체 계획을 한입 크기의 태스크(bite-sized tasks) 형태로 제공하세요. DRY, YAGNI, TDD 및 빈번한 커밋을 적용하세요.

상대방이 숙련된 개발자이지만 우리의 툴셋이나 문제 도메인에 대해서는 거의 모른다고 가정하세요. 또한 좋은 테스트 디자인에 대해서도 잘 모른다고 가정하세요.

**시작 시 안내:** "구현 계획을 생성하기 위해 writing-plans 스킬을 사용합니다."

**컨텍스트:** 격리된 worktree에서 작업 중인 경우, 이는 실행 시점에 `superpowers:using-git-worktrees` 스킬을 통해 생성되었어야 합니다.

**계획 저장 위치:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (계획 위치에 대한 사용자 선호도가 있는 경우 이 기본값이 오버라이드됨)

## 범위 점검 (Scope Check)

명세서(spec)가 여러 독립적인 하위 시스템을 다루는 경우, 브레인스토밍 단계에서 하위 프로젝트 명세서로 나누어졌어야 합니다. 그렇지 않았다면 이를 별도의 계획(하위 시스템당 하나)으로 나눌 것을 제안하세요. 각 계획은 그 자체로 작동하고 테스트 가능한 소프트웨어를 만들어내야 합니다.

## 파일 구조 (File Structure)

태스크를 정의하기 전에 어떤 파일이 생성되거나 수정될지, 그리고 각 파일의 역할이 무엇인지 정리하세요. 이 단계에서 분해에 대한 결정이 확정됩니다.

- 명확한 경계와 잘 정의된 인터페이스를 가진 단위로 설계하세요. 각 파일은 하나의 명확한 책임을 가져야 합니다.
- 한 번에 컨텍스트에 담을 수 있는 코드에 대해 더 잘 추론할 수 있고, 파일이 집중되어 있을 때 수정 작업이 더 안정적입니다. 너무 많은 일을 하는 큰 파일보다 작고 집중된 파일을 선호하세요.
- 함께 변경되는 파일은 함께 배치되어야 합니다. 기술 레이어가 아닌 책임에 따라 분할하세요.
- 기존 코드베이스에서는 확립된 패턴을 따르세요. 코드베이스가 큰 파일을 사용하는 경우 일방적으로 재구조화하지 마세요. 다만 수정하려는 파일이 너무 다루기 힘들어졌다면 계획에 분할을 포함하는 것이 합리적입니다.

이 구조는 태스크 분해에 영향을 줍니다. 각 태스크는 독립적으로 의미가 통하는 자체 포함된 변경 사항을 생성해야 합니다.

## 적절한 태스크 크기 조정 (Task Right-Sizing)

태스크는 자체 테스트 사이클을 가지고 새로운 검토자의 게이트(gate)를 거칠 가치가 있는 가장 작은 단위입니다. 태스크 경계를 설정할 때: 설정, 구성, 스캐폴딩 및 문서화 단계는 산출물에 해당 단계가 필요한 태스크 내부로 접어 넣으세요; 검토자가 이웃 태스크는 승인하면서 해당 태스크는 의미 있게 거부할 수 있는 지점에서만 분할하세요. 각 태스크는 독립적으로 테스트 가능한 산출물로 끝납니다.

## 한입 크기 태스크의 세분성 (Bite-Sized Task Granularity)

**각 단계는 하나의 작업입니다 (2~5분 소요):**
- "실패하는 테스트 작성" - 단계
- "실패하는지 확인하기 위해 실행" - 단계
- "테스트를 통과하기 위한 최소한의 코드 구현" - 단계
- "테스트를 실행하여 통과하는지 확인" - 단계
- "커밋" - 단계

## 계획 문서 헤더 (Plan Document Header)

**모든 계획은 반드시 다음 헤더로 시작해야 합니다:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

---
```

## 태스크 구조 (Task Structure)

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter
  and return types. A task's implementer sees only their own task; this
  block is how they learn the names and types neighboring tasks use.]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 플레이스홀더 사용 금지 (No Placeholders)

모든 단계에는 엔지니어에게 필요한 실제 내용이 포함되어야 합니다. 다음과 같은 표현은 **계획의 결함**이므로 절대로 작성하지 마세요:
- "TBD", "TODO", "추후 구현", "세부 사항 채우기"
- "적절한 에러 처리 추가" / "검증 추가" / "엣지 케이스 처리"
- "위 내용에 대한 테스트 작성" (실제 테스트 코드 없음)
- "Task N과 유사함" (코드를 반복하세요 — 엔지니어가 순서 없이 태스크를 읽을 수 있습니다)
- 방법을 보여주지 않고 무엇을 해야 하는지만 설명하는 단계 (코드 단계에는 코드 블록 필수)
- 어떤 태스크에서도 정의되지 않은 타입, 함수 또는 메서드 참조

## 기억할 점 (Remember)
- 항상 정확한 파일 경로 사용
- 모든 단계에 완전한 코드 제공 — 단계에서 코드를 변경하는 경우 코드를 그대로 표시
- 예상 출력 결과가 포함된 정확한 명령어
- DRY, YAGNI, TDD, 빈번한 커밋

## 자체 검토 (Self-Review)

전체 계획을 작성한 후 새 시각으로 명세서를 보고 계획을 명세서와 비교 점검하세요. 이는 서브에이전트를 파견하지 않고 직접 수행하는 체크리스트입니다.

**1. 명세서 커버리지 (Spec coverage):** 명세서의 각 섹션/요구사항을 훑어보세요. 이를 구현하는 태스크를 지적할 수 있나요? 누락된 부분을 기록하세요.

**2. 플레이스홀더 스캔 (Placeholder scan):** 계획에서 위험 신호(위의 "플레이스홀더 사용 금지" 섹션 패턴들)를 검색하세요. 문제가 있다면 수정하세요.

**3. 타입 일관성 (Type consistency):** 이후 태스크에서 사용한 타입, 메서드 시그니처 및 속성 이름이 이전 태스크에서 정의한 것과 일치하나요? Task 3에서는 `clearLayers()`라고 부르고 Task 7에서는 `clearFullLayers()`라고 부른다면 버그입니다.

문제를 발견하면 인라인으로 바로 수정하세요. 재검토할 필요 없이 수정 후 다음으로 넘어가면 됩니다. 태스크가 없는 명세서 요구사항을 발견하면 태스크를 추가하세요.

## 실행 핸드오프 (Execution Handoff)

계획을 저장한 후 실행 옵션을 제시하세요:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?"**

**Subagent-Driven을 선택한 경우:**
- **필수 서브스킬:** superpowers:subagent-driven-development 사용
- 태스크당 새로운 서브에이전트 + 2단계 검토

**Inline Execution을 선택한 경우:**
- **필수 서브스킬:** superpowers:executing-plans 사용
- 검토용 체크포인트가 포함된 일괄 실행
