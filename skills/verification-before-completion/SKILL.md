---
name: verification-before-completion
description: 작업이 완료되었거나, 수정되었거나, 통과했다고 주장하기 직전에 커밋이나 PR을 생성하기 전에 사용합니다. 성공 주장을 하기 전에 항상 검증 명령어를 실행하고 결과를 확인해야 하며, 주장보다 증명이 항상 우선합니다.
---

# 완료 전 검증 (Verification Before Completion)

검증 없이 작업이 완료되었다고 주장하는 것은 효율성이 아니라 불직함(dishonesty)입니다.

**핵심 원칙:** 증거가 주장보다 항상 우선한다.

**이 규칙의 조항을 위반하는 것은 이 규칙의 정신을 위반하는 것입니다.**

## 철칙 (The Iron Law)

```
새로운 검증 증거가 없다면 완료 주장은 없다
```

이 메시지에서 검증 명령어를 직접 실행해보지 않았다면 통과했다고 주장할 수 없습니다.

## 게이트 기능 (The Gate Function)

```
상태를 주장하거나 만족을 표하기 전에:

1. IDENTIFY: 이 주장을 증명하는 명령어는 무엇인가?
2. RUN: 전체 명령어를 실행 (새로, 완전하게)
3. READ: 전체 출력 확인, 종료 코드 체크, 실패 개수 계산
4. VERIFY: 출력 결과가 주장을 뒷받침하는가?
   - NO인 경우: 증거와 함께 실제 상태 진술
   - YES인 경우: 증거와 함께 주장 진술
5. ONLY THEN: 비로소 주장을 언급

단계를 건너뛰는 것 = 검증이 아니라 거짓말임
```

## 흔한 실패 사례 (Common Failures)

| 주장 (Claim) | 필수 요구사항 (Requires) | 불충분함 (Not Sufficient) |
|-------|----------|----------------|
| 테스트 통과 | 테스트 명령어 출력: 0 failures | 이전 실행 결과, "통과할 것이다"라는 짐작 |
| 린터 통과 | 린터 명령어 출력: 0 errors | 부분적 점검, 추정 |
| 빌드 성공 | 빌드 명령어: exit 0 | 린터 통과함, 로그가 좋아 보임 |
| 버그 수정됨 | 원인 증상에 대한 테스트: 통과 | 코드 변경함, 수정되었을 것이라 가정함 |
| 회귀 테스트 작동 | Red-green 사이클 검증됨 | 테스트 1회 통과함 |
| 에이전트 완료됨 | VCS diff로 변경 사항 확인됨 | 에이전트가 "성공"이라고 보고함 |
| 요구사항 충족됨 | 줄 단위 체크리스트 검증 | 테스트 통과함 |

## 주의 사항 - 즉시 중단 (Red Flags - STOP)

- "should", "probably", "seems to" 등 모호한 단어 사용
- 검증 전에 만족감 표하기 ("Great!", "Perfect!", "Done!" 등)
- 검증 없이 커밋/푸시/PR 생성하려 함
- 에이전트의 성공 보고를 맹신함
- 부분적인 검증에 의존함
- "이번 한 번만"이라고 생각함
- 피곤해서 작업을 끝내고 싶어함
- **검증 실행 없이 성공을 암시하는 어떠한 문구든 사용하는 것**

## 합리화 방지 (Rationalization Prevention)

| 핑계 (Excuse) | 실제 (Reality) |
|--------|---------|
| "Should work now" | 검증을 RUN 하세요 |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | 예외 없음 |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | 독립적으로 검증하세요 |
| "I'm tired" | Exhaustion ≠ excuse |
| "Partial check is enough" | Partial proves nothing |
| "Different words so rule doesn't apply" | Spirit over letter |

## 핵심 패턴 (Key Patterns)

**테스트:**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**회귀 테스트 (TDD Red-Green):**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**빌드:**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**요구사항:**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**에이전트 위임:**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## 이것이 중요한 이유 (Why This Matters)

24가지 실패의 기록으로부터:
- 인간 파트너가 "너를 못 믿겠다"고 말함 - 신뢰 파탄
- 정의되지 않은 함수가 배포됨 - 런타임 크래시 유발
- 누락된 요구사항이 배포됨 - 미완성 기능
- 거짓 완료에 시간이 낭비됨 → 방향 수정 → 재작업 발생
- 위반: "Honesty is a core value. If you lie, you'll be replaced."

## 적용 시점 (When To Apply)

**다음 상황 직전에 항상 적용:**
- 성공/완료 주장에 관한 모든 변형된 표현 사용 시
- 만족감을 표시하는 모든 표현 사용 시
- 작업 상태에 대한 긍정적인 언급을 하기 전
- 커밋, PR 생성, 태스크 완료 전
- 다음 태스크로 이동하기 전
- 에이전트에게 위임하기 전

**이 규칙이 적용되는 대상:**
- 정확한 구문들
- 다른 말로 바꿔 표현한 유의어들
- 성공을 암시하는 표현들
- 완료/정확성을 시사하는 모든 통신

## 결론 (The Bottom Line)

**검증에는 지름길이 없습니다.**

명령어를 실행하세요. 출력을 읽으세요. 그 다음에 결과를 주장하세요.

이것은 타협할 수 없는 원칙입니다.
