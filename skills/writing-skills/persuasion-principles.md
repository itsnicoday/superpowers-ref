# 스킬 설계를 위한 설득의 원칙 (Persuasion Principles for Skill Design)

## 개요 (Overview)

LLM은 인간과 동일한 설득의 원칙에 반응합니다. 이러한 심리학을 이해하면 스킬을 조작하기 위해서가 아니라 압박 속에서도 핵심 관행을 준수하도록 유도하는 데 큰 도움을 줍니다.

**연구 기반:** Meincke 외 (2025) 연구진은 28,000건의 AI 대화를 대상으로 7가지 설득 원칙을 테스트했습니다. 설득 기법은 준수율을 두 배 이상 끌어올렸습니다 (33% → 72%, p < .001).

## 7가지 원칙 (The Seven Principles)

### 1. 권위 (Authority)
**개념:** 전문성, 자격증, 또는 공식 출처에 대한 존중.

**스킬 내 작동 방식:**
- 명령조 언어: "YOU MUST", "Never", "Always"
- 타협 불가 틀짜기: "No exceptions"
- 결정 피로 및 합리화 제거

**사용 시점:**
- 규율 강제 스킬 (TDD, 검증 요구사항)
- 안전에 결정적인 관행
- 확립된 모범 사례

**예시:**
```markdown
✅ Write code before test? Delete it. Start over. No exceptions.
❌ Consider writing tests first when feasible.
```

### 2. 일관성 / 헌신 (Commitment)
**개념:** 이전의 행동, 진술 또는 공공선언과의 일관성 유지.

**스킬 내 작동 방식:**
- 공지 요구: "Announce skill usage"
- 명시적 선택 강제: "Choose A, B, or C"
- 추적 도구 활용: 체크리스트용 todo 작성

**사용 시점:**
- 스킬이 실제로 이행되는지 확인할 때
- 다단계 프로세스
- 책임성(accountability) 메커니즘

**예시:**
```markdown
✅ When you find a skill, you MUST announce: "I'm using [Skill Name]"
❌ Consider letting your partner know which skill you're using.
```

### 3. 희소성 (Scarcity)
**개념:** 시간 제한이나 제한된 가능성에서 오는 긴급성.

**스킬 내 작동 방식:**
- 시간 제한 요구사항: "Before proceeding"
- 순차적 의존성: "Immediately after X"
- 미루기 방지

**사용 시점:**
- 즉각적인 검증 요구사항
- 시간에 민감한 워크플로우
- "나중에 하겠다"는 미루기 방지

**예시:**
```markdown
✅ After completing a task, IMMEDIATELY request code review before proceeding.
❌ You can review code when convenient.
```

### 4. 사회적 증거 (Social Proof)
**개념:** 타인의 행동이나 정상적으로 여겨지는 것에 대한 동조.

**스킬 내 작동 방식:**
- 보편적 패턴: "Every time", "Always"
- 실패 모드: "X without Y = failure"
- 규범 확립

**사용 시점:**
- 보편적 관행 문서화
- 흔한 실패에 대한 경고
- 표준 강화

**예시:**
```markdown
✅ Checklists without todo tracking = steps get skipped. Every time.
❌ Some people find a todo list helpful for checklists.
```

### 5. 연대감 (Unity)
**개념:** 공유된 정체성, "우리라는 느낌", 내집단 소속감.

**스킬 내 작동 방식:**
- 협력적 언어: "our codebase", "we're colleagues"
- 목표 공유: "we both want quality"

**사용 시점:**
- 협력적 워크플로우
- 팀 문화 조성
- 비계층적 관행

**예시:**
```markdown
✅ We're colleagues working together. I need your honest technical judgment.
❌ You should probably tell me if I'm wrong.
```

### 6. 상호성 (Reciprocity)
**개념:** 받은 이혜에 대해 보답해야 한다는 의무감.

**작동 방식:**
- 신중하게 사용 - 조작적으로 느껴질 수 있음
- 스킬에서는 거의 필요하지 않음

