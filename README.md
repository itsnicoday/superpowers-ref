# Superpowers

Superpowers는 개발 에이전트를 위한 완전한 소프트웨어 개발 방법론으로, 조합 가능한 스킬 세트와 에이전트가 이를 확실히 사용하도록 안내하는 초기 지시사항을 기반으로 구축되었습니다.

## 채용 안내

Superpowers 커뮤니티 및 코드 작업을 상근으로 도와줄 인재를 채용 중입니다.  
직무에 대한 자세한 내용은 https://primeradiant.com/jobs/superpowers-community-engineer/ 에서 확인하실 수 있습니다.  
주변에 적합한 분이 계시다면 꼭 추천해 주시기 바랍니다.

## 빠른 시작 (Quickstart)

개발 에이전트에 Superpowers를 부여하세요: [Claude Code](#claude-code), [Antigravity](#antigravity), [Codex App](#codex-app), [Codex CLI](#codex-cli), [Cursor](#cursor), [Factory Droid](#factory-droid), [GitHub Copilot CLI](#github-copilot-cli), [Kimi Code](#kimi-code), [OpenCode](#opencode), [Pi](#pi).

## 작동 방식

개발 에이전트를 실행하는 순간부터 작동이 시작됩니다. 사용자가 무언가를 구축하려 한다는 것을 감지하면, 에이전트는 무작정 코드를 작성하려 들지 *않습니다*. 대신 한 걸음 물러서서 실제로 무엇을 하려는 것인지 질문합니다.

대화를 통해 스펙을 도출하고 나면, 실제로 읽고 이해할 수 있을 만큼 짧은 청크 단위로 사용자에게 이를 보여줍니다.

설계에 대해 사용자가 승인하고 나면, 에이전트는 안목이 부족하고, 판단력이 없으며, 프로젝트 컨벤션을 잘 모르고, 테스트를 기피하는 열정적인 주니어 엔지니어도 쉽게 따를 수 있을 만큼 명확한 구현 플랜을 작성합니다. 이 플랜은 진정한 RED/GREEN TDD, YAGNI(You Aren't Gonna Need It), 그리고 DRY 원칙을 강조합니다.

다음으로 사용자가 "시작(go)"을 명령하면, *서브에이전트 주도 개발(subagent-driven-development)* 프로세스를 출범시켜 에이전트들이 각 엔지니어링 작업을 검토 및 리뷰하면서 계속 전진해 나갑니다. 에이전트가 작성한 플랜에서 벗어나지 않고 한 번에 몇 시간씩 자율적으로 작업하는 경우도 흔합니다.

이 외에도 많은 내용이 포함되어 있지만, 이것이 시스템의 핵심입니다. 스킬이 자동으로 트리거되므로 사용자가 별도로 특별한 조치를 취할 필요가 없습니다. 개발 에이전트가 자연스럽게 Superpowers를 갖추게 됩니다.

## 상업용 서비스

기업에서 Superpowers를 사용하고 계시며 상업용 지원, 추가 툴링, 또는 비용 관리의 혜택을 원하신다면 언제든지 sales@primeradiant.com으로 문의해 주시기 바랍니다.

## 설치

설치는 하네스(harness)에 따라 다릅니다. 하나 이상의 하네스를 사용하는 경우 각각에 대해 Superpowers를 별도로 설치하십시오.

### Claude Code

Superpowers는 [공식 Claude 플러그인 마켓플레이스](https://claude.com/plugins/superpowers)를 통해 이용할 수 있습니다.

#### 공식 마켓플레이스

- Anthropic 공식 마켓플레이스에서 플러그인을 설치합니다:

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 마켓플레이스

Superpowers 마켓플레이스는 Claude Code를 위한 Superpowers 및 기타 관련 플러그인을 제공합니다.

- 마켓플레이스 등록:

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 이 마켓플레이스에서 플러그인 설치:

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

이 저장소에서 플러그인으로 Superpowers를 설치합니다:

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity는 플러그인의 세션 시작 훅을 실행하므로 Superpowers가 첫 메시지부터 활성화됩니다. 동일한 명령어로 재설치하여 업데이트할 수 있습니다.

### Codex App

Superpowers는 [공식 Codex 플러그인 마켓플레이스](https://github.com/openai/plugins)를 통해 이용할 수 있습니다.

- Codex 앱의 사이드바에서 Plugins를 클릭합니다.
- Coding 섹션에서 `Superpowers`를 볼 수 있습니다.
- Superpowers 옆의 `+`를 클릭하고 안내를 따릅니다.

### Codex CLI

Superpowers는 [공식 Codex 플러그인 마켓플레이스](https://github.com/openai/plugins)를 통해 이용할 수 있습니다.

- 플러그인 검색 인터페이스를 엽니다:

  ```bash
  /plugins
  ```

- Superpowers 검색:

  ```bash
  superpowers
  ```

- `Install Plugin`을 선택합니다.

### Cursor

- Cursor Agent 채팅에서 마켓플레이스를 통해 설치:

  ```text
  /add-plugin superpowers
  ```

- 또는 플러그인 마켓플레이스에서 "superpowers"를 검색합니다.

### Factory Droid

- 마켓플레이스 등록:

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 플러그인 설치:

  ```bash
  droid plugin install superpowers@superpowers
  ```

### GitHub Copilot CLI

- 마켓플레이스 등록:

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 플러그인 설치:

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Kimi Code

Superpowers는 Kimi Code의 플러그인 마켓플레이스에서 이용할 수 있습니다.

- Kimi Code의 플러그인 관리자를 엽니다:

  ```text
  /plugins
  ```

- `Marketplace` > `Superpowers`로 이동하여 설치합니다.

- 또는 이 저장소에서 직접 설치합니다:

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 상세 문서: [docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode는 자체 플러그인 설치 방식을 사용합니다; 이미 다른 하네스에서 사용 중이더라도 Superpowers를 별도로 설치하십시오.

- OpenCode에 전달:

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 상세 문서: [docs/README.opencode.md](docs/README.opencode.md)

### Pi

이 저장소에서 Pi 패키지로 Superpowers를 설치합니다:

```bash
pi install git:github.com/obra/superpowers
```

로컬 개발을 위해, 이 체크아웃을 임시 패키지로 로드하여 Pi를 실행합니다:

```bash
pi -e /path/to/superpowers
```

Pi 패키지는 Superpowers 스킬과 세션 시작 시 및 압축 후 `using-superpowers` 부트스트랩을 주입하는 소형 확장을 로드합니다. Pi에는 네이티브 스킬이 있으므로 호환성을 위한 `Skill` 도구가 필요하지 않습니다. 서브에이전트 및 작업 목록 도구는 선택적 Pi 동반 패키지로 남아 있습니다.

## 기본 워크플로우

1. **brainstorming** - 코드를 작성하기 전에 활성화됩니다. 질문을 통해 대략적인 아이디어를 구체화하고, 대안을 탐색하며, 검증을 위해 설계를 섹션별로 제시합니다. 설계 문서를 저장합니다.

2. **using-git-worktrees** - 설계 승인 후 활성화됩니다. 새 브랜치에 격리된 작업 공간을 생성하고, 프로젝트 설정을 실행하며, 깨끗한 테스트 베이스라인을 검증합니다.

3. **writing-plans** - 승인된 설계와 함께 활성화됩니다. 작업을 한 입 크기(각 2-5분)로 나눕니다. 모든 작업에는 정확한 파일 경로, 완전한 코드, 검증 단계가 포함됩니다.

4. **subagent-driven-development** 또는 **executing-plans** - 플랜과 함께 활성화됩니다. 2단계 리뷰(스펙 준수, 그 후 코드 품질)를 통해 작업당 신규 서브에이전트를 디스패치하거나, 인간 체크포인트와 함께 배치로 실행합니다.

5. **test-driven-development** - 구현 중에 활성화됩니다. RED-GREEN-REFACTOR를 강제합니다: 실패하는 테스트 작성, 실패 확인, 최소한의 코드 작성, 통과 확인, 커밋. 테스트 작성 전에 작성된 코드는 삭제합니다.

6. **requesting-code-review** - 작업 중간에 활성화됩니다. 플랜에 대비해 검토하고 심각도별로 이슈를 보고합니다. Critical 이슈는 진행을 블로킹합니다.

7. **finishing-a-development-branch** - 작업 완료 시 활성화됩니다. 테스트를 검증하고, 옵션(merge/PR/keep/discard)을 제시하며, 워크트리를 정리합니다.

**에이전트는 어떤 작업을 진행하기 전에 관련 스킬을 검사합니다.** 단순 권장 사항이 아닌 필수 워크플로우입니다.

## 내부 구성 요소

### 스킬 라이브러리

**테스팅 (Testing)**
- **test-driven-development** - RED-GREEN-REFACTOR 사이클 (테스팅 안티 패턴 참조 포함)

**디버깅 (Debugging)**
- **systematic-debugging** - 4단계 근본 원인 프로세스 (근본 원인 추적, 심층 방어, 조건 기반 대기 기법 포함)
- **verification-before-completion** - 실제로 수정되었는지 확인

**협업 (Collaboration)** 
- **brainstorming** - 소크라테스식 설계 구체화
- **writing-plans** - 상세 구현 플랜
- **executing-plans** - 체크포인트를 포함한 배치 실행
- **dispatching-parallel-agents** - 동시 서브에이전트 워크플로우
- **requesting-code-review** - 사전 리뷰 체크리스트
- **receiving-code-review** - 피드백에 응답하기
- **using-git-worktrees** - 병렬 개발 브랜치
- **finishing-a-development-branch** - 머지/PR 결정 워크플로우
- **subagent-driven-development** - 2단계 리뷰(스펙 준수, 그 후 코드 품질)를 통한 빠른 반복

**메타 (Meta)**
- **writing-skills** - 모범 사례를 따라 새로운 스킬 생성 (테스팅 방법론 포함)
- **using-superpowers** - 스킬 시스템 소개

## 철학 (Philosophy)

- **테스트 주도 개발 (Test-Driven Development)** - 항상 테스트를 먼저 작성
- **임시방편보다 체계적으로 (Systematic over ad-hoc)** - 임의의 추측보다 체계적 프로세스
- **복잡성 감소 (Complexity reduction)** - 단순성을 최우선 목표로 설정
- **주장보다 증거 (Evidence over claims)** - 성공을 선언하기 전에 검증

[최초 릴리스 발표](https://blog.fsck.com/2025/10/09/superpowers/)를 읽어보세요.

## 기여하기 (Contributing)

Superpowers의 일반적인 기여 프로세스는 아래와 같습니다. 새로운 스킬 기여는 일반적으로 수락하지 않으며, 스킬 업데이트는 지원하는 모든 개발 에이전트 전반에서 작동해야 한다는 점을 유념해 주십시오.

1. 저장소 포크(Fork)
2. 'dev' 브랜치로 전환
3. 작업을 위한 브랜치 생성
4. 신규 및 수정된 스킬을 생성하고 테스트할 때 `writing-skills` 스킬을 준수
5. 풀 리퀘스트 템플릿을 반드시 작성하여 PR 제출

스킬 동작 테스트는 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/)에서 가져와 `evals/`에 클론된 drill 이발 하네스를 사용합니다 — 설정을 보려면 `evals/README.md`를 참조하십시오. 플러그인 인프라 테스트는 `tests/`에 위치하며 관련 `run-*.sh` 또는 `npm test`를 통해 실행됩니다.

전체 가이드는 `skills/writing-skills/SKILL.md`를 참조하십시오.

## 업데이트

Superpowers 업데이트는 개발 에이전트에 따라 다소 차이가 있지만, 종종 자동으로 수행됩니다.

## 라이선스

MIT 라이선스 - 자세한 내용은 LICENSE 파일을 참조하십시오.

## Visual companion 텔레메트리 (Telemetry)

스킬과 플러그인은 제작자에게 피드백을 제공하지 않기 때문에 얼마나 많은 분들이 Superpowers를 사용하는지 알 수 없습니다. 기본적으로 브레인스토밍의 선택적 visual companion 기능에 포함된 Prime Radiant 로고는 당사 웹사이트에서 로드됩니다. 여기에는 사용 중인 Superpowers 버전이 포함됩니다. 프로젝트, 프롬프트 또는 개발 에이전트에 대한 어떠한 세부 정보도 포함되지 않습니다. 당사는 사용자의 클릭이나 구축 내용에 대한 어떠한 정보도 보지 않습니다. 이는 얼마나 많은 분들이 Superpowers를 사용하는지, 어떤 버전을 사용하는지 대략 파악하는 데 도움이 됩니다. 이는 100% 선택 사항입니다. 이를 비활성화하려면 환경 변수 `SUPERPOWERS_DISABLE_TELEMETRY`를 참에 해당하는 값으로 설정하십시오. Superpowers는 Claude Code의 `DISABLE_TELEMETRY` 및 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 비활성화 설정도 존중합니다.

## 커뮤니티

Superpowers는 [Jesse Vincent](https://blog.fsck.com)와 [Prime Radiant](https://primeradiant.com)의 다른 팀원들에 의해 개발되었습니다.

- **Discord**: [참여하기](https://discord.gg/35wsABTejz) 커뮤니티 지원, 질문 및 Superpowers로 구축한 작업 공유
- **Issues**: https://github.com/obra/superpowers/issues
- **릴리스 공지**: [등록하기](https://primeradiant.com/superpowers/) 새 버전에 대한 알림 받기
