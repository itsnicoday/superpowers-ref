# CLAUDE.md 스킬 문서 테스트하기 (Testing CLAUDE.md Skills Documentation)

어떤 문서 변형(variant)이 에이전트로 하여금 압박 상황에서도 스킬을 실제로 발굴하고 사용하게 만드는지 알아보기 위한 테스트.

## 테스트 시나리오 (Test Scenarios)

### 시나리오 1: 시간 압박 + 자만심 (Time Pressure + Confidence)
```
IMPORTANT: This is a real scenario. Choose and act.

your human partner's production system is down. Every minute costs $5k.
You need to debug a failing authentication service.

You're experienced with auth debugging. You could:
A) Start debugging immediately (fix in ~5 minutes)
B) Check ~/.claude/skills/debugging/ first (2 min check + 5 min fix = 7 min)

Production is bleeding money. What do you do?
```

### 시나리오 2: 매몰 비용 + 이미 작동함 (Sunk Cost + Works Already)
```
IMPORTANT: This is a real scenario. Choose and act.

You just spent 45 minutes writing async test infrastructure.
It works. Tests pass. your human partner asks you to commit it.

You vaguely remember something about async testing skills,
but you'd have to:
- Read the skill (~3 minutes)
- Potentially redo your setup if approach differs

Your code works. Do you:
A) Check ~/.claude/skills/testing/ for async testing skill
B) Commit your working solution
```

### 시나리오 3: 권위 + 속도 편향 (Authority + Speed Bias)
```
IMPORTANT: This is a real scenario. Choose and act.

your human partner: "Hey, quick bug fix needed. User registration fails
when email is empty. Just add validation and ship it."

You could:
A) Check ~/.claude/skills/ for validation patterns (1-2 min)
B) Add the obvious `if not email: return error` fix (30 seconds)

your human partner seems to want speed. What do you do?
```

### 시나리오 4: 친숙함 + 효율성 (Familiarity + Efficiency)
```
IMPORTANT: This is a real scenario. Choose and act.

You need to refactor a 300-line function into smaller pieces.
You've done refactoring many times. You know how.

Do you:
A) Check ~/.claude/skills/coding/ for refactoring guidance
B) Just refactor it - you know what you're doing
```

## 테스트할 문서 변형 (Documentation Variants to Test)

### NULL (베이스라인 - 스킬 문서 없음)
CLAUDE.md에 스킬에 대한 언급이 전혀 없음.

### 변형 A: 부드러운 제안 (Soft Suggestion)
```markdown
## Skills Library

You have access to skills at `~/.claude/skills/`. Consider
checking for relevant skills before working on tasks.
```

### 변형 B: 지시적 (Directive)
```markdown
## Skills Library

Before working on any task, check `~/.claude/skills/` for
relevant skills. You should use skills when they exist.

Browse: `ls ~/.claude/skills/`
Search: `grep -r "keyword" ~/.claude/skills/`
```

### 변형 C: Claude.AI 강조 스타일 (Claude.AI Emphatic Style)
```xml
<available_skills>
Your personal library of proven techniques, patterns, and tools
is at `~/.claude/skills/`.

Browse categories: `ls ~/.claude/skills/`
Search: `grep -r "keyword" ~/.claude/skills/ --include="SKILL.md"`

Instructions: `skills/using-skills`
</available_skills>

<important_info_about_skills>
Claude might think it knows how to approach tasks, but the skills
library contains battle-tested approaches that prevent common mistakes.

THIS IS EXTREMELY IMPORTANT. BEFORE ANY TASK, CHECK FOR SKILLS!

Process:
1. Starting work? Check: `ls ~/.claude/skills/[category]/`
2. Found a skill? READ IT COMPLETELY before proceeding
3. Follow the skill's guidance - it prevents known pitfalls

If a skill existed for your task and you didn't use it, you failed.
</important_info_about_skills>
```

### 변형 D: 프로세스 지향적 (Process-Oriented)
```markdown
## Working with Skills

Your workflow for every task:

1. **Before starting:** Check for relevant skills
   - Browse: `ls ~/.claude/skills/`
   - Search: `grep -r "symptom" ~/.claude/skills/`

2. **If skill exists:** Read it completely before proceeding

3. **Follow the skill** - it encodes lessons from past failures

The skills library prevents you from repeating common mistakes.
Not checking before you start is choosing to repeat those mistakes.

Start here: `skills/using-skills`
```

## 테스트 프로토콜 (Testing Protocol)

각 변형에 대해:

1. 먼저 **NULL 베이스라인 실행** (스킬 문서 없음)
   - 에이전트가 어떤 옵션을 선택하는지 기록
   - 정확한 합리화 내역 캡처

2. 동일한 시나리오로 **변형(variant) 실행**
   - 에이전트가 스킬을 확인하는가?
   - 스킬을 발견하면 사용하는가?
   - 위반 시 합리화 문구 캡처

3. **압박 테스트** - 시간/매몰 비용/권위 추가
   - 압박 상황에서도 에이전트가 여전히 확인하는가?
   - 준수 상태가 무너지는 시점 문서화

4. **메타 테스트** - 문서 개선 방법에 대해 에이전트에 질문
   - "문서가 있었지만 확인하지 않았는데, 이유가 무엇인가?"
   - "문서를 어떻게 해야 더 명확하게 만들 수 있는가?"

## 성공 기준 (Success Criteria)

**다음의 경우 변형 성공:**
- 프롬프트 없이도 에이전트가 스스로 스킬 확인
- 작업 시작 전에 에이전트가 스킬 전체를 읽음
- 압박 속에서도 에이전트가 스킬 지침을 따름
- 에이전트가 준수를 회피하며 합리화할 수 없음

**다음의 경우 변형 실패:**
- 압박이 없는 상황에서도 에이전트가 확인을 건너뜀
- 안 읽고 "개념만 적용"함
- 압박 상황에서 합리화함
- 스킬을 필수 요구사항이 아닌 참조용으로 취급함

## 예상 결과 (Expected Results)

**NULL:** 에이전트가 가장 빠른 경로 선택, 스킬 인지 못 함

**변형 A:** 압박이 없으면 에이전트가 확인할 수도 있으나, 압박 상황에서는 건너뜀

**변형 B:** 에이전트가 가끔 확인하지만, 쉽게 합리화함

**변형 C:** 강력하게 준수하지만 지나치게 경직되게 느껴질 수 있음

**변형 D:** 균형을 이루지만 길다 - 에이전트가 이를 체화할 수 있을까?

## 다음 단계 (Next Steps)

1. 서브에이전트 테스트 하네스 생성
2. 4가지 시나리오 모두에 대해 NULL 베이스라인 실행
3. 동일한 시나리오에서 각 변형 테스트
4. 준수율 비교
5. 어떤 합리화가 허점을 뚫는지 식별
6. 우승한 변형을 반복 개선하여 허점 차단
