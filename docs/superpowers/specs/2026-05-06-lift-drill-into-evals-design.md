# Drill을 `evals/`로 Superpowers에 이관하기 — 설계 스펙

## 배경

Drill은 `obra/drill` 저장소에 독립적으로 존재하던 Python 기반의 스킬 준수(skill-compliance) 벤치마크입니다. 실제 tmux 세션을 실행하고, 시뮬레이션된 사용자로 LLM actor를 실행하며, 생성된 트랜스크립트에 대해 LLM verifier를 실행하여 시나리오별 합격/불합격을 보고합니다. Claude Code, Codex, Gemini CLI, 그리고 (최근 커밋에 따라) OpenCode 및 Copilot CLI를 지원합니다.

Drill은 이미 superpowers의 *사실상의(de facto)* 이발(eval) 하네스입니다. drill 저장소의 PRI-1397 커밋 시리즈는 약 22개의 superpowers bash 테스트를 drill 시나리오로 이관했으며, 가장 최근의 superpowers 커밋(`a2292c5`)은 *"drill 행동 커버리지로 대체됨"*이라는 메시지와 함께 중복된 bash 테스트를 명시적으로 제거했습니다. 마이그레이션 추진력이 존재하므로, 이 스펙은 이를 완료합니다.

이 작업은 drill을 `evals/`라는 이름으로 superpowers 내부로 이동하고, drill 시나리오의 커버리지를 파일별로 검증한 후 중복된 bash 테스트를 삭제하며, 기여자가 새로운 구조에 도달할 수 있도록 문서를 업데이트합니다.

## 목표

1. `evals/`는 superpowers의 표준 이발 하네스가 됨 — 전체 drill 소스, 시나리오, 픽스처, 프롬프트, 백엔드 설정 및 테스트 포함.
2. drill 시나리오에 의해 100% 커버되는 것으로 개별 검증된 `superpowers/tests/` 내의 bash 테스트는 삭제됨; 나머지는 보존됨.
3. `tests/` (플러그인 인프라: bash + node + python 통합 테스트)와 `evals/` (actor + verifier 기반의 LLM 동작) 간의 분리가 의미 있게 유지되고 문서화됨.
4. 최상위 문서 (`README.md`, `CLAUDE.md`, `docs/testing.md`)가 기여자를 올바른 위치로 안내함.
5. 독립된 `obra/drill` 저장소는 계속 존재하며 (이 PR은 이를 건드리지 않음), 이 PR이 머지된 후 별도의 수동 단계를 통해 아카이브 처리됨.

## 비목표 (Non-goals)

- **CI 통합.** 이 작업에서는 수동 실행만 다룹니다. 자물쇠를 푸는 향후 작업은 "티어 방식"입니다: 매 PR마다 빠른 서브셋 실행, 야간 및 요청 시 전체 스위프 실행. 이를 위해서는 API 예산 결정, GitHub Actions 보안 비밀, 그리고 `tmux` + `node` + `python` + `claude` / `codex` / `gemini` CLI가 설치된 러너 이미지가 필요합니다. 본 범위 밖입니다.
- **스킬과 시나리오의 동일 위치 배치 (Co-location).** 시나리오는 `evals/scenarios/`에 중앙 집중화되어 유지됩니다. 향후 각 스킬이 자체 시나리오를 소유하기로 결정하더라도, 이는 경로 찾기 및 이름 변경 작업일 뿐이며 YAML 형식은 변경되지 않습니다.
- **내부 Python 패키지 이름 변경** (`drill` → `evals`). 디렉토리는 `evals/` (사용자 대면)입니다; Python 패키지는 diff를 작게 유지하기 위해 `drill` 이름을 유지합니다. `evals/README.md`에 짧은 노트를 통해 설명합니다.
- **Drill 저장소 아카이브.** 이 PR은 `obra/drill`을 건드리지 않습니다. 머지 후 drill 저장소는 수동으로 아카이브됩니다 (GitHub에서 읽기 전용, README에서 `obra/superpowers/evals/`로 안내).
- **`tests/claude-code/analyze-token-usage.py`를 `evals/bin/`으로 이동.** 유용한 유틸리티이지만 테스트 코드가 아닙니다. 나중에 이동할 수 있으며 이 PR에서 필수 사항은 아닙니다.

