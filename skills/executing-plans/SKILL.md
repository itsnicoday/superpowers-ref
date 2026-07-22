---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# 계획 실행하기 (Executing Plans)

## 개요

계획을 로드하고, 비판적으로 검토하며, 모든 작업을 실행하고, 완료 시 보고합니다.

**시작 시 알림:** "이 계획을 구현하기 위해 executing-plans 스킬을 사용합니다."

**참고:** 사람 파트너에게 Superpowers는 서브에이전트 접근 권한이 있을 때 훨씬 더 잘 동작함을 알려주세요. 서브에이전트를 지원하는 플랫폼(Claude Code, Codex CLI, Codex App, Copilot CLI 모두 해당됨; `../using-superpowers/references/` 의 플랫폼별 도구 참조 참조)에서 실행할 경우 작업 결과물의 품질이 훨씬 더 높아집니다. 서브에이전트를 사용할 수 있는 환경이라면 이 스킬 대신 superpowers:subagent-driven-development 스킬을 사용하세요.

## 실행 절차 (The Process)

### Step 1: 계획 로드 및 검토
1. 계획 파일 읽기
2. 비판적으로 검토하기 - 계획에 대한 질문이나 우려 사항 식별
3. 우려 사항이 있는 경우: 시작하기 전에 사람 파트너에게 이를 제기
4. 우려 사항이 없는 경우: 계획 항목에 대한 todo를 생성하고 진행

### Step 2: 작업 실행

각 작업에 대해:
1. in_progress로 표시
2. 각 단계를 정확히 이행 (계획에는 소규모 단계들이 포함됨)
3. 명시된 대로 검증 실행
4. completed로 표시

### Step 3: 개발 완료

모든 작업이 완료되고 검증된 후:
- 알림: "이 작업을 완료하기 위해 finishing-a-development-branch 스킬을 사용합니다."
- **필수 하위 스킬:** superpowers:finishing-a-development-branch 사용
- 해당 스킬에 따라 테스트를 검증하고, 옵션을 제시하며, 선택안을 실행

## 멈추고 도움을 요청해야 할 때

**다음 상황에서는 즉시 실행을 중단하세요:**
- 차단 요인(blocker) 발생 시 (의존성 누락, 테스트 실패, 불명확한 지침)
- 계획에 시작을 방해하는 치명적인 공백이 있을 때
- 지시사항을 이해하지 못하겠을 때
- 검증이 반복적으로 실패할 때

**추측하기보다는 명확한 설명을 요청하세요.**

## 이전 단계를 다시 찾아야 할 때

**다음 상황에서는 검토 단계(Step 1)로 돌아가세요:**
- 파트너가 귀하의 피드백을 바탕으로 계획을 업데이트했을 때
- 근본적인 접근 방식을 다시 제고해야 할 때

**차단 요인을 억지로 밀어붙이지 마세요** - 멈추고 물어보세요.

## 기억할 사항
- 먼저 계획을 비판적으로 검토하세요
- 계획의 단계를 정확히 따르세요
- 검증 단계를 건너뛰지 마세요
- 계획에서 지시할 때 관련 스킬을 참조하세요
- 막혔을 때는 멈추고 추측하지 마세요
- 사용자의 명시적인 동의 없이 main/master 브랜치에서 직접 구현을 시작하지 마세요

## 통합 (Integration)

**필수 워크플로우 스킬:**
- **superpowers:using-git-worktrees** - 격리된 워크스페이스 보장 (생성하거나 기존 것 검증)
- **superpowers:writing-plans** - 이 스킬이 실행하는 계획 작성
- **superpowers:finishing-a-development-branch** - 모든 작업 후 개발 완료 처리
