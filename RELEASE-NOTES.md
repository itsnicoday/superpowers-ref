# Superpowers 릴리즈 노트 (Release Notes)

## v6.1.1 (2026-07-02)

### Codex

- **Codex가 더 이상 Claude SessionStart 훅을 재등록하지 않습니다.** v6.1.0에서 Codex 훅 설정 및 매니페스트 `hooks` 포인터가 제거되었으나, `hooks` 필드가 없자 Codex가 저장소 루트의 마켓플레이스에서 제공하는 Claude Code SessionStart 훅인 `hooks/hooks.json`을 자동 감지(auto-discover)하여 설치 시 신뢰 프롬프트와 함께 재등록하는 문제가 있었습니다. 이제 Codex 매니페스트는 명시적인 빈 훅 객체(`hooks: {}`)를 선언하여 Codex가 자동 감지 폴백으로 이동하는 대신 "훅 없음"으로 읽도록 합니다. 필드가 누락되었거나 `[]` 또는 빈 인라인 리스트는 모두 폴백으로 축소되므로 값은 정확히 `{}`이어야 합니다.
- **고립된 Codex session-start 데드 코드를 제거했습니다.** Codex 훅 설정이 삭제된 후 `hooks/session-start-codex`는 호출자가 없었으므로 해당 파일 및 불필요한 테스트 케이스가 제거되었습니다. `docs/porting-to-a-new-harness.md`에 기재된 쉘 훅 작동 예제는 Codex(현재 session-start 훅이 없는 네이티브 스킬 탐색)에서 Cursor(라이브 쉘 훅 하네스)로 이동되었으며, `docs/windows/polyglot-hooks.md`의 오래된 `hooks-codex.json` 포인터가 수정되었습니다. 또한 Codex 플러그인 카테고리가 "Developer Tools"로 수정되었습니다.

### Packaging

- **Codex 포털 패키지 빌드를 위한 새로운 `package-codex-plugin.sh` 추가.** 메인테이너 스크립트가 엔트리 타임스탬프를 정규화하고, 실행 모드를 보존하며, 패키징된 모든 스킬이 OpenAI 메타데이터를 제공하는지 검증하고, 앱 및 작성자 아이콘을 포함하며, 더티 작업 트리(dirty worktree)에 대한 실행을 거부하는 결정론적 Codex "포털" 아카이브(기본값 `.zip`, 요청 시 `tar.gz`)를 생성합니다. 패키징된 매니페스트는 소스 `hooks: {}` 객체를 유지하여 포털에 설치된 플러그인이 동일한 SessionStart 자동 감지를 회피하며, 스크립트는 저장된 메타데이터 소스로부터 바이너리 수준에서 동일한 아카이브를 재빌드할 수 있습니다. 새로운 테스트 수트로 검증되었습니다.

## v6.1.0 (2026-06-30)

### 세션당 토큰 비용 절감

`using-superpowers` 부트스트랩은 모든 세션에 주입되므로 크기가 일정하게 소모됩니다. 이번 릴리즈에서는 동작 형성 콘텐츠를 제거하지 않으면서 부트스트랩과 참조하는 하네스별 참조 자료의 크기를 줄였습니다.

- **`using-superpowers` 부트스트랩 압축.** graphviz 스킬 흐름 다이어그램을 서술문 형태로 대체하고, 독립된 지침 우선순위(Instruction-Priority) 섹션을 사용자 지침(User Instructions)에 통합하였으며, 플랫폼별 "스킬 접근 방법" 안내를 축소하고, 플랫폼 적응(Platform Adaptation) 포인터를 참조 파일을 여전히 제공하는 하네스로 제한했습니다. 전체 Red Flags 합리화 테이블 및 사용자 지침 우선순위 규칙은 변경되지 않았습니다.
- **하네스별 도구 매핑 참조 다듬기.** 장황했던 동작-도구(action-to-tool) 테이블은 현대적인 에이전트가 이미 따르는 지침을 재확인하는 수준이었습니다. 각 참조 파일은 하네스 특화 참고사항(서브에이전트 디스패치, 작업 추적, 지침 파일 경로)만 남기고 정리되었으며, 하네스 특화 내용이 남아있지 않은 `claude-code-tools.md` 및 `copilot-tools.md`는 삭제되었습니다.

### Codex

- **Codex를 마켓플레이스에서 설치할 수 있습니다.** Codex 마켓플레이스 소스는 마켓플레이스 루트에 `.agents/plugins/marketplace.json`을 요구하지만 저장소에는 Claude 마켓플레이스 파일만 포함되어 있어 Codex가 마켓플레이스 이름을 지정할 수 있었으나 설치 가능한 플러그인 항목을 찾지 못했습니다. 이제 저장소 로컬 Codex 마켓플레이스 매니페스트가 동일한 저장소 루트를 가리키므로 Codex에서 플러그인을 설치할 수 있습니다.
- **Codex가 더 이상 SessionStart 훅을 제공하지 않습니다.** Codex는 자체적으로 스킬을 안정적으로 트리거하며, 부트스트랩 훅은 사용성을 높이기보다 저해했습니다. Codex 훅 설정(`hooks-codex.json`) 및 매니페스트 등록이 제거되었습니다.

### 하네스 지원 (Harness Support)