## 브랜치 전략

`dev`에서 `f/evals-lift` 브랜치를 생성합니다. 이 작업은 열려 있는 `f/cross-platform` PR과 독립적입니다 — `README.md` 외에는 공유 파일 변경이 없으며, `README.md`는 충돌이 발생하더라도 머지 시점에 쉽게 해결할 수 있을 만큼 적은 변경입니다.

## 이동 후 아키텍처

```
superpowers/
  evals/                              ← 신규 (전체 drill 복사본)
    pyproject.toml                    (Python 3.11, uv 관리)
    uv.lock
    .gitignore                        (drill 자체 gitignore; results/, .venv/, .env)
    README.md                         (이전 drill의 README; 설치 지침 업데이트됨)
    CLAUDE.md                         (이전 drill의 CLAUDE.md; 경로 업데이트됨)
    docs/
      design.md                       (drill 설계 문서 — 원문 그대로 보존, 이 스펙에서 상호 링크)
      manual-testing.md
      pressure-and-red-testing.md
    drill/                            (Python 패키지; 이름 유지; cli, engine, actor, verifier 등)
    backends/                         (claude-*.yaml, codex.yaml, gemini.yaml)
    scenarios/                        (32개 이상의 YAML 시나리오)
    setup_helpers/                    (15개 Python 헬퍼; create_base_repo, sdd_*, spec_*, worktree 등)
    fixtures/                         (template-repo, sdd-go-fractals, sdd-svelte-todo)
    prompts/                          (actor.md, verifier.md)
    bin/                              (어설션 헬퍼 스크립트: tool-called, tool-count 등)
    tests/                            (drill 자체 pytest 수트)

  tests/                              ← 기본적으로 보존되는 bash 테스트들
    brainstorm-server/                ← 유지 (brainstorm-server JS 코드를 위한 node 테스트)
    opencode/                         ← 유지 (플러그인 로딩 테스트)
    codex-plugin-sync/                ← 유지 (동기화 검증)
    claude-code/                      ← 대부분 유지 — 삭제 게이트 참조
    explicit-skill-requests/          ← 대체 검증될 때까지 유지
    skill-triggering/                 ← 대체 검증될 때까지 유지
    subagent-driven-dev/              ← 대체 검증될 때까지 유지

  docs/
    testing.md                        ← 업데이트됨 ("Plugin tests" + "Skill behavior evals"로 분리)
    superpowers/
      specs/
        2026-05-06-lift-drill-into-evals-design.md   ← 본 스펙

  README.md                           ← evals/로 안내하는 소규모 Contributing 섹션 포인터
  CLAUDE.md                           ← 한 줄짜리 "Eval harness lives at evals/" 포인터
```

이 PR 이후 `tests/` 및 `evals/` 디렉토리는 명확히 구별되는 역할을 수행합니다:

- **`tests/`** — 플러그인의 비-LLM 코드가 작동하는가? brainstorm-server JS 코드, OpenCode 플러그인 로딩, codex-plugin-sync 동기화 검증을 위한 단위 및 통합 테스트. Bash + node + python.
- **`evals/`** — 에이전트가 실제 LLM 세션에서 올바르게 동작하는가? actor + verifier가 포함된 Drill 시나리오. Python 전용, 실제 tmux 세션 실행.

## 삭제 게이트 (bash 테스트별 기준)

bash 테스트는 drill 시나리오가 해당 테스트의 모든 어설션을 100% 커버함이 검증된 경우에만 삭제됩니다. 구현 플랜은 파일별로 이 검증 내용을 문서화합니다: bash 테스트를 읽고, 검사 항목을 열거하며, drill 시나리오를 찾고, 각 검사 항목에 일치하는 `verify.assertions` 또는 `verify.criteria` 항목이 있는지 확인합니다. 단 하나의 검사 항목이라도 누락된 경우, 옵션은 drill 시나리오를 확장하거나 bash 테스트를 유지하는 것입니다. 기본값은 유지하는 것입니다.

