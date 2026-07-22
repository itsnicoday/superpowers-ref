# 스킬 지침의 긍정적 지시사항 재설계 — 설계 스펙 (Design Spec)

**상태:** 제안됨 (2026-06-09 SDD 리뷰 디스패치 작업의 후속 작업; 1PR당 1문제 규칙에 따라 별도 PR로 진행)  
**동기:** 스킬 산문의 일부 부정적 지시사항은 역효과를 내는 반면 다른 것들은 잘 작동하며, 그 차이가 예측 가능하다는 측정된 증거(2026-06-10)에 기반함.

## 이 스펙이 일반화하는 측정된 발견

2026-06-10 마이크로 테스트(opus, 표현 방식당 5회 반복, 프로그래밍 방식 점수 산출; 아래에 하네스 설명)를 통해 지침 표현 방식이 컨트롤러가 구상하는 내용을 어떻게 바꾸는지 측정했습니다:

| Case | Phrasing | Result |
|---|---|---|
| 디스패치 구상 ("don't restate the brief") | 금지형 (prohibition) | **4.4** 스펙 값이 다시 입력됨 — *지침이 없는 경우*(3.6)보다 악화됨 |
| 디스패치 구상 | 긍정적 레시피 ("your dispatch should contain: (1)…(5)") | **3.0, 변동성 0** — 채택됨 |
| 디스패치 구상 | 레시피 + 뉘앙스 조항 ("quote only the fragment…") | 3.8, 노이즈 있음 — 뉘앙스가 레시피를 희석시킴 |
| 테스트 재실행 지침 ("do not ask reviewer to re-run tests") | 금지형 | **위반 0/5** — 잘 작동함 (대조군: 3/5) |
| 테스트 재실행 지침 | 긍정적 레시피 | 0/5 — 동일함, 그러나 더 긺 |

**교리 (The doctrine)** (부정적 지시사항을 분류할 때 사용할 규칙):

1. **트립와이어(Tripwires)는 작동합니다.** 구체적인 토큰에 대한 문장 수준의 자체 검사("작성 중인 프롬프트에 'do not flag'가 포함되어 있다면… 중단하라")는 신뢰성 있게 작동합니다.
2. **인식 테이블(Recognition tables)은 작동합니다.** 구상 시점이 아닌 결정 시점에 읽히는 Red-Flags/합리화 테이블은 잘 기능합니다.
3. **단일 지시 금지형은 작동합니다.** 모델에 Y를 수행할 경합 유인이 없을 때 "X에게 Y를 하도록 요청하지 마라"는 유효합니다.
4. **구상 금지형은 역효과를 냅니다.** 모델이 출력에 대해 자체적인 목적을 가지고 있을 때(예: 스펙 재진술이 유용한 큐레이션처럼 느껴짐) 금지형은 역효과를 냅니다. 오직 긍정적 구상 레시피만이 이를 개선할 수 있으며 — 성공적인 레시피에 뉘앙스 조항을 추가하면 더 나빠집니다.
5. **동등한 경우 더 짧은 표현이 승리합니다.** Codex는 긴 세션 동안 SKILL.md를 ~500회 재읽습니다 (2026-06-10 측정); 산문 길이는 실제 비용입니다.

## 감사 결과 (2026-06-10, 전체 ~30개 스킬 + 프롬프트 템플릿)

집계: 3개 트립와이어 (유지), 14개 인식 테이블 (유지), ~20개 정책 게이트 (유지 — "허가 없이 절대 푸시하지 마라"는 구상 형태가 아닌 정책임), 5개 구상 금지형:

| # | Location | Disposition |
|---|---|---|
| 1 | `subagent-driven-development/task-reviewer-prompt.md` — "Cite, don't narrate" | **PR #1717 배치에 큐로 들어감**: 긍정적 절반을 전면에 배치 ("Your report should point at evidence: file:line for every finding…"), 금지형 절반은 삭제 (쓸모없는 무게 — 긍정적 절반이 이미 존재하고 역할을 수행함) |
| 2 | `subagent-driven-development/SKILL.md` — "Do not add open-ended directives" | **현상 유지**: 마이크로 테스트에서 15개 샘플 동안 실패를 유도할 수 없었음; 어느 쪽이든 증거 없음; 더 짧은 것이 승리함 |
| 3 | `subagent-driven-development/SKILL.md` — "Do not ask a reviewer to re-run tests" | **현상 유지**: 위반 0/5 측정됨; 금지형이 스스로를 디스패치에 유용하게 전파함 |
| 4 | `subagent-driven-development/SKILL.md` — "do not re-review on top of it" | **PR #1717 배치에 큐로 들어감**: 3개 요소 체크리스트로 교체 ("Before re-dispatching the reviewer, confirm the fix report contains: the covering tests, the command run, and the output") |
| 5 | `writing-plans/SKILL.md` — "No Placeholders" 금지 패턴 목록 | **이 스펙의 주요 대상** — 아래 참조 |

