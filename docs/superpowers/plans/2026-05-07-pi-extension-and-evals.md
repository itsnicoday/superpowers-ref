# Pi 확장 및 평가(Evals) 구현 계획

> **에이전트 작업자 참고:** 필수 서브 스킬: 이 계획을 작업별로 구현하려면 superpowers:subagent-driven-development (권장) 또는 superpowers:executing-plans를 사용하세요. 추적을 위해 단계에 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** Superpowers에 대한 최고 수준(first-class)의 Pi 패키지 지원을 추가하고 Pi를 Drill 평가 백엔드로 추가합니다.

**아키텍처:** Pi 패키지는 루트 `package.json`에 선언되며 기존 `skills/` 및 소규모 Pi 확장을 로드합니다. 확장은 세션 시작 시 및 압축(compaction) 후에 Pi 전용 도구 매핑과 함께 `using-superpowers` 부트스트랩을 사용자 역할 메시지로 제공자 컨텍스트에 주입합니다. Drill은 `pi` 백엔드, Pi 세션 로그 정규화, 그리고 테스트를 갖추게 됩니다.

**기술 스택:** Pi TypeScript 확장 API, Node 내장 테스트 러너, Drill Python 평가 하네스, pytest.

---

### 작업 1: Pi 패키지 매니페스트 및 확장 테스트

**파일:**
- 수정: `package.json`
- 생성: `tests/pi/test-pi-extension.mjs`

- [ ] **단계 1: 실패하는 패키지/확장 테스트 작성**

`extensions/superpowers.ts`를 임포트하고, 가짜 Pi 핸들러를 등록하며, 다음을 단정하는 테스트를 포함하여 `tests/pi/test-pi-extension.mjs`를 생성합니다:
- 루트 `package.json`의 `keywords`에 `pi-package`가 포함됨
- 루트 `package.json`에 `pi.skills: ["./skills"]`가 포함됨
- 루트 `package.json`에 `pi.extensions: ["./extensions/superpowers.ts"]`가 포함됨
- 확장이 `resources_discover`, `session_start`, `session_compact`, `context`, `agent_end`를 등록함
- 시작 시 `context`가 정확히 하나의 사용자 역할 부트스트랩 메시지를 주입함
- `agent_end`가 시작 주입을 초기화함
- `session_compact`가 주입을 다시 활성화함
- 확장이 `session_before_compact`를 등록하지 않음

- [ ] **단계 2: 테스트를 실행하고 RED 검증**

실행: `node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

예상 결과: 실패 — `extensions/superpowers.ts`가 존재하지 않고 `package.json`에 `pi` 매니페스트가 없기 때문.

- [ ] **단계 3: 매니페스트 필드 구현**

기존 `name`, `version`, `type`, `main`을 유지하면서 `description`, `keywords`, `pi.extensions`, `pi.skills`로 `package.json`을 업데이트합니다.

- [ ] **단계 4: `extensions/superpowers.ts` 구현**

다음 기능을 수행하는 런타임 의존성 없는 확장을 생성합니다:
- `import.meta.url`에서 패키지 루트 위치를 찾음
- `skills/using-superpowers/SKILL.md`를 읽음
- YAML 프론트매터를 제거함
- Pi 전용 도구 매핑을 추가함
- 스킬 경로가 포함된 `resources_discover`를 노출함
- `session_start` 및 `session_compact` 발생 시 부트스트랩을 대기 상태로 표시함
- `context`에 사용자 역할 부트스트랩 메시지를 주입함
- 선두의 `compactionSummary` 메시지 뒤에 압축 후 부트스트랩을 삽입함
- `agent_end` 시 대기 중인 부트스트랩을 지움

- [ ] **단계 5: 테스트를 실행하고 GREEN 검증**

실행: `node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

예상 결과: 통과.

### 작업 2: Pi 도구 매핑 참조 문서

**파일:**
- 생성: `skills/using-superpowers/references/pi-tools.md`
- 수정: `tests/pi/test-pi-extension.mjs`

- [ ] **단계 1: Pi 참조 문서에 대한 실패하는 테스트 작성**

`skills/using-superpowers/references/pi-tools.md`가 존재하고 `Skill`, `Task`, `TodoWrite` 및 내장 도구 이름에 대한 매핑을 문서화함을 단정하는 테스트를 추가합니다.

- [ ] **단계 2: 테스트를 실행하고 RED 검증**

실행: `node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

예상 결과: 실패 — `pi-tools.md`가 존재하지 않기 때문.

- [ ] **단계 3: Pi 참조 문서 추가**

Pi 네이티브 스킬, 선택적 `pi-subagents`, 정식 todo/tasklist 플러그인의 부재, 내장 소문자 도구를 설명하는 `skills/using-superpowers/references/pi-tools.md`를 생성합니다.

- [ ] **단계 4: 테스트를 실행하고 GREEN 검증**

실행: `node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

예상 결과: 통과.

### 작업 3: Drill Pi 백엔드 및 세션 로그 정규화

**파일:**
- 생성: `evals/backends/pi.yaml`
- 수정: `evals/drill/backend.py`
- 수정: `evals/drill/engine.py`
- 수정: `evals/drill/normalizer.py`
- 수정: `evals/tests/test_backend.py`
- 수정: `evals/tests/test_normalizer.py`

- [ ] **단계 1: 실패하는 백엔드/정규화기 테스트 작성**

다음 항목에 대한 pytest 커버리지를 추가합니다:
- `load_backend("pi")`가 `family == "pi"`를 반환함
- Pi 백엔드 명령어가 `pi`로 시작하고 `-e ${SUPERPOWERS_ROOT}`를 포함함
- Pi용 `_resolve_log_dir()`가 `~/.pi/agent/sessions` 아래를 가리킴
- `filter_pi_logs_by_cwd()`가 헤더 `cwd`가 시나리오 작업 디렉터리와 일치하는 세션 파일만 유지함
- `normalize_pi_logs()`가 Pi 어시스턴트 세션 엔트리에서 `toolCall` 블록을 추출하고 내장 소문자 도구를 정식 이름으로 매핑함

- [ ] **단계 2: 테스트를 실행하고 RED 검증**

실행: `uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

예상 결과: 실패 — Pi 백엔드 및 정규화기가 존재하지 않기 때문.

- [ ] **단계 3: `evals/backends/pi.yaml` 추가**

백엔드가 `pi -e ${SUPERPOWERS_ROOT}`를 실행하도록 설정하고, 허용 범위가 넓은 TUI 준비 상태(readiness), `/quit` 종료 및 Pi 세션 로그 위치를 사용하도록 구성합니다.

- [ ] **단계 4: Pi 패밀리 지원 구현**

Pi 로그 필터링 및 정규화를 적용하여 `Backend.family`, `Engine._resolve_log_dir`, `Engine._collect_tool_calls` 및 `normalizer.py`를 업데이트합니다.

- [ ] **단계 5: 테스트를 실행하고 GREEN 검증**

실행: `uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

예상 결과: 통과.

### 작업 4: 문서화 및 전체 검증

**파일:**
- 수정: `README.md`
- 수정: `evals/README.md`

- [ ] **단계 1: Pi 설치 및 평가 백엔드 문서화**

README 빠른 시작/설치 목록에 Pi를 추가하고 `evals/README.md`에 백엔드 항목/사용법을 추가합니다.

- [ ] **단계 2: 검증 실행**

실행:
```bash
node --experimental-strip-types --test tests/pi/test-pi-extension.mjs
uv run pytest evals/tests/test_backend.py evals/tests/test_setup.py evals/tests/test_normalizer.py -q
```

예상 결과: 모든 테스트 통과.