**잠정 커밋 커버리지 맵** (커밋 메시지 기반; 삭제 전 파일별 검증 필요):

| Bash test | Claimed drill replacement | Coverage status |
|-----------|---------------------------|-----------------|
| `tests/skill-triggering/prompts/*` (6개 프롬프트 파일) | `triggering-*.yaml` (6개 시나리오) | 후보 — 삭제 전 프롬프트별 검증 |
| `tests/skill-triggering/run-test.sh`, `run-all.sh` | 해당 없음 (테스트가 아닌 러너) | **유지** — 러너 스크립트 |
| `tests/explicit-skill-requests/prompts/please-use-brainstorming.txt` | 검증 필요 — drill에 명확한 대응 항목이 아직 없음 | drill 시나리오가 추가되지 않는 한 **유지** 가능성 높음 |
| `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` | 검증 필요 — drill에 명확한 대응 항목 없음 | drill 시나리오가 추가되지 않는 한 **유지** 가능성 높음 |
| `tests/explicit-skill-requests/run-claude-describes-sdd.sh` | 부분적 → `mid-conversation-skill-invocation.yaml` | 후보 — 스크립트별 검증 |
| `tests/explicit-skill-requests/run-haiku-test.sh` | Haiku 전용 동작을 다루는 drill 시나리오 없음 | **유지** |
| `tests/explicit-skill-requests/run-multiturn-test.sh`, `run-extended-multiturn-test.sh` | 다중 턴 구축을 다루는 drill 시나리오 없음 | drill 시나리오가 추가되지 않는 한 **유지** |
| `tests/explicit-skill-requests/run-test.sh`, `run-all.sh` | 해당 없음 (러너) | **유지** |
| `tests/subagent-driven-dev/go-fractals/`, `tests/subagent-driven-dev/svelte-todo/` | `sdd-go-fractals.yaml`, `sdd-svelte-todo.yaml` | 후보 — 삭제 전 검증 (테스트 수트 통과에 대한 실제 어설션 포함됨) |
| `tests/claude-code/test-document-review-system.sh` | `spec-reviewer-catches-planted-flaws.yaml` | 후보 — 삭제 전 검증 |
| `tests/claude-code/test-requesting-code-review.sh` | `code-review-catches-planted-bugs.yaml` | 후보 — 삭제 전 검증 |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | `sdd-rejects-extra-features.yaml` (YAGNI 서브셋) | **부분적** — bash 테스트는 ≥3개 커밋 / `npm test` 통과 / `analyze-token-usage.py` 실행도 어설션함. Drill 시나리오는 forbidden-exports + reviewer-as-gate 어설션함. 대부분 일치하지 않음 — 거의 확실히 **유지 + drill 시나리오 확장**. |
| `tests/claude-code/test-subagent-driven-development.sh` | 메타/문서화 테스트 (에이전트에게 SDD를 *설명*하도록 요청); 설명 테스트를 다루는 drill 시나리오 없음 | drill 시나리오가 추가되지 않는 한 **유지** |
| `tests/claude-code/test-worktree-native-preference.sh` | `worktree-creation-under-pressure.yaml` | 후보 — 삭제 전 검증 |
| `tests/claude-code/test-helpers.sh`, `run-skill-tests.sh`, `analyze-token-usage.py` | 해당 없음 (테스트가 아닌 유틸리티) | **유지** — 라이브러리/도구 |

## 검증 프로토콜 (서브에이전트 게이트 방식)

구현 플랜의 모든 변경사항은 커밋 전 독립적인 서브에이전트에 의해 크로스 체크를 받습니다.