경계선 상의 항목, #5와 함께 연기됨: `task-reviewer-prompt.md` "Don't flag pre-existing file sizes — focus on what this change contributed" (긍정적 절반이 존재하고 주요 역할을 함; 영향 낮음; 편리하다면 #5와 함께 테스트).

## writing-plans 변경사항 (연기된 항목 #5)

### 현재 상태

`skills/writing-plans/SKILL.md`, "No Placeholders": 하나의 긍정적 문장("Every step must contain the actual content an engineer needs")에 이어 6개 불릿으로 구성된 금지 패턴 목록("never write them: 'TBD', 'TODO', 'Add appropriate error handling', 'Write tests for the above', 'Similar to Task N', …")이 이어짐.

### 이것이 중요한 이유와 진정으로 불확실한 이유

- 플랜은 워크플로우에서 **가장 큰 생성 아티팩트**이며, 모델은 자리표시자를 출력하려는 실제 경합 유인을 가집니다 (길이 압박 하에서 가장 노력이 적게 드는 경로임) — 금지형이 측정 가능하게 역효과를 낸 사례와 동일한 유인 구조입니다.
- 그러나 금지된 항목들은 **단일하고 인지 가능한 토큰**입니다 — 금지형이 측정 가능하게 효과를 나타낸 사례의 형태입니다.
- **이 목록은 다른 곳에서 주요한 역할을 합니다:** 스킬의 Self-Review 섹션이 이를 참조합니다 ("Placeholder scan: search your plan for red flags — any of the patterns from the 'No Placeholders' section above"). 이 토큰들은 리뷰 시점 스캔 목록의 역할도 수행하며, 리뷰 시점 인지는 작동하는 범주입니다. 긍정적 체크리스트로 성급히 교체하면 해당 참조가 깨지고 유용한 트립와이어 토큰이 버려집니다.

### 테스트할 변형 (Variants)

- **V0 (현재):** 구상 시점의 긍정적 문장 + 금지 목록; Self-Review에서 해당 목록 참조.
- **V1 (감사자의 체크리스트):** 구상 시점의 긍정적 레시피만 사용 — "Before finalizing a step, confirm it has: the literal code to write, a runnable command with expected output, types and method names defined within this plan, error handling shown explicitly. A step is complete when an engineer could implement it without asking any follow-up questions." Self-Review는 일반적인 자리표시자 스캔을 유지.
- **V2 (메커니즘에 따른 재구조화 — 승리 예측 모델):** 구상 시점에는 V1의 긍정적 레시피만 배치; 지칭된 패턴들은 Self-Review의 자리표시자 스캔 단계로 통째로 이동하여 인지 관점으로 재구성됨 ("when you scan, look for: 'TBD', 'TODO', 'Similar to Task N', …"). 토큰은 동일하지만, 자극을 유발하는 범주에서 감지하는 범주로 위치가 변경됨.
- **V3 (대조군):** 긍정적 문장만 사용, 어디에도 목록 없음.

### 마이크로 테스트 설계

- **작업:** opus가 의도적으로 미흡하게 작성된 스펙으로부터 2-3개 작업의 구현 플랜을 작성함 (미흡한 작성이 자리표시자를 유도함). 테스트용 스펙 사용: 명확히 규정된 작업 1개, 에러 처리를 스펙이 얼렁뚱땅 넘긴 작업 1개, 첫 번째 작업과 유사한 작업 1개 ("Similar to Task 1" 유도).
- **샘플링:** 변형당 5회 이상 반복, 기본 온도, 모델 `claude-opus-4-8` (실제 플랜을 작성하는 모델).
- **프로그래밍 방식 점수 산출** (별도 표기가 없으면 낮을수록 좋음):
  - 금지 토큰 수: `TBD|TODO|implement later|fill in details|appropriate error handling|handle edge cases|Similar to Task|Write tests for the above`
  - 코드를 변경하지만 코드 블록이 누락된 단계 수
  - 플랜 출력 어디에도 정의되지 않은 타입/함수 참조 수
  - (높을수록 좋음) 작업당 예상 출력이 포함된 실행 가능한 명령어 수
- **V2를 위한 2단계 점수 산출:** Self-Review 절반도 테스트 — 각 생성된 플랜을 해당 변형의 Self-Review 섹션과 함께 다시 제공하여 스캔이 심어진 자리표시자를 실제로 포착하는지 측정 (테스트 플랜에 알고 있는 자리표시자 2개 삽입; 감지율이 지표임).
- **수락 기준:** 코드 블록 커버리지나 self-review 감지율 손실 없이 금지 토큰 수에서 V0를 이기는 변형만 채택함. 예상 비용: 총 ~$6-10.

