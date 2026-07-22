---
name: requesting-code-review
description: 태스크를 완료할 때, 주요 기능을 구현할 때, 또는 작업이 요구사항을 충족하는지 검증하기 위해 머지하기 전에 사용합니다.
---

# 코드 검토 요청하기 (Requesting Code Review)

문제가 더 많은 작업으로 확산되기 전에 이를 잡아낼 수 있도록 코드 검토자 서브에이전트를 파견하세요. 검토자는 세션 히스토리가 아닌 검증에 정교하게 제작된 컨텍스트를 전달받습니다. 이는 검토자가 당신의 사고 과정이 아닌 작업 결과물에 집중할 수 있게 하며, 지속적인 작업을 위해 사용자의 자체 컨텍스트를 보존합니다.

**핵심 원칙:** 일찍 검토하고, 자주 검토하라 (Review early, review often).

## 검토 요청 시점 (When to Request Review)

**필수:**
- 서브에이전트 기반 개발(subagent-driven development)에서 각 태스크 완료 후
- 주요 기능 완료 후
- main으로 머지하기 전

**선택 사항이지만 유용한 경우:**
- 막혔을 때 (새로운 시각 필요)
- 리팩토링 전 (기준선 점검)
- 복잡한 버그를 수정한 후

## 요청 방법 (How to Request)

**1. git SHA 가져오기:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 코드 검토자 서브에이전트 파견:**

[code-reviewer.md](code-reviewer.md) 템플릿의 내용을 채워서 `general-purpose` 서브에이전트를 파견하세요.

**플레이스홀더:**
- `{DESCRIPTION}` - 구축한 내용에 대한 간단한 요약
- `{PLAN_OR_REQUIREMENTS}` - 수행해야 할 작업
- `{BASE_SHA}` - 시작 커밋
- `{HEAD_SHA}` - 종료 커밋

**3. 피드백 조치:**
- Critical 이슈는 즉시 수정
- Important 이슈는 진행 전에 수정
- Minor 이슈는 나중을 위해 메모
- 검토자가 틀렸다면 논거를 가지고 이의 제기

## 예시 (Example)

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch code reviewer subagent]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## 워크플로우와의 통합 (Integration with Workflows)

**서브에이전트 기반 개발 (Subagent-Driven Development):**
- 각 태스크 완료 후 검토
- 문제가 복합되기 전에 잡아냄
- 다음 태스크로 이동하기 전에 수정

**계획 실행 (Executing Plans):**
- 각 태스크 후 또는 자연스러운 체크포인트에서 검토
- 피드백 수신, 적용, 지속

**임의 개발 (Ad-Hoc Development):**
- 머지 전 검토
- 막혔을 때 검토

## 주의 사항 (Red Flags)

**절대 금지:**
- "단순하다"는 이유로 검토 건너뛰기
- Critical 이슈 무시하기
- 수정되지 않은 Important 이슈가 있는 상태로 진행하기
- 타당한 기술 피드백에 대해 감정적으로 반박하기

**검토자가 틀린 경우:**
- 기술적 논거를 바탕으로 반박
- 작동을 증명하는 코드/테스트 제시
- 설명 명확화 요청

템플릿 참조: [code-reviewer.md](code-reviewer.md)