| Change category | Subagent verification |
|----------------|----------------------|
| 각 bash-test 삭제 | (a) bash 테스트 파일 내용, (b) 후보 drill 시나리오 YAML, (c) 프롬프트: *"bash 테스트가 수행하는 모든 어설션을 나열하라. drill 시나리오의 모든 verify 항목을 나열하라. 각 bash 어설션에 대해 일치하는 drill 검사를 찾거나 일치하지 않음을 보고하라. 어설션별 표를 출력하라."*를 포함하여 서브에이전트 디스패치. 서브에이전트의 출력이 게이트임 — 모든 bash 어설션에 일치 항목이 있는 경우에만 삭제. |
| 초기 `evals/` 복사 | 서브에이전트 검증: (a) 출처를 감사할 수 있도록 복사 시점의 drill SHA가 이관 커밋 메시지에 기록됨; (b) 파일 수뿐만 아니라 모든 파일에 대해 **파일별 SHA-256 체크섬**이 drill 저장소와 일치함; (c) 제외된 경로(`.git/`, `.venv/`, `results/`, `.env`, `__pycache__/`, `*.egg-info/`, 모든 `.private-journal/`)가 `evals/`에 존재하지 않음; (d) 모든 백엔드 YAML이 이동 후 존재하는 경로를 참조함; (e) `pyproject.toml`, `uv.lock`, `.gitignore`가 손상되지 않음. |
| Drill 자체 pytest 수트 | 서브에이전트가 경로 기본값 변경 후 `cd evals && uv run pytest` 실행. Drill은 `SUPERPOWERS_ROOT` 환경 변수 동작을 검사하는 `test_backend.py`를 포함하여 `evals/tests/`에 자체 pytest 수트를 포함함 — 이 테스트들은 헬퍼와 맞추어 업데이트되어야 하며 계속 통과해야 함. |
| 삭제 후 참조 스크러빙 | 서브에이전트가 전체 superpowers 트리(`node_modules/`, `.venv/`, `evals/` 제외)에서 삭제된 bash 테스트 경로 참조를 grep함. 검색 대상: `docs/`, `docs/superpowers/plans/`, `RELEASE-NOTES.md`, `CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, `README.md`, `.github/`, `scripts/`, `.opencode/INSTALL.md`, `.codex-plugin/INSTALL.md`, `lefthook.yml`. 일치 항목은 업데이트되거나 누락된 의존성을 표면화함. |
| 경로 기본값 변경 (`SUPERPOWERS_ROOT` 기본값) | 서브에이전트가 경로 변경 후 적어도 하나의 저렴한 drill 시나리오(예: `triggering-test-driven-development`)를 실행하고 여전히 통과함을 확인함. 단순 코드 리뷰가 아닌 실제 검증. |
| 최종 PR 전 적대적 검토 (Adversarial review) | 병렬로 2개의 서브에이전트 실행, "가장 타당한 이슈를 찾은 사람에게 5점 부여" 프레이밍 — 크로스 플랫폼 PR에서 사용된 것과 동일한 프로토콜. 소스 코드와 동작 모두 검증. |

각 서브에이전트 작업은 명시적 입력 및 통과 기준과 함께 구현 플랜에 자체 불릿을 갖습니다. 추적이 감사 가능하도록 서브에이전트의 출력은 관련 커밋 메시지("Subagent verification: …")에 요약됩니다.

## 구체적인 경로/설정 편집

**본 스펙 작성 전 검증됨.** `drill/cli.py`는 `PROJECT_ROOT = Path(__file__).parent.parent`를 정의합니다. 이동 후 `cli.py`는 `evals/drill/cli.py`에 위치하므로, `PROJECT_ROOT`는 `evals/`로 해석되고 `PROJECT_ROOT.parent`는 superpowers 저장소 루트로 해석됩니다. 이것이 `SUPERPOWERS_ROOT`가 기본적으로 가져야 할 값입니다.

**YAML 치환 감사.** 4개의 `claude*.yaml` 백엔드 설정만 `args`에 `${SUPERPOWERS_ROOT}`를 보간(interpolate)합니다 (`--plugin-dir` 플래그를 위해); `codex.yaml` 및 `gemini.yaml`은 pre/post-run 훅의 `engine.py:233` / `setup.py:25`에 있는 `os.environ["SUPERPOWERS_ROOT"]` 조회에서 소비되는 `required_env`에만 `SUPERPOWERS_ROOT`를 나열합니다. 헬퍼의 `os.environ` 변경은 두 코드 경로를 모두 다룹니다.

| File | Current | After |
|------|---------|-------|
| `drill/cli.py` | 모듈 임포트 시 `load_dotenv(PROJECT_ROOT / ".env")`; `SUPERPOWERS_ROOT` 관련 내용 없음 | `load_dotenv` 호출 후, 아직 설정되어 있지 않은 경우에만 `os.environ["SUPERPOWERS_ROOT"]`를 `str(PROJECT_ROOT.parent)`로 설정하는 새 헬퍼 `_set_superpowers_root_default()` 호출. 순서: `load_dotenv` → 기본값 설정 → click 그룹 정의. |
| `drill/engine.py:233`, `drill/setup.py:25` | `os.environ["SUPERPOWERS_ROOT"]` 직접 접근 (미설정 시 KeyError) | 변경 없음. CLI 시작 훅이 engine/setup 실행 시점에 환경 변수가 설정되어 있음을 보장함. |
| `backends/claude*.yaml` (5개 파일) | `--plugin-dir`을 위한 `args`에 `${SUPERPOWERS_ROOT}` 보간됨 | 변경 없음. YAML 보간은 CLI 시작 후인 백엔드 로드 시점에 `os.environ`을 읽음. |
| `backends/codex.yaml`, `backends/gemini.yaml` | `required_env`에만 `SUPERPOWERS_ROOT` 존재 | `required_env`에서 제거 (헬퍼가 제공함). `claude*.yaml`은 호환성을 위해 `required_env` 유지 (환경 변수가 재정의로 작동함). |
| `evals/tests/test_backend.py` | 테스트가 `SUPERPOWERS_ROOT`가 `required_env` 목록에 있음을 어설션함, 및 경로 해석 테스트 포함 | 새로운 계약에 맞게 테스트 업데이트: 헬퍼가 제공하는 기본값, 환경 변수 재정의가 여전히 작동함, codex/gemini에 대해 `required_env`가 더 이상 요구되지 않음. |
| `evals/README.md` | "export SUPERPOWERS_ROOT=/path/to/superpowers" | export 줄 제거; 환경 변수가 `evals/` 상위 경로로 자동 기본 지정됨을 명시; 유일하게 필수적인 설정은 `ANTHROPIC_API_KEY` (또는 `OPENAI_API_KEY` / Gemini 인증)임을 언급. |
| `evals/CLAUDE.md` | 동일함 | 동일함 |
| `evals/.gitignore` | drill의 기존 패턴 (`results/`, `.venv/`, `__pycache__/`, `.env`, `*.pyc`, `*.egg-info/`, `dist/`, `build/`, `.claude/`) | 그대로 복사됨. 패턴은 파일 위치에 상대적이므로 `evals/` 아래에 올바르게 적용됨. |
| `evals/lefthook.yml` | drill이 `pre-commit: uv run ruff check && uv run ty check`를 정의하는 `lefthook.yml`을 제공함 | `evals/lefthook.yml`로 이동. (a) superpowers 루트에 lefthook을 설치하고 `evals/lefthook.yml`로 연동하거나, (b) 기여자가 수동으로 `cd evals && lefthook run pre-commit`을 실행하도록 문서화. **구현 결정: 단순성을 위해 옵션 (b) 선택** — superpowers의 최상위 워크플로우는 변경되지 않음. |

`.env` 위치: `evals/.env` 유지 (gitignored). 기여자는 거기서 구성을 제공받거나 쉘 환경에서 `ANTHROPIC_API_KEY`를 설정합니다.

**소규모 추가가 필요한 최상위 superpowers 파일들:**

- `superpowers/.gitignore`: `evals/results/`, `evals/.venv/`, `evals/.env` 추가 (만약을 대비함; evals/.gitignore가 이미 로컬에서 이를 다룸).
- `superpowers/CLAUDE.md`: 에이전트가 이를 발견할 수 있도록 한 줄짜리 포인터 "Eval harness lives at `evals/` — see `evals/README.md`" 추가.
- `superpowers/docs/testing.md`: "## Plugin tests" (삭제된 테스트 참조가 정돈된 기존 tests/ 내용) 및 "## Skill behavior evals" (한 단락 요약 + `evals/` 포인터)로 분리.
- `superpowers/README.md`: 스킬 동작 테스트를 위해 `evals/`를 가리키는 한 줄을 Contributing 섹션에 추가.

## 마이그레이션 순서

각 단계는 별도의 커밋 (또는 소규모 커밋 그룹)입니다. Step 2가 가장 큰 단일 커밋(원문 그대로의 drill 복사)이며, 이후 단계는 소규모의 원자적 커밋입니다.

```
1. `dev`에서 브랜치 생성 (f/evals-lift)