- **Gemini CLI 지원 제거.** Google은 2026-06-18에 Gemini CLI의 지원을 종료(EOL)했으며, 더 이상 확장 프로그램을 설치하거나 업데이트할 수 없습니다. Gemini는 설치 문서, 서브에이전트 지원 플랫폼 목록 및 평가 하네스 설명에서 제거되었으며 도구 매핑 참조가 삭제되었습니다.

## v6.0.3 (2026-06-18)

### 서브에이전트 기반 개발 (Subagent-Driven Development)

- **SDD 스크래치 파일이 `.git/` 외부로 이동되었습니다.** Claude Code는 `.git/`을 보호된 경로로 취급하여 에이전트 쓰기를 차단하므로, 구현자 서브에이전트가 `.git/sdd/`에 보고서를 작성하려 할 때 실행 도중 차단되었습니다. 이제 작업 개요(task briefs), 구현자 보고서, 리뷰 diff 및 진행 상황 대장은 작업 트리의 셀프 이그노어(self-ignoring) `.superpowers/sdd/` 디렉터리에 위치하며, `git status` 및 커밋 대상에서 제외되고 공유 `sdd-workspace` 헬퍼에 의해 작업 트리별로 해석됩니다. 주의사항: 작업 공간이 git-ignore 처리된 작업 트리 스크래치이므로 `git clean -fdx` 실행 시 진행 대장이 삭제됩니다. 이 경우 `git log`에서 복구하세요. (#1780)

## v6.0.2 (2026-06-16)

### 설치 수정 사항 (Install Fixes)

- **`evals` 서브모듈을 더 이상 제공하지 않습니다.** 일부 사용자에게 플러그인 설치 오류를 일으켰으므로, 평가 하네스는 배포된 플러그인과 분리되어 자체 저장소로 이동했습니다. (#1778, #1774)

## v6.0.1 (2026-06-16)

### Codex 수정 사항 (Codex Fixes)

- **브레인스토밍 컴패니언의 버전 표시** — 패키징된 Codex 플러그인은 루트 `package.json` 없이 배포되므로 시각적 컴패니언이 버전을 "unknown"으로 표시했습니다. `readSuperpowersVersion()`은 `package.json`이 없을 때 `.codex-plugin/plugin.json`을 폴백으로 사용하도록 수정되었습니다.
- **더 깔끔해진 Codex 플러그인 동기화** — sync-to-codex 스크립트가 `.gitmodules` 및 `.pre-commit-config.yaml`을 제외하여 패키징된 Codex 플러그인에 저장소 메타데이터가 들어가지 않도록 합니다.

## v6.0.0 (2026-06-16)

Superpowers 6.0은 대규모 릴리즈입니다. 핵심은 `subagent-driven-development`가 각 작업을 검토하는 방식을 재작성하여 비용을 줄이고 검증을 더욱 엄격하며 우회하기 어렵게 만든 것입니다.

이 수치가 모든 하네스와 워크로드에 동일하게 적용되지는 않겠지만, 당사의 평가 결과 Claude Code와 Codex는 약 2배 빠른 속도로 50% 적은 토큰을 소비하면서 동일한 고품질 결과를 생성했습니다.

또한 3개의 새로운 하네스(Kimi Code, Pi, Antigravity) 지원이 추가되었고, 브레인스토밍 시각 컴패니언의 보안 모델이 향상되었으며, 여러 스킬의 도구 호출이 벤더 중립적으로 재작성되었습니다.

### 주요 변경 사항 (Visible Changes)

- **작업별 리뷰어 프롬프트 2개가 하나로 통합되었습니다.** `spec-reviewer-prompt.md`와 `code-quality-reviewer-prompt.md`가 제거되고 단일 `task-reviewer-prompt.md`로 대체되었습니다. 이전 파일을 직접 디스패치하던 경우 새로운 파일로 전환하세요.
- **레거시 글로벌 작업 트리 디렉터리가 제거되었습니다.** `using-git-worktrees` 및 `finishing-a-development-branch`는 더 이상 `~/.config/superpowers/worktrees/`를 사용하지 않습니다. 별도로 지정하지 않는 한 작업 트리는 프로젝트 내부(기존 `.worktrees/` 또는 `worktrees/`, 없으면 새로 생성된 `.worktrees/`)에 생성됩니다.

### 새로운 하네스 지원 (New Harness Support)

Superpowers가 3개의 하네스를 추가로 지원합니다. 각 하네스는 자체 부트스트랩, 도구 매핑 참조, 테스트 및 README 설치 섹션을 포함합니다.

- **Kimi Code** — 플러그인 매니페스트, 설치 문서 및 매니페스트 테스트; Kimi 마켓플레이스 또는 저장소에서 직접 설치 가능합니다. (@qer 님의 최초 매니페스트)
- **Pi** — 스킬을 등록하고 `using-superpowers` 부트스트랩을 주입하는 session-start 확장 모듈. Pi는 네이티브 스킬을 제공하므로 호환성 심(shim)이 필요 없습니다.
- **Antigravity (`agy`)** — 플러그인을 직접 설치하고 첫 번째 메시지에서 부트스트랩을 수행합니다. 표준 "react todo list 만들기" 승인 테스트를 통해 엔드투엔드로 검증되었습니다.

### 서브에이전트 기반 개발 (Subagent-Driven Development)

실제 프로젝트에서 진행된 비용 및 품질 실험을 통해 컨트롤러가 각 작업을 검토하는 프로세스가 재구성되었습니다. 기존 흐름은 작업당 2명의 리뷰어를 실행하고 모델 선택 및 심각도에 대해 컨트롤러의 판단에 의존했으나, 이는 비용이 많이 들고 우회하기 쉬웠습니다. 새로운 흐름은 작업당 1명의 리뷰어를 실행하고 텍스트 붙여넣기 대신 파일로 전달하며, 컨트롤러로부터 여러 주관적 판단 권한을 회수했습니다.

- **작업당 1명의 리뷰어, 2개의 평가.** 단일 `task-reviewer-prompt.md`가 작업 diff를 한 번 읽고 사양 준수 평가와 품질 평가를 모두 반환하여 한 번의 수정 패스로 둘 다 해결합니다. 수정되지 않은 코드에 존재하는 요구사항을 컨트롤러가 직접 확인하도록 플래그를 지정하는 새로운 "diff에서 검증 불가" 평가가 추가되었습니다. (#1538, #1543)
- **마지막 단계의 단일 종합 리뷰.** 작업을 하나씩 재검토하는 대신 가장 성능이 뛰어난 모델에서 브랜치 전체에 대한 단일 종합 리뷰로 실행을 완료합니다.
- **계획에 대한 사전 검토(Pre-flight read).** 첫 번째 작업을 시작하기 전에 컨트롤러가 계획의 내부 충돌 및 리뷰어가 결함으로 지목할 만한 사항을 사전에 확인하여 실행 도중 차단되는 것을 방지합니다.
- **파일 기반 Diffs 및 작업 텍스트 이동.** 붙여넣은 diff는 비싼 컨텍스트를 영구 점유하고, diff가 없는 리뷰어는 손으로 재구성해야 하여 비용의 주요 원인이 되었습니다. 두 개의 새로운 스크립트 `task-brief`와 `review-package`가 서브에이전트가 읽을 작업 텍스트와 리뷰 diff를 파일에 작성합니다.
- **모든 디스패치에 모델 명시.** 선택권을 남겨두었을 때 컨트롤러가 모델 지정을 누락하고, 지정되지 않은 모델이 세션의 가장 비싼 모델을 상속받아 단일 실행의 26개 리뷰어 모두 최고 등급 모델을 사용하는 문제가 있었습니다. 템플릿은 작업이 허용할 때 더 저렴한 등급을 활용하도록 지침과 함께 모델 명시를 의무화합니다.
- **컨트롤러가 리뷰어에게 무시할 항목을 지시할 수 없음.** 실제 실행에서 컨트롤러가 발견 항목을 건너뛰거나 "최대 경미함"으로 평가하도록 리뷰어를 유도하는 결함이 관찰되었습니다. 발견 항목 억제 및 사전 등급 산정은 전면 금지되며, 계획 자체가 요구하는 결함이라도 묵인되지 않고 사용자가 결정하도록 보고됩니다.
- **리뷰어 읽기 전용화 및 정당화 사유 회의적 평가.** 리뷰가 더 이상 작업 트리나 브랜치를 수정하지 않으며(기존에 `git checkout`을 실행하던 리뷰어가 이후 커밋을 고립시키는 문제 방지), 구현자의 "의도적으로 추상화하지 않았다"는 주장이 실제 지적 사항을 무마하지 못합니다.
- **강력한 증거 및 보고.** 리뷰어는 각 답변을 파일 및 라인 번호로 뒷받침하고, 구현자의 보고서는 파일로 이동하여 TDD 적용 시 레드/그린 증거를 첨부하며, 진행 상황 대장을 통해 컨텍스트를 잃은 컨트롤러가 이미 완료된 작업을 재작업하지 않고 재개할 수 있습니다. (#994)

### 계획 작성 (Writing Plans)

이제 계획은 컨트롤러와 리뷰어가 디스패치 시마다 재도출해야 했던 구조를 미리 포함합니다.

- **전역 제약 조건(Global Constraints) 블록** — 버전 하한선, 의존성 제한, 명명 규칙, 정확한 값 등 모든 작업에 적용되는 규칙을 글자 그대로 복사하여 구현자와 리뷰어에게 정확히 전달되도록 합니다.
- **작업별 인터페이스(Interfaces) 블록** — 각 작업이 소비하고 생성하는 항목을 정확히 명시하여 해당 작업만 보는 구현자도 이웃 작업의 계약을 파악할 수 있도록 합니다.
- **적절한 크기 조정 지침** — 자체 테스트 주기와 리뷰어 승인을 받을 수 있는 적정한 크기로 작업을 유지하며 설정, 구성, 문서를 필요한 작업 내에 포함합니다. 테스트에서 이러한 방식으로 작성된 계획은 컨트롤 그룹이 2~4회의 수정 단계를 거치고 결함을 포함했던 것에 비해 단 1회의 수정만 필요했습니다.

### 브레인스토밍 시각 컴패니언 (Brainstorming Visual Companion)

시각 컴패니언은 에이전트가 대화와 함께 실행하는 소형 웹 서버입니다. 기존에는 인증이 전혀 없어 공유/원격 머신에서 포트에 접근할 수 있는 사람이면 누구나 브레인스토밍을 읽거나 사용자의 입력으로 처리되는 이벤트를 주입할 수 있었습니다. 이번 릴리즈에서는 올바른 보안 모델을 추가하고 재시작 및 연결 단절 시에도 유지되도록 개선되었습니다.

- **세션별 키 보안 적용.** 에이전트 URL에 일회성 키가 포함되며, 브라우저가 탭 범위를 가진 쿠키에 키를 저장하고 모든 요청 및 WebSocket 연결에서 키를 제시해야 합니다. 이를 통해 DNS 리바인딩 사례를 포함하여 로컬 탭 및 원격 호스트의 무단 접근을 차단합니다. (#1014 해결)
- **샌드박스 내부 파일 서버 지향.** 심볼릭 링크, 닷파일, 콘텐츠 디렉터리를 벗어나는 경로를 거부하며, macOS 리소스 포크 파일을 무시하고 standard no-store 및 deny-framing 헤더를 전송합니다. 세션 키를 보관하는 파일은 소유자 전용으로 작성됩니다.
- **도움이 되는 경우에만 컴패니언 제안.** 질문을 서술보다 시각적으로 보여주는 것이 좋을 때 별도 메시지로 제안하며 거절 시 수용합니다. 수락하면 브라우저에 첫 화면이 열립니다. (#755 해결)
- **재시작 및 불안정한 연결에 유연하게 대응.** 프로젝트 디렉터리가 주어지면 서버는 재시작 후에도 동일한 포트와 키를 유지하므로 열려 있는 탭이 자동으로 재연결됩니다. 페이지는 스스로 재연결을 시도하고 상태 알약을 표시하며 서버가 다운되었을 때 "일시 중지" 오버레이를 띄웁니다.
- **더 긴 유휴 시간 및 안전한 종료.** 유휴 타임아웃이 30분에서 4시간으로 늘어났으며, `stop-server.sh`가 신호를 보내기 전에 올바른 프로세스를 소유하고 있는지 확인하여 리부트 후 무관한 `node` 프로세스를 종료하지 않습니다. (#1703)
- **Windows 실행 강화** — 쉘 감지를 통합하였으며, Node가 MSYS2에서 POSIX 프로세스 소유권을 추적할 수 없으므로 Windows는 종료 시 유휴 타임아웃에 의존합니다.

### 기존 하네스 업데이트 (Existing Harness Updates)

- **Codex**는 공유 배선 대신 자체 SessionStart 훅을 통해 부트스트랩되며, Codex 앱에 설치 섹션과 확장된 도구 문서(웹 검색, `AGENTS.md`, 개인 스킬)가 추가되었습니다. (#1540)
- **OpenCode**는 플러그인, 설치 문서, README 전반에 걸쳐 동작 기반 도구 매핑이 적용되었고 부트스트랩 캐싱 테스트가 추가되었습니다.
- **Cursor** 매니페스트는 해당 디렉터리가 더 이상 존재하지 않으므로 `agents` 및 `commands` 항목을 제거했습니다.

### 모든 하네스를 위한 단일 스킬 세트 (One Set of Skills, Every Harness)

스킬 용어가 기존의 Claude Code 전용 어휘("Task 도구 사용", "CLAUDE.md에 배치")에서 실제 동작 중심의 어휘("서브에이전트 디스패치", "지침 파일")로 재작성되었으며, 동작을 올바른 도구로 매핑하는 하네스별 참조 파일이 추가되었습니다. "Claude"를 가리키던 서술문은 "당신의 에이전트"로 변경되었습니다.

- **하네스별 도구 참조**가 `skills/using-superpowers/references/`에 추가되어 Claude Code, Codex, Copilot, Gemini, Pi, Antigravity를 지원합니다.
- **`finishing-a-development-branch`가 포지(forge) 중립적으로 변경되었습니다.** — `gh pr create`를 하드코딩하지 않으므로 에이전트가 어떤 포지 툴링이든 사용할 수 있습니다. (#1609)
- **이름 변경:** "Claude Search Optimization" 기법이 Claude에 국한되지 않으므로 "Skill Discovery Optimization"으로 변경되었습니다.

### 스킬 작성 (Writing Skills)

스킬 작성자를 위한 두 가지 기능이 추가되었습니다.

- **실패 형태에 맞는 형태 맞추기 (Match the Form to the Failure)** — 적절한 지침 종류를 선택하기 위한 요약 표. 단호한 "X를 하지 마라" 지침은 규율 소홀에는 효과적이지만 출력의 *형태*가 문제일 때 부작용이 생길 수 있으며, 구체적인 작동 예시가 더 나은 결과를 냅니다.
- **마이크로 테스트 문구 작성 (Micro-Test Wording)** — 지침을 적용하기 전에 문구를 저렴하게 검증하는 방법: 지침이 없는 대조군과 함께 몇 번 샘플링하고 매 결과를 직접 읽어 실행 간 편차를 경고 신호로 처리합니다.

### 테스트 (Testing)

스킬 동작 테스트는 `tests/`에서 실제 Claude Code, Codex, Gemini 세션을 실행하고 LLM으로 평가하는 새로운 `evals/` 서브모듈("drill" 기반)로 이동했습니다. 엄격한 drill 시나리오가 커버하면서 기존 트리의 여러 bash 수트가 정리되었습니다. 앞으로 `tests/`는 플러그인 코드 테스트를 다루고 `evals/`는 스킬 동작 테스트를 다루며 `docs/testing.md`에 설명되어 있습니다. (#1541)

### 버그 수정 (Bug Fixes)

- **systematic-debugging이 더 이상 모든 세션에서 연장된 사고(extended thinking)를 강제하지 않습니다.** 문구 하나가 Claude Code가 스캔하는 키워드와 정확히 일치하여 스킬을 로드한 모든 세션의 스위치를 유발하던 현상이 하이픈 추가로 해결되었습니다. (#1283, @Nick Galatis 님 기여)
- **Windows SessionStart 훅이 세션마다 쓰기 오류를 출력하던 문제를 수정했습니다.** — 각 `printf`가 broken pipe를 흡수하도록 `cat`을 통과하도록 변경되었습니다. (#1612, @silvertakana 님 제보)
- **Windows 포그라운드 모드**가 올바른 프로세스를 추적하고 MSYS2에서 소유자 PID를 명확히 정리합니다. (@nestorluiscamachopaz 님 기여)
- **`using-superpowers` 부트스트랩**이 존재하지 않는 스킬로 "debugging"을 나열하지 않도록 수정되었습니다. (@mhat 님 제보)
- **TDD 스킬**이 테스트 안티패턴 참조 링크를 연결합니다. (#1532, #1529; 링크 수정 #1474 @Stable Genius 님 기여)
- **`using-git-worktrees`**의 단계 번호를 수정하고 오래된 Cursor 참조를 제거했습니다. (#1522, @fuleinist 님 기여)
- **Codex 리뷰 스킬**이 사적인 농담 대신 명확한 지침으로 대체되었습니다. (#1531)

### 문서 및 기여자 가이드라인 (Documentation & Contributor Guidelines)

- **Superpowers를 새로운 하네스로 포팅하기 위한 가이드**(`docs/porting-to-a-new-harness.md`)가 추가되어 세 가지 필수 요소를 제시합니다.
- **모든 PR 및 이슈에 작성 방식 공개 의무화** — 모델, 하네스, 버전 및 설치된 플러그인 정보, 또는 수작업 작성 여부를 밝혀야 합니다. PR 대상 브랜치도 `main`이 아닌 `dev`로 변경되었습니다.
- **기여자 목록:** @mattvanhorn, @nawfal, @Nick Galatis, @silvertakana, @nestorluiscamachopaz, @qer, @mhat, @Stable Genius, @fuleinist, @dev_Hakaze, @robotsnh, Rahul, @arittr 님께 감사드립니다.

## v5.1.0 (2026-04-30)

### 기능 제거 (Removals)

- **레거시 슬래시 명령어 제거** — `/brainstorm`, `/execute-plan`, `/write-plan`이 제거되었습니다. 해당 스킬을 호출하라고 안내만 하던 사용 중단 스텁이었습니다. 대신 `superpowers:brainstorming`, `superpowers:executing-plans`, `superpowers:writing-plans`를 직접 호출하세요. (#1188)
- **`superpowers:code-reviewer` 네임드 에이전트 제거** — 해당 에이전트는 플러그인의 유일한 네임드 에이전트로 단 2개의 스킬에서만 사용되었으나, 저장소의 다른 모든 리뷰어/구현자 서브에이전트는 프롬프트 템플릿과 함께 `general-purpose`를 디스패치합니다. 에이전트의 페르소나와 체크리스트는 `skills/requesting-code-review/code-reviewer.md`에 독립된 Task 디스패치 템플릿으로 통합되었습니다. `Task (superpowers:code-reviewer)`를 디스패치하던 경우 프롬프트 템플릿과 함께 `Task (general-purpose)`로 전환하세요. (PR #1299)
- **스킬에서 통합 섹션 제거** — 에이전트에 네이티브 스킬 시스템이 없던 시절의 레거시로 지침 제어에 도움이 되지 않았습니다.

### 작업 트리 스킬 재작성 (Worktree Skills Rewrite)

`using-git-worktrees` 및 `finishing-a-development-branch`는 에이전트가 이미 격리된 작업 트리 내부에서 실행 중인지 감지하고 `git worktree`로 폴백하기 전에 하네스의 네이티브 작업 트리 제어를 우선 사용하도록 개선되었습니다. 5개 하네스에서 TDD 검증 및 교차 플랫폼 검사를 완료했습니다. (PRI-974, PR #1121)

- **환경 감지** — 두 스킬 모두 작업 전 `GIT_DIR != GIT_COMMON`을 확인하여 이미 연결된 작업 트리에 있는 경우 생성을 건너뜁니다.
- **작업 트리 생성 전 동의 요청** — `using-git-worktrees`가 더 이상 암묵적으로 작업 트리를 생성하지 않으며 사용자에게 먼저 확인합니다. (#991 해결)
- **네이티브 도구 우선선택 (1a 단계)** — 하네스가 자체 작업 트리 도구를 제공할 때 스킬이 이를 우선시합니다.
- **출처 기반 정리** — `finishing-a-development-branch`는 `.worktrees/` 내부의 작업 트리(superpowers가 생성함)만 정리하고 외부 항목은 유지합니다. (#940, #999, #238 해결)
- **분리된 HEAD (Detached HEAD) 처리** — 병합할 브랜치가 없는 경우 정리 메뉴가 2개 옵션으로 축소됩니다.
- **하드코딩된 `/Users/jesse` 경로**가 범용 자리표시자로 대체되었습니다. (#858, PR #1122)

### AI 에이전트를 위한 기여자 가이드라인 (Contributor Guidelines for AI Agents)

`CLAUDE.md` (동일한 `AGENTS.md` 지향) 상단에 AI 에이전트를 직접 대상으로 하는 2개의 새로운 섹션이 추가되었습니다. 이 저장소에 대해 닫힌 최근 100개 PR을 감사한 결과, AI가 생성한 저품질 작업(PR 템플릿 미준수, 중복 생성, 허위 문제 작성, 포크/도메인 특화 변경 사항의 업스트림 푸시 등)으로 인해 94%의 거절률을 보였습니다.

- **제출 전 체크리스트** — PR 템플릿 읽기, 기존 PR 검색, 실제 문제 존재 여부 확인, 코어 포함 여부 확인, 제출 전 인간 파트너에게 전체 diff 제시.
- **수용 불가 항목** — 서드파티 의존성, 스킬 내용의 "준수" 재작성, 프로젝트 특화 설정, 대량 PR, 추측성 수정, 도메인 특화 스킬, 포크 특화 변경, 날조된 콘텐츠, 관련 없는 변경 사항 번들링.
- **새로운 하네스 PR 시 세션 트랜스크립트 제출 의무화** — 인수 테스트("react todo list 만들기" 실행 시 클린 세션에서 `brainstorming` 자동 트리거 필요) 및 전체 트랜스크립트 제출이 요구됩니다.

### Codex 플러그인 미러 툴링 (Codex Plugin Mirror Tooling)

새로운 `sync-to-codex-plugin` 스크립트가 superpowers를 OpenAI Codex 플러그인 마켓플레이스(`prime-radiant-inc/openai-codex-plugins`)로 미러링합니다. 경로 및 사용자에 독립적이므로 팀원 누구나 실행할 수 있습니다. (PR #1165)

### OpenCode

- **모듈 레벨에서 부트스트랩 콘텐츠 캐싱** — `getBootstrapContent()`가 매 에이전트 단계마다 호출되던 문제를 수정하여 세션 수명 동안 1회 읽고 캐싱합니다. (#1202 해결)
- **통합 테스트 현대화**.
- **README의 설치 관련 주의사항 명확화**.

### 코드 리뷰 통합 (Code Review Consolidation)

`requesting-code-review`가 이제 독립적으로 작동합니다. 페르소나, 체크리스트 및 디스패치 템플릿이 `skills/requesting-code-review/code-reviewer.md`에 위치하며 스킬이 `Task (general-purpose)`를 직접 디스패치합니다. (PR #1299)

### 서브에이전트 기반 개발 (Subagent-Driven Development)

- **3개 작업마다 정지하는 배치 리뷰 제거** — 연속 실행 지침으로 대체되었습니다.
- **SDD 통합 테스트의 검증 실행 수정** — 부작용을 일으키던 3가지 독립 버그를 수정하여 6개의 검증 테스트가 실제 엔드투엔드 SDD 실행에 대해 동작합니다.

### Cursor

- **Windows SessionStart 훅**이 확장자 없는 `session-start` 스크립트를 직접 호출하는 대신 `run-hook.cmd`를 통해 라우팅됩니다. `hooks-cursor.json`에서 우발적인 UTF-8 BOM도 제거했습니다.

### Gemini CLI

- **서브에이전트 디스패치 매핑** — Gemini의 `Task` 디스패치가 `@agent-name` / `@generalist`로 매핑되며 병렬 서브에이전트 디스패치가 문서화되었습니다.

### 스킬 (Skills)

- 스킬 전반에 걸친 용어 정돈.

### 문서 및 설치 (Documentation & Install)

- **Factory Droid 설치 지침**이 README에 추가되었습니다.
- **Quickstart 설치 링크** 추가 (PR #1293 by @arittr).
- **Codex 플러그인 설치 안내** 업데이트 (PR #1288 by @arittr).
- **Codex `wait` 매핑**이 도구 참조에서 `wait_agent`로 수정되었습니다.
- **설치 순서 재정비** 및 Codex 설치 지침 정리.
- 단일 출처로서 `RELEASE-NOTES.md`를 위해 **잔재 `CHANGELOG.md` 제거** (PR #1163 by @shaanmajid).
- **Discord 초대 링크** 수정 및 커뮤니티 섹션 내용 추가.

### 커뮤니티 (Community)

- @shaanmajid — 잔재 `CHANGELOG.md` 제거 (PR #1163)
- @arittr — README 퀵스타트 설치 링크 (#1293), Codex 플러그인 설치 안내 (#1288), `sync-to-codex-plugin` `interface.defaultPrompt` 시드 (#1180)

---

## v5.0.7 (2026-03-31)

### GitHub Copilot CLI 지원

- **SessionStart 컨텍스트 주입** — Copilot CLI v1.0.11에서 sessionStart 훅 출력의 `additionalContext` 지원이 추가되었습니다. 세션 시작 훅이 `COPILOT_CLI` 환경 변수를 감지하고 SDK 표준 `{ "additionalContext": "..." }` 형식을 출력하여 Copilot CLI 사용자에게 superpowers 부트스트랩을 제공합니다. (@culinablaz 님 PR #910)
- **도구 매핑** — Claude Code와 Copilot CLI 도구 간 동등성 테이블이 포함된 `references/copilot-tools.md` 추가.
- **스킬 및 README 업데이트** — Copilot CLI 항목 추가.

### OpenCode 수정 사항

- **스킬 경로 일관성 유지** — 부트스트랩 텍스트가 잘못된 경로를 안내하지 않도록 단일 출처에서 파생된 경로를 사용합니다. (#847, #916)
- **사용자 메시지로 부트스트랩 주입** — 첫 번째 사용자 메시지 앞에 주입하여 토큰 팽창을 방지하고 모델 호환성을 해결합니다. (#750, #894)

## v5.0.6 (2026-03-24)

### 인라인 자체 리뷰가 서브에이전트 리뷰 루프를 대체

서브에이전트 리뷰 루프(새로운 에이전트를 디스패치하여 계획/사양 검토)는 품질 향상 없이 실행 시간을 2배(~25분 오버헤드)로 늘렸습니다.

- **brainstorming** — 사양 리뷰 루프를 인라인 사양 자체 리뷰 체크리스트(자리표시자 스캔, 내부 일관성, 범위 검사, 모호성 검사)로 대체했습니다.
- **writing-plans** — 계획 리뷰 루프를 인라인 자체 리뷰 체크리스트(사양 커버리지, 자리표시자 스캔, 타입 일관성)로 대체했습니다.
- **writing-plans** — 계획 실패 요인을 정의하는 명시적인 "자리표시자 금지(No Placeholders)" 섹션을 추가했습니다.
- 자체 리뷰는 서브에이전트 방식과 유사한 결함 발견율을 유지하면서 ~25분 대신 ~30초 만에 실행당 3~5개의 실제 버그를 찾아냅니다.

### 브레인스토밍 서버 (Brainstorm Server)

- **세션 디렉터리 구조 재편** — 브레인스토밍 서버 세션 디렉터리가 이제 2개의 동등한 하위 디렉터리 `content/`(브라우저에 제공되는 HTML 파일)와 `state/`(이벤트, 서버 정보, pid, 로그)로 나누어져 보안성이 강화되었습니다. (吉田仁 님 제보)

### 버그 수정 (Bug Fixes)

- **소유자 PID 수명주기 수정** — EPERM 오류를 "살아있음"으로 처리하고 시작 시 소유자 PID를 검증하여 60초 후 오작동 종료되던 버그를 해결했습니다. (#879)
- **writing-skills** — SKILL.md 프론트매터 필드 관련 설명을 수정하고 agentskills.io 명세를 링크했습니다. (PR #882 by @arittr)

### Codex App 호환성

- **codex-tools** — Claude Code의 네임드 에이전트 유형을 Codex의 작업자 역할을 가진 `spawn_agent`로 변환하는 방법을 문서화했습니다. (PR #647 by @arittr)
- **codex-tools** — 작업 트리 인식 스킬을 위한 환경 감지 및 Codex App 마감 섹션 추가. (by @arittr)
- **디자인 명세** — 읽기 전용 환경 감지, 작업 트리 안전 스킬 동작 등을 다루는 Codex App 호환성 디자인 명세(PRI-823) 추가. (by @arittr)

## v5.0.5 (2026-03-17)

### 버그 수정

- **브레인스토밍 서버 ESM 수정** — `server.js` → `server.cjs`로 변경하여 Node.js 22+ 환경에서 구동 오작동을 해결했습니다. (PR #784 by @sarbojitrana, #774, #780, #783 해결)
- **Windows 환경 브레인스토밍 소유자 PID** — PID 네임스페이스 모니터링을 건너뛰어 서버 자가 종료를 방지합니다. (#770, PR #768 문서)
- **stop-server.sh 신뢰성 향상** — 프로세스가 실제로 종료되었는지 확인하도록 검증 로직 강화. (#723)

### 변경 사항

- **실행 위임 선택권 복원** — 계획 작성 후 서브에이전트 기반 실행과 인라인 실행 간 선택권을 사용자에게 다시 제공합니다.

## v5.0.4 (2026-03-16)

### 리뷰 루프 개선 (Review Loop Refinements)

불필요한 리뷰 단계를 없애고 초점을 집중시켜 토큰 사용량을 대폭 줄이고 사양/계획 리뷰 속도를 향상했습니다.

- **단일 전체 계획 리뷰** — 계획 리뷰어가 청크별 검토 대신 한 번의 패스로 전체 계획을 리뷰합니다.
- **차단 이슈 기준 상향** — 구현 중 실제 문제를 일으킬 수 있는 이슈만 플래그를 지정하도록 조정했습니다.
- **최대 리뷰 반복 횟수 축소** — 사양 및 계획 리뷰 루프 모두 5회에서 3회로 축소했습니다.
- **리뷰어 체크리스트 간소화** — 사양 리뷰어는 7개 카테고리에서 5개로, 계획 리뷰어는 7개에서 4개로 정리했습니다.

### OpenCode

- **한 줄 플러그인 설치** — `opencode.json`에 한 줄 추가로 설치가 완료됩니다. (PR #753)
- **`package.json` 추가** — OpenCode가 git에서 npm 패키지로 superpowers를 설치할 수 있습니다.

### 버그 수정

- **서버 실제 정지 검증** — `stop-server.sh` 프로세스 정지 확인 로직 강화. (PR #751)
- **범용 에이전트 용어 적용** — 대기 페이지에서 "Claude" 대신 "에이전트"로 서술합니다.

## v5.0.3 (2026-03-15)

### Cursor 지원

- **Cursor 훅** — Cursor의 camelCase 형식(`sessionStart`, `version: 1`)이 적용된 `hooks/hooks-cursor.json` 추가 및 플러그인 매니페스트 참조 업데이트. (PR #709 기반)

### 버그 수정

- **`--resume` 시 SessionStart 훅 트리거 방지** — 대화 이력이 이미 존재하는 재개 세션에서 컨텍스트 재주입을 방지합니다.
- **Bash 5.3+ 훅 멈춤 현상 수정** — heredoc을 `printf`로 대체하여 macOS 멈춤 현상을 해결했습니다. (#572, #571)
- **POSIX 안전 훅 스크립트** — `${BASH_SOURCE[0]:-$0}`를 `$0`으로 대체. (#553)
- **이식성 높은 쉬뱅(Shebang)** — `#!/usr/bin/env bash` 적용. (#700)
- **Windows 브레인스토밍 서버** — Windows/Git Bash 자동 감지 및 포그라운드 모드 전환. (#737)
- **Codex 문서 수정** — 구식 `collab` 플래그를 `multi_agent`로 수정. (PR #749)

## v5.0.2 (2026-03-11)

### 의존성 없는 브레인스토밍 서버 (Zero-Dependency Brainstorm Server)

- Express/Chokidar/WebSocket 의존성을 제거하고 Node.js 내장 `http`, `fs`, `crypto` 모듈을 사용하는 완전 자립형 서버로 전환.
- 30분 유휴 후 자동 종료 및 소유자 프로세스 추적 기능 추가.

### 서브에이전트 컨텍스트 격리 (Subagent Context Isolation)

- 모든 위임 스킬에 컨텍스트 격리 원칙을 포함하여 컨텍스트 윈도우 오염 방지.

## v5.0.1 (2026-03-10)

### Agentskills 준수 (Agentskills Compliance)

- `lib/brainstorm-server/` → `skills/brainstorming/scripts/`로 이동.
- Gemini CLI 확장 기능 지원 및 설치 안내 추가.
- 다중 플랫폼 브레인스토밍 서버 실행 지침 및 의존성 번들링.
- 쿼팅 문제로 인한 Windows/Linux 훅 실행 실패 해결. (#577, #529, #644, PR #585)

## v5.0.0 (2026-03-09)

### 주요 변경 사항 (Breaking Changes)

- 사양(specs)은 `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`, 계획(plans)은 `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`에 저장됩니다.
- 서브에이전트 지원 하네스에서 서브에이전트 기반 개발(subagent-driven-development)이 의무화되었습니다.
- 실행 계획(executing-plans)에서 3개 작업 배치 정지 패턴이 제거되어 연속 실행합니다.
- 슬래시 명령어가 단종 중단 예정(deprecated) 안내를 표시합니다.

### 신규 기능 및 개선 사항

- 브라우저 기반 시각적 브레인스토밍 컴패니언 도입.
- 사양 및 계획 문서 검증을 위한 문서 리뷰 시스템(Document review system) 도입.
- 스킬 파이프라인 전반에 아키텍처 지침(격리 설계, 파일 크기 인지) 반영.

---

## v4.3.1 (2026-02-21)

- Cursor 플러그인 시스템 지원 추가 (`.cursor-plugin/plugin.json`).
- Windows 환경 훅 실행 신뢰성을 높이기 위해 다국어 래퍼(polyglot wrapper) 복원 및 확장자 없는 파일명 적용. (#518, #504 등)

## v4.3.0 (2026-02-12)

- 브레인스토밍 스킬에 하드 게이트(`<HARD-GATE>`), 필수 체크리스트 및 그래프비즈 프로세스 흐름을 도입하여 워크플로우 강제.
- `EnterPlanMode` 차단을 추가하여 Claude 네이티브 플랜 모드 진입 방지.
- SessionStart 훅 동기 실행(`async: false`) 적용.

## v4.2.0 (2026-02-05)

- Codex: 부트스트랩 CLI를 네이티브 스킬 탐색(`~/.agents/skills/superpowers/`)으로 대체.
- Windows 환경 Claude Code 2.1.x 훅 실행 및 O(n^2) 문자열 처리 성능 문제 수정.
- 구현 전 작업 트리 격리(`using-git-worktrees`) 의무화.

## v4.1.1 (2026-01-23)

- OpenCode: 공식 문서에 맞춰 `plugins/` 디렉터리로 표준화 및 심볼릭 링크 지침 수정.

## v4.1.0 (2026-01-23)

- OpenCode: 네이티브 `skill` 도구 시스템으로 전환.

## v4.0.3 (2025-12-26)

- 명시적 스킬 요청 시 에이전트가 우회하지 않고 해당 스킬을 반드시 호출하도록 규칙 강화.

## v4.0.2 (2025-12-23)

- 슬래시 명령어에 `disable-model-invocation: true` 설정 추가로 모델 자동 호출 차단.

## v4.0.1 (2025-12-23)

- Claude Code에서 Read 도구 대신 Skill 도구로 스킬 콘텐츠를 직접 로드하도록 설명 명확화.

## v4.0.0 (2025-12-17)

- `subagent-driven-development`에 2단계 코드 리뷰(사양 준수 리뷰 -> 코드 품질 리뷰) 도입.
- 디버깅 기법 통합(`systematic-debugging`), 테스트 안티패턴 참조 자료 포함.
- DOT 순서도를 프로세스 정의의 권위적 명세로 채택.
- 스킬 명칭 표준화 및 소문자 케밥 케이스(kebab-case)로 통일.

---

## v3.6.2 ~ v2.0.0 주요 내역

- Linux 호환성 및 OpenCode 부트스트랩 개선.
- Codex 실험적 지원 및 통합 모듈 구현.
- 스킬 리포지토리 분리 (`obra/superpowers-skills`) 및 세션 시작 시 자동 업데이트 체계 도입.