### PR 범위 지정

별도 PR (writing-plans는 다른 스킬입니다; 해당 "No Placeholders" 목록은 기여 가이드라인이 이발 증거를 요구하는 조정된 콘텐츠입니다). PR에는 마이크로 테스트 하네스 + 결과 테이블, 변경 전/후 텍스트, V2 위치 변경 근거가 포함되어야 합니다.

## 마이크로 테스트 하네스 (유실되지 않도록 보존하는 방법론)

`/tmp/sdd-exp/micro/run-micro.py` 및 `/tmp/sdd-exp/micro2/run-micro2.py`
(2026-06-10; superpowers-evals에 `docs/superpowers/skills/micro-testing-prompt-guidance.md` + 스크립트로 커밋 예정):

- 샘플당 한 번의 API 호출: 시스템 프롬프트 = 현실적인 주변 컨텍스트 내의 스킬 지침 변형; 사용자 = 현실적인 워크플로우 중간 시나리오; 출력 = 구상된 아티팩트 (디스패치 프롬프트, 플랜, 리포트).
- 명확한 마커를 grep하여 프로그래밍 방식으로 점수 산출; **평결을 믿기전에 모든 일치 항목을 수동으로 점검하십시오** — 오늘 밤의 "위반" 중 하나는 컨트롤러가 금지 사항을 올바르게 인용한 것이었고, 자동화된 부정 감지가 다른 일치 항목에 잘못된 라벨을 붙였습니다.
- 샘플당 ~$0.15-0.30, 45분짜리 전체 이발 실행 시 $12가 드는 반면 반복당 수초 소요. 여기서 표현 방식을 반복 수정합니다; 변경 사항이 구조적일 때만 전체 실행에서 승자를 확정합니다.
- 항상 지침이 없는 대조군을 포함하십시오 — 오늘 밤 대조군은 역효과(재진술: 금지가 아무것도 없는 것보다 나쁨)와 유효한 금지(테스트 재실행: 3/5 대조군 실패 대 두 표현 방식 모두 0/5)를 모두 밝혀냈습니다.

## 결과: writing-plans 마이크로 테스트 (이 스펙 작성 후 2026-06-10 실행)

**해결됨 — 변경 불필요.** Stage 1 (3개 작업 스펙, 압박 없음): 지침이 없는 대조군을 포함하여 4개 변형 전체의 20개 플랜 모두에서 자리표시자 0개. Stage 1b (10개 작업 스펙, "Similar to Task N"을 유도하는 5개의 거의 동일한 명령어, 명시적 ~2,500자 절약 목표): 40/40 깔끔함 — 유일한 정규식 일치 항목은 V2 self-review가 "TBD/TODO 없음 ✓"을 *증명*한 것이었습니다. 현재 세대의 opus는 금지 패턴 목록의 유무와 관계없이 의도적인 압박 하에서도 플랜 자리표시자를 생성하지 않습니다. 조치: No Placeholders 섹션을 있는 그대로 유지 (비용이 거의 들지 않으며 반대 가설은 측정 불가능함); 후속 PR을 열지 마십시오. V2 위치 변경 설계는 향후 모델 세대가 퇴보할 경우를 대비해 여기에 기록으로 남겨둡니다.

## 또한 명시적으로 폐기되지 않음 (테스트되었으며 거부됨, 데이터 포함)

새로운 증거 없이 아무도 재제안하지 않도록 기록됨 — 전체 수치는 2026-06-09 SDD 설계 스펙의 Cost-iterations 섹션에 있음:

- **컨트롤러 turn 배치 처리 / 한 메시지 내 병렬 도구 호출:** 컨트롤러는 메시지당 정확히 하나의 도구 호출을 출력합니다 (지침 유무와 관계없이 측정된 모든 실행에서 0개의 다중 도구 메시지). 컨트롤러 turn의 46%는 도구 호출이 없는 사고/서술입니다 — 프롬프트로 줄일 수 없는 하한선입니다.
- **병렬 호출을 통한 파이프라인 리뷰:** 동일한 이유로 폐기됨.
- **`run_in_background`를 통한 파이프라인 리뷰:** 메커니즘이 제공될 때 채택되었으나 (7/28 디스패치) 45분 시나리오에서 이점이 노이즈 하한선 이하였음 (리뷰는 개당 ~30-60초에 불과함); 이중 결과 스트림 조정이 추가됨. 리뷰가 개별적으로 긴 플랜에 대해서만 다시 검토할 가치가 있습니다.
- **성공적인 레시피에 덧붙여진 뉘앙스 조항:** 측정 가능하게 악화시킴 (C2: 3.8 노이즈 있음 대 C: 3.0 일관됨). 단서 조항을 덧붙이지 말고 레시피를 다시 도출하여 수정하십시오.