2. drill 저장소를 evals/에 복사 (단일 커밋, 쉽게 되돌릴 수 있음)
   ├─ 복사 시점의 drill SHA를 커밋 메시지에 기록
   ├─ `rsync -a --exclude=.git --exclude=.venv --exclude=results
   │  --exclude=.env --exclude=__pycache__ --exclude='*.egg-info'
   │  --exclude=.private-journal /path/to/drill/ evals/` 사용
   │  (명시적 제외를 위해 `cp -r` 대신 rsync 선택;
   │  `find evals -name '.git' -type d`가 아무것도 반환하지 않는지 검증)
   ├─ 서브에이전트 게이트: 제외되지 않은 모든 파일에 대해 파일별 SHA-256 체크섬이
   │  drill 저장소와 일치함; 제외된 경로가 evals/에 존재하지 않음
   └─ 스모크 체크: `cd evals && uv sync` 성공 (설치 성공만 증명;
      동작 테스트는 아님)

3. 경로 기본값 업데이트
   ├─ drill/cli.py에 _set_superpowers_root_default() 헬퍼 추가
   ├─ load_dotenv 뒤, click 그룹 정의 전으로 연결
   ├─ evals/README.md 및 evals/CLAUDE.md 업데이트 (SUPERPOWERS_ROOT 설치 단계 제거)
   ├─ codex.yaml/gemini.yaml의 required_env에서 SUPERPOWERS_ROOT 제거
   │  (claude*.yaml에는 재정의용으로 유지)
   └─ 새로운 계약에 맞게 evals/tests/test_backend.py 업데이트

