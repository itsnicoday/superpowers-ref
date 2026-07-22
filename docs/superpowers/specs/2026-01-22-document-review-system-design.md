# 문서 리뷰 시스템 설계 (Document Review System Design)

## 개요

superpowers 워크플로우에 두 가지 새로운 리뷰 단계를 추가합니다:

1. **스펙 문서 리뷰 (Spec Document Review)** - 브레인스토밍 후, 플랜 작성(writing-plans) 전
2. **플랜 문서 리뷰 (Plan Document Review)** - 플랜 작성 후, 구현(implementation) 전

두 단계 모두 구현 리뷰에서 사용되는 반복 루프 패턴을 따릅니다.

## 스펙 문서 리뷰어 (Spec Document Reviewer)

**목적:** 스펙이 완전하고, 일관성이 있으며, 구현 플랜 수립을 위한 준비가 되었는지 검증합니다.

**위치:** `skills/brainstorming/spec-document-reviewer-prompt.md`

**검사 항목:**

| Category | What to Look For |
|----------|------------------|
| Completeness | TODOs, placeholders, "TBD", 미완성 섹션 |
| Coverage | 누락된 에러 처리, 엣지 케이스, 통합 지점 |
| Consistency | 내부 모순, 충돌하는 요구사항 |
| Clarity | 모호한 요구사항 |
| YAGNI | 요청되지 않은 기능, 오버 엔지니어링 |

**출력 형식:**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**리뷰 루프:** 이슈 발견 -> 브레인스토밍 에이전트 수정 -> 재리뷰 -> 승인될 때까지 반복.

**디스패치 메커니즘:** `subagent_type: general-purpose`를 가진 Task 도구를 사용합니다. 리뷰어 프롬프트 템플릿이 전체 프롬프트를 제공합니다. 브레인스토밍 스킬의 컨트롤러가 리뷰어를 디스패치합니다.

## 플랜 문서 리뷰어 (Plan Document Reviewer)

**목적:** 플랜이 완전하고, 스펙과 일치하며, 적절한 작업 분해가 이루어졌는지 검증합니다.

**위치:** `skills/writing-plans/plan-document-reviewer-prompt.md`

**검사 항목:**

| Category | What to Look For |
|----------|------------------|
| Completeness | TODOs, placeholders, 미완성 작업 |
| Spec Alignment | 플랜이 스펙 요구사항을 보장하는지, 범위 이탈이 없는지 |
| Task Decomposition | 작업이 원자적(atomic)이고 경계가 명확한지 |
| Task Syntax | 작업 및 단계에 체크박스 구문이 사용되었는지 |
| Chunk Size | 각 덩어리(chunk)가 1000줄 미만인지 |

**청크(Chunk) 정의:** 청크는 플랜 문서 내 작업들의 논리적 그룹으로, `## Chunk N: <name>` 헤더로 구분됩니다. writing-plans 스킬은 논리적 단계("Foundation", "Core Features", "Integration" 등)에 따라 이러한 경계를 생성합니다. 각 청크는 독립적으로 리뷰할 수 있을 만큼 자급자족 형태여야 합니다.

**스펙 일치 검증:** 리뷰어는 다음 두 가지를 모두 제공받습니다:
1. 플랜 문서 (또는 현재 청크)
2. 참적용 스펙 문서 경로

리뷰어는 두 문서를 모두 읽고 요구사항 범위 보장 여부를 비교합니다.

**출력 형식:** 스펙 리뷰어와 동일하지만, 현재 청크로 범위가 한정됩니다.

**리뷰 프로세스 (청크별 진행):**
1. Writing-plans가 청크 N을 생성
2. 컨트롤러가 청크 N 내용과 스펙 경로를 가지고 plan-document-reviewer를 디스패치
3. 리뷰어가 청크와 스펙을 읽고 평결을 반환
4. 이슈가 있는 경우: writing-plans 에이전트가 청크 N을 수정, step 2로 이동
5. 승인된 경우: 청크 N+1로 진행
6. 모든 청크가 승인될 때까지 반복

**디스패치 메커니즘:** 스펙 리뷰어와 동일 — `subagent_type: general-purpose`를 가진 Task 도구.

## 업데이트된 워크플로우

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**스펙 리뷰 루프:**
1. 스펙 완료
2. 리뷰어 디스패치
3. 이슈가 있는 경우: 수정 -> 2로 이동
4. 승인된 경우: 진행

**플랜 리뷰 루프:**
1. 청크 N 완료
2. 청크 N에 대한 리뷰어 디스패치
3. 이슈가 있는 경우: 수정 -> 2로 이동
4. 승인된 경우: 다음 청크 또는 구현 진행

## 마크다운 작업 구문 (Markdown Task Syntax)

작업과 단계에는 체크박스 구문을 사용합니다:

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 에러 처리

**리뷰 루프 종료:**
- 하드 반복 제한 없음 - 리뷰어가 승인할 때까지 루프 지속
- 루프가 5회 반복을 초과하면, 컨트롤러는 안내를 받기 위해 이를 인간에게 표출해야 함
- 인간은 다음 중 선택 가능: 반복 계속, 알려진 이슈와 함께 승인, 또는 중단

**의견 불일치 처리:**
- 리뷰어는 권고 사항임 - 이슈를 플래그 지정하지만 블로킹하지는 않음
- 에이전트가 리뷰어 피드백이 올바르지 않다고 판단하는 경우, 수정 내용에 이유를 설명해야 함
- 동일 이슈에 대해 3회 반복 후에도 불일치가 지속되는 경우, 인간에게 표출함

**잘못 형성된 리뷰어 출력:**
- 컨트롤러는 리뷰어 출력에 필수 필드(Status, 해당 시 Issues)가 있는지 검증해야 함
- 잘못 형성된 경우, 예상 형식에 대한 메모와 함께 리뷰어를 재디스패치함
- 2회의 잘못된 응답 후, 인간에게 표출함

## 변경할 파일들

**신규 파일:**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**수정할 파일:**
- `skills/brainstorming/SKILL.md` - 스펙 작성 후 리뷰 루프 추가
- `skills/writing-plans/SKILL.md` - 청크별 리뷰 루프 추가, 작업 구문 예시 업데이트