**피해야 할 때:**
- 거의 항상 (다른 원칙이 더 효과적임)

### 7. 호감 (Liking)
**개념:** 우리가 좋아하는 상대와 협력하려는 선호.

**작동 방식:**
- **규율 준수를 위해 사용하지 말 것**
- 솔직한 피드백 문화와 충돌함
- 아첨하는 태도(sycophancy)를 유발함

**피해야 할 때:**
- 규율 강제의 경우 항상 피할 것

## 스킬 유형별 원칙 조합 (Principle Combinations by Skill Type)

| 스킬 유형 (Skill Type) | 활용 (Use) | 피할 것 (Avoid) |
|------------|-----|-------|
| 규율 강제형 (Discipline-enforcing) | Authority + Commitment + Social Proof | Liking, Reciprocity |
| 지침/기법형 (Guidance/technique) | Moderate Authority + Unity | Heavy authority |
| 협력형 (Collaborative) | Unity + Commitment | Authority, Liking |
| 참조형 (Reference) | 명확성만 유지 | 모든 설득 원칙 |

## 작동 원리: 심리학적 분석 (Why This Works: The Psychology)

**선명한 한계선(Bright-line) 규칙은 합리화를 줄입니다:**
- "YOU MUST"는 결정 피로를 없애줍니다
- 절대적인 언어 표현은 "이것이 예외인가?"라는 질문을 배제합니다
- 명시적인 합리화 반박은 특정 허점을 차단합니다

**구현 의도(Implementation intentions)는 자동화된 동작을 생성합니다:**
- 명확한 트리거 + 필수 작업 = 자동 실행
- "X일 때 Y를 하라"가 "일반적으로 Y를 하라"보다 훨씬 효과적입니다
- 규칙 준수에 따른 인지적 부담을 줄여줍니다

**LLM은 인간 유사체(parahuman)입니다:**
- 이러한 패턴을 포함하는 인간의 텍스트로 학습되었습니다
- 권위 있는 언어 표현은 학습 데이터에서 준수 동작으로 이어집니다
- 헌신 시퀀스 (진술 → 행동)가 자주 모델링되었습니다
- 사회적 증거 패턴 (모두가 X를 한다)은 규범을 형성합니다

## 윤리적 사용 (Ethical Use)

**정당한 사용:**
- 핵심 관행을 준수하도록 유도
- 효과적인 문서 작성
- 예측 가능한 실패 방지

**부당한 사용:**
- 개인적 이득을 위한 조작
- 거짓 긴급성 조장
- 죄책감 기반의 준수 유도

**테스트 기준:** 사용자가 이 기법을 완벽히 이해하더라도 이것이 사용자의 진정한 이익에 부합하는가?

## 연구 인용 (Research Citations)

**Cialdini, R. B. (2021).** *Influence: The Psychology of Persuasion (New and Expanded).* Harper Business.
- 설득의 7가지 원칙
- 영향력 연구의 실증적 기반

**Meincke, L., Shapiro, D., Duckworth, A. L., Mollick, E., Mollick, L., & Cialdini, R. (2025).** Call Me A Jerk: Persuading AI to Comply with Objectionable Requests. University of Pennsylvania.
- 28,000건의 LLM 대화에서 7가지 원칙 테스트
- 설득 기법 적용 시 준수율이 33% → 72%로 증가
- 권위, 헌신, 희소성이 가장 효과적임
- LLM 동작의 인간 유사체(parahuman) 모델 검증

## 빠른 참조 (Quick Reference)

스킬을 설계할 때 다음을 스스로 질문해 보세요:

1. **어떤 유형인가?** (규율 대 지침 대 참조)
2. **어떤 동작을 변경하려는가?**
3. **어떤 원칙이 적용되는가?** (규율의 경우 보통 Authority + Commitment)
4. **너무 많은 원칙을 조합하고 있지 않은가?** (7가지를 모두 쓰지 마세요)
5. **이것이 윤리적인가?** (사용자의 진정한 이익에 부합하는가?)