4. 새 위치에서 검증 (두 가지 검사)
   ├─ drill 자체 pytest 실행: `cd evals && uv run pytest` — 통과해야 함
   └─ 저렴한 drill 시나리오 실행: `cd evals && uv run drill run
      triggering-test-driven-development -b claude` — 통과해야 함.
      단순 코드 리뷰가 아닌 실제 동작 검증.

5. Bash 테스트 삭제 단계 — 서브에이전트 게이트가 포함된 파일별 진행
   삭제 후보 목록의 각 파일에 대해:
   a. 서브에이전트가 bash 테스트 어설션과 drill 시나리오 verify 블록을 비교
   b. 통과 기준: 모든 bash 어설션에 일치하는 drill 검사 존재
   c. 통과 시 → bash 테스트 파일 삭제 (파일당 또는 연관 그룹당 1커밋)
   d. 실패 시 → drill 시나리오 확장(별도 커밋 + 검증) 또는
      bash 테스트 유지 (커밋 없음)

6. 오래된 참조 스크러빙 (Stale-reference scrub)
   ├─ 서브에이전트가 삭제된 파일 경로에 대해 superpowers 트리(node_modules/, .venv/,
   │  evals/ 제외)를 grep함
   ├─ 검색 대상: docs/, docs/superpowers/plans/, RELEASE-NOTES.md,
   │  CLAUDE.md, GEMINI.md, AGENTS.md, README.md, .github/, scripts/,
   │  .opencode/INSTALL.md, .codex-plugin/INSTALL.md, lefthook.yml
   ├─ 활성 참조 업데이트 (예: docs/testing.md, README.md 설치)
   └─ docs/superpowers/plans/*.md 및 RELEASE-NOTES.md의 역사적 참조는
      재작성하지 않고 주석("(test removed; behavior covered by drill scenario X)")과
      함께 보존됨 — 이들은 날짜가 지정된 아티팩트이지 살아있는 문서가 아님.

7. 최상위 문서
   ├─ docs/testing.md 분리
   ├─ CLAUDE.md 포인터
   └─ README.md Contributing 섹션

8. 스모크 체크 재실행 (회귀 게이트)
   ├─ `cd evals && uv run pytest`
   └─ `cd evals && uv run drill run triggering-test-driven-development -b claude`

9. 최종 적대적 검토
   └─ 병렬로 2개 서브에이전트 실행, 전체 diff, "가장 타당한 이슈를
      찾은 사람에게 5점 부여" 프레이밍. 푸시 전 지적 사항 처리.

10. 브랜치 푸시 + dev 대상 PR 생성
    └─ PR 설명 포함 사항: 복사 시점 고정된 drill SHA, 아카이브 조치
       항목 ("머지 후: obra/drill 아카이브 처리, obra/superpowers/evals/로 안내하는
       README 포인터 추가"), 삭제된 파일별 커버리지 영수증.
```

## 검증 (구현 후)

구현 플랜은 다음을 보여주어야 합니다:

- Step 2 이후 제외되지 않은 모든 drill 소스 파일이 `evals/`에 존재함 (`obra/drill@<recorded-sha>` 대비 서브에이전트의 **파일별 SHA-256 체크섬 diff**).
- 제외된 경로(`.git/`, `.venv/`, `results/`, `.env`, `__pycache__/`, `*.egg-info/`, `.private-journal/`)가 `evals/`에 존재하지 않음.
- Step 2 커밋 메시지가 drill 소스 SHA를 기록함.
- `SUPERPOWERS_ROOT`가 설정되지 않은 상태에서 `cd evals && uv sync`가 성공함.
- `cd evals && uv run pytest`가 통과함 (drill 자체 pytest 수트).
- `cd evals && uv run drill list`가 기록된 SHA의 독립 실행형 drill 저장소와 동일한 시나리오 수를 반환함.
- `cd evals && uv run drill run triggering-test-driven-development -b claude`가 통과함 (경로 기본값이 엔드투엔드로 작동함을 증명).
- 삭제된 각 bash 테스트에 대해: 커밋 메시지에 drill 검사에 매핑된 모든 어설션을 보여주는 서브에이전트 검증 테이블 포함.
- 삭제된 파일 경로에 대한 grep 실행 시 살아있는 superpowers 문서 전반에서 결과 0건 반환 (Step 6 이후); `docs/superpowers/plans/*.md` 및 `RELEASE-NOTES.md`에 있는 역사적 참조는 주석 처리되며 재작성되지 않음.
- `docs/testing.md`에 "Plugin tests" 및 "Skill behavior evals" 섹션이 모두 존재함.
- drill 저장소의 기록이 건드려지지 않음; `obra/drill`은 이 PR에 의해 영향받지 않음.
- PR 설명에 머지 후 `obra/drill`을 아카이브할 조치 항목 명시.

## 미결 질문 (Open questions)

없음. 모든 명확화 결정이 완료되었습니다:

| Question | Decision |
|----------|----------|
| Drill이 superpowers 내부 어디에 위치하는가? | `evals/` (drill에서 이름 변경); 독립 저장소는 별도 단계로 아카이브됨 |
| 중복된 bash 테스트의 운명은? | 커버리지에 대한 서브에이전트 검증을 거쳐 파일별 삭제; 기본값은 유지 |
| 시나리오 레이아웃은? | `evals/scenarios/`에 중앙 집중화 |
| Python 툴체인 위치는? | `evals/`에 자급자족 형태로 위치 |
| CI 통합은? | 이 PR에서는 수동 전용; 향후 경로 문서화됨 |
| 마이그레이션 메커니즘은? | 단순 복사; drill 저장소의 기록은 아카이브된 저장소에 보존되며 트리에 포함되지 않음 |
| 내부 Python 패키지 이름은? | `drill`로 유지 (디렉토리는 `evals/`) |
| 브랜치 전략은? | `dev`에서 독립 실행 (`f/cross-platform` 위에 쌓이지 않음) |
