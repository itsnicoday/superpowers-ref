# drill을 `evals/`로 superpowers에 이전(lift)하기 — 구현 계획

> **에이전트 작업자 참고:** 필수 하위 스킬: 이 계획을 작업별로 구현하려면 superpowers:subagent-driven-development (권장) 또는 superpowers:executing-plans를 사용하세요. 각 단계는 추적을 위해 체크박스 (`- [ ]`) 구문을 사용합니다.

**목표:** 독립형 `obra/drill` 스킬 준수 벤치마크를 최상위 `evals/` 디렉터리로 superpowers에 이전하고, 드릴 시나리오 커버리지에 대한 파일별 하위 에이전트 검증 후 `superpowers/tests/` 아래의 중복 bash 테스트를 삭제하며, 기여자가 새로운 구조에 쉽게 적응할 수 있도록 최상위 문서를 업데이트합니다.

**아키텍처:** 새 브랜치 `f/evals-lift`에서 `dev`를 타겟으로 하는 단일 PR. 새 디렉터리에서 `.git/`, `.venv/` 등을 제외하기 위해 명시적인 rsync exclude 옵션을 사용하여 드릴 소스를 그대로 복사합니다. `drill/cli.py`의 작은 헬퍼가 `SUPERPOWERS_ROOT`의 기본값을 `evals/` 디렉터리의 상위 폴더로 설정하므로 기여자가 환경 변수를 별도로 설정할 필요가 없습니다. 각 bash 테스트 삭제는 bash 테스트의 단정문(assertion)과 주장하는 드릴 시나리오의 verify 블록을 비교하는 하위 에이전트에 의해 게이트(gate) 처리됩니다. 계획 문서 및 릴리스 노트의 과거 참조는 재작성하지 않고 주석(annotation)을 답니다.

**기술 스택:** Python 3.11 + uv (드릴의 기존 툴체인, 변경 없음); rsync; bash; git.

**스펙:** `docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md` — 이것부터 읽어보세요.

**Drill 소스 위치:** `/Users/jesse/Documents/GitHub/superpowers/drill/` (`superpowers/`와 형제 디렉터리).

---

## 작업 1: dev에서 브랜치 생성

**파일:** 없음 (git 작업만 수행)

- [ ] **1단계: 깨끗한 작업 트리 확인**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git status --short
```

예상: 빈 출력 (또는 추적되지 않는 `.opencode/package-lock.json`만 출력되면 괜찮음).

- [ ] **2단계: 최신 dev 가져오기**

```bash
git fetch origin dev:dev
```

- [ ] **3단계: 브랜치 생성**

```bash
git checkout -b f/evals-lift dev
```

예상: `Switched to a new branch 'f/evals-lift'`.

- [ ] **4단계: 정상 동작 확인(Sanity check)**

```bash
git log --oneline -1
```

예상: 출력이 `origin/dev`가 가리키는 커밋으로 시작함 (현재 `b4363df docs: turned the dash in "- Jesse" into an escape sequence (#1474)`).

---

## 작업 2: 복사 시점의 drill SHA 캡처

**파일:** 없음 (이전 커밋 메시지에 사용할 값을 기록)

- [ ] **1단계: 현재 drill HEAD SHA 가져오기**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
DRILL_SHA=$(git rev-parse HEAD)
echo "$DRILL_SHA"
```

- [ ] **2단계: drill에 커밋되지 않은 작업이 없는지 확인**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
git status --short
```

예상: 비어 있음 (추적되지 않거나 수정된 파일 없음). 출력이 비어 있지 않으면 중단하고 보고하세요 — 이전 전 drill 작업 트리는 깨끗해야 합니다. 그렇지 않으면 SHA 고정이 의미가 없습니다.

- [ ] **3단계: 다음 작업을 위해 셸 환경 변수에 SHA 저장**

```bash
echo "DRILL_SHA=$DRILL_SHA"  # 작업 3에서 사용할 수 있도록 기록해 둡니다
```

---

## 작업 3: drill을 evals/로 rsync 복사

**파일:**
- 생성: `evals/` (제외 항목을 뺀 drill의 전체 디렉터리 트리)

- [ ] **1단계: 소스 및 대상 경로 확인**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
test -d /Users/jesse/Documents/GitHub/superpowers/drill && echo "drill source: OK"
test ! -d evals && echo "evals/ does not yet exist: OK"
```

예상: 두 echo 명령어 모두 출력됨.

- [ ] **2단계: 명시적 exclude 옵션과 함께 drill을 evals/로 rsync 복사**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
rsync -a \
  --exclude=.git \
  --exclude=.venv \
  --exclude=results \
  --exclude=.env \
  --exclude=__pycache__ \
  --exclude='*.egg-info' \
  --exclude=.private-journal \
  --exclude='*.pyc' \
  /Users/jesse/Documents/GitHub/superpowers/drill/ \
  evals/
```

- [ ] **3단계: exclude 옵션이 올바르게 작동했는지 확인**

```bash
find evals -name '.git' -type d
find evals -name '.venv' -type d
find evals -name 'results' -type d
find evals -name '.env'
find evals -name '__pycache__' -type d
find evals -name '*.egg-info' -type d
```

예상: 모든 명령어가 아무것도 출력하지 않음. 경로가 출력되는 경우 계속 진행하기 전에 수동으로 `rm -rf` 하세요.

- [ ] **4단계: 커밋 메시지에 사용할 소스 SHA 확인**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
DRILL_SHA=$(git rev-parse HEAD)
echo "$DRILL_SHA"
```

예상: 작업 2의 1단계에서 얻은 SHA.

- [ ] **5단계: 모두 스테이징**

```bash
git add evals/
git status --short | head -20
```

예상: 추가된 수많은 파일을 나열하는 `A  evals/...` 줄로 출력이 시작됨. 대부분은 scenarios/, drill/, backends/, setup_helpers/ 등에 있음.

- [ ] **6단계: 커밋**

```bash
: "${DRILL_SHA:?Set DRILL_SHA from Task 2 before committing}"
git commit -m "$(cat <<EOF
Lift drill into evals/ at $DRILL_SHA

rsync of obra/drill@$DRILL_SHA into superpowers/evals/, excluding
.git/, .venv/, results/, .env/, __pycache__/, *.egg-info/,
.private-journal/.

The drill repo is unaffected by this commit; archival is a separate
manual step after this PR merges.

Source SHA recorded in this commit message for provenance.
EOF
)"
```

---

## 작업 4: 체크섬으로 복사본 검증

**파일:** 없음 (검증 전용)

- [ ] **1단계: drill에 존재하지만 evals에 없어야 하는 파일 목록(제외 대상) 가져오기**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
find . \
  \( -name '.git' -prune \
  -o -name '.venv' -prune \
  -o -name 'results' -prune \
  -o -name '__pycache__' -prune \
  -o -name '*.egg-info' -prune \
  -o -name '.private-journal' -prune \
  -o -name '*.pyc' -prune \
  -o -name '.env' -prune \) \
  -o -type f -print | sort > /tmp/drill-files.txt
wc -l /tmp/drill-files.txt
```

- [ ] **2단계: evals/ 내부의 파일 목록 가져오기**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
find evals -type f | sed 's|^evals/|./|' | sort > /tmp/evals-files.txt
wc -l /tmp/evals-files.txt
```

- [ ] **3단계: 두 목록의 diff 확인**

제외된 경로가 제거된 후 두 파일 목록은 정확히 일치해야 합니다.

```bash
diff /tmp/drill-files.txt /tmp/evals-files.txt
```

예상: 출력 없음.

- [ ] **4단계: 파일별 체크섬 검증**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
while read -r f; do
  sha1=$(shasum -a 256 "$f" | cut -d' ' -f1)
  sha2=$(shasum -a 256 "/Users/jesse/Documents/GitHub/superpowers/superpowers/evals/${f#./}" | cut -d' ' -f1)
  if [ "$sha1" != "$sha2" ]; then
    echo "MISMATCH: $f ($sha1 vs $sha2)"
  fi
done < /tmp/drill-files.txt | head -20
```

예상: 출력 없음 (drill과 evals 간의 모든 파일 체크섬 일치).

- [ ] **5단계: 스모크 체크 - 의존성 설치**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv sync
```

예상: `Installed N packages` 또는 이와 유사한 출력. 오류 없음.

- [ ] **6단계: 스모크 체크 - drill 목록**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run drill list 2>&1 | head -5
```

예상: 시나리오 이름으로 시작함. (누락된 SUPERPOWERS_ROOT에 대한 경고나 오류가 발생할 수 있습니다 — 괜찮습니다, 다음 작업에서 수정됨.)

- [ ] **7단계: 검증 하위 에이전트 디스패치**

다음 프롬프트로 `general-purpose` 하위 에이전트를 디스패치합니다:

```
You are verifying a verbatim copy of the drill repo at
/Users/jesse/Documents/GitHub/superpowers/drill into
/Users/jesse/Documents/GitHub/superpowers/superpowers/evals.

Verify:

1. The lift commit message records the SHA reported by:
  cd /Users/jesse/Documents/GitHub/superpowers/drill && git rev-parse HEAD

2. None of these excluded paths exist under evals/: .git/, .venv/,
results/, .env/, __pycache__/, *.egg-info/, .private-journal/.

3. Every non-excluded file in drill has a SHA-256-identical
counterpart in evals/, and there are no extra files in evals/.

4. The pyproject.toml, uv.lock, scenarios/*.yaml, backends/*.yaml,
setup_helpers/*.py, drill/*.py, prompts/*.md, fixtures/, bin/, and
docs/ are all present.

Report each check with PASS/FAIL. If any FAIL, dump enough detail
that the parent can fix.
```

하위 에이전트가 FAIL을 보고하는 경우 계속 진행하기 전에 근본적인 문제(유출된 파일 삭제, 재rsync 등)를 해결하세요.

---

## 작업 5: `SUPERPOWERS_ROOT` 기본값 헬퍼 추가

**파일:**
- 수정: `evals/drill/cli.py:11-14`

- [ ] **1단계: 현재 cli.py 헤더 읽기**

```bash
sed -n '1,20p' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/drill/cli.py
```

예상 출력:

```python
"""Drill CLI: run, compare, list."""

from __future__ import annotations

import secrets
from pathlib import Path

import click
from dotenv import load_dotenv

PROJECT_ROOT: Path = Path(__file__).parent.parent

load_dotenv(PROJECT_ROOT / ".env")
```

- [ ] **2단계: 헬퍼에 대한 실패하는 테스트 작성**

`evals/tests/test_cli.py` 파일을 열고 끝에 다음 테스트를 추가합니다:

```python
def test_set_superpowers_root_default_when_unset(monkeypatch, tmp_path):
    """When SUPERPOWERS_ROOT is unset, helper sets it to PROJECT_ROOT.parent."""
    monkeypatch.delenv("SUPERPOWERS_ROOT", raising=False)
    from drill.cli import _set_superpowers_root_default, PROJECT_ROOT

    _set_superpowers_root_default()

    import os
    assert os.environ["SUPERPOWERS_ROOT"] == str(PROJECT_ROOT.parent)


def test_set_superpowers_root_default_respects_existing(monkeypatch):
    """When SUPERPOWERS_ROOT is already set, helper does not override."""
    monkeypatch.setenv("SUPERPOWERS_ROOT", "/custom/path")
    from drill.cli import _set_superpowers_root_default

    _set_superpowers_root_default()

    import os
    assert os.environ["SUPERPOWERS_ROOT"] == "/custom/path"
```

- [ ] **3단계: 테스트 실행 및 실패 확인**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_cli.py -k set_superpowers_root_default -v
```

예상: 2개 테스트가 `AttributeError: module 'drill.cli' has no attribute '_set_superpowers_root_default'` 오류로 실패함.

- [ ] **4단계: cli.py에 헬퍼 추가**

`/Users/jesse/Documents/GitHub/superpowers/superpowers/evals/drill/cli.py` 파일을 수정합니다. 1~14행을 다음 내용으로 교체합니다:

```python
"""Drill CLI: run, compare, list."""

from __future__ import annotations

import os
import secrets
from pathlib import Path

import click
from dotenv import load_dotenv

PROJECT_ROOT: Path = Path(__file__).parent.parent

load_dotenv(PROJECT_ROOT / ".env")


def _set_superpowers_root_default() -> None:
    """Default SUPERPOWERS_ROOT to the parent of evals/ if not already set.

    Drill historically required contributors to export SUPERPOWERS_ROOT
    pointing at the superpowers checkout. After lifting drill into
    superpowers/evals/, the parent of PROJECT_ROOT is always the
    superpowers root, so we can supply this default automatically.

    Existing SUPERPOWERS_ROOT environment values are respected as overrides.
    """
    os.environ.setdefault("SUPERPOWERS_ROOT", str(PROJECT_ROOT.parent))


_set_superpowers_root_default()
```

모듈 하단의 `_set_superpowers_root_default()` 호출은 `load_dotenv()` 직후 임포트 시점에 실행됩니다. 이렇게 하면 `engine.py`와 `setup.py`(`os.environ["SUPERPOWERS_ROOT"]`를 직접 읽음) 및 YAML 보간(백엔드 YAML이 로드될 때 `os.environ`을 읽음) 모두 이 값을 볼 수 있습니다.

- [ ] **5단계: 테스트를 실행하여 통과 확인**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_cli.py -k set_superpowers_root_default -v
```

예상: 2개 테스트 통과.

- [ ] **6단계: 커밋**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/drill/cli.py evals/tests/test_cli.py
git commit -m "evals: default SUPERPOWERS_ROOT to parent of evals/ if unset

Adds _set_superpowers_root_default() to drill/cli.py, called at
module import after load_dotenv(). PROJECT_ROOT resolves to evals/
post-lift; its parent is the superpowers repo root, which is the
correct value for SUPERPOWERS_ROOT.

Existing env values are respected as overrides via os.environ.setdefault.

Tests:
- helper sets default when var is unset
- helper does not override when var is already set"
```

---

## 작업 6: 새 환경 계약을 반영하도록 백엔드 YAML 업데이트

**파일:**
- 수정: `evals/backends/codex.yaml` (`required_env`에서 `SUPERPOWERS_ROOT` 제거)
- 수정: `evals/backends/gemini.yaml` (`required_env`에서 `SUPERPOWERS_ROOT` 제거)

5개의 `claude*.yaml` 백엔드 설정은 `--plugin-dir` 플래그의 `args`로 `${SUPERPOWERS_ROOT}`를 보간하므로 — 보간에 필수적이므로 `required_env`에 `SUPERPOWERS_ROOT`를 유지합니다. codex/gemini 설정은 헬퍼가 충족하는 engine.py/setup.py의 `os.environ` 읽기를 위해서만 이를 나열했습니다.

- [ ] **1단계: 현재 상태 확인**

```bash
grep -A3 'required_env:' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/codex.yaml
grep -A2 'required_env:' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/gemini.yaml
```

예상: 출력에 `- SUPERPOWERS_ROOT` 줄 포함.

- [ ] **2단계: codex.yaml 전체 읽기**

```bash
cat /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/codex.yaml
```

- [ ] **3단계: codex.yaml 수정 — `required_env` 아래의 `- SUPERPOWERS_ROOT` 줄 삭제**

`evals/backends/codex.yaml`을 열고 다음을 찾습니다:

```yaml
required_env:
  - OPENAI_API_KEY
  - SUPERPOWERS_ROOT
```

다음으로 교체:

```yaml
required_env:
  - OPENAI_API_KEY
```

- [ ] **4단계: gemini.yaml 수정 — `required_env` 아래의 `- SUPERPOWERS_ROOT` 줄 삭제**

`evals/backends/gemini.yaml`을 열고 다음을 찾습니다:

```yaml
required_env:
  - SUPERPOWERS_ROOT
```

다음으로 교체:

```yaml
required_env: []
```

(필드를 삭제하는 대신 빈 리스트를 사용하여 YAML 스키마 검증이 실패하지 않도록 합니다.)

- [ ] **5단계: 문제가 없는지 확인하기 위해 drill의 pytest 스위트 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest -x 2>&1 | tail -20
```

예상: 모든 테스트 통과. `tests/test_backend.py`가 codex/gemini의 `required_env` 멤버십에 대해 에러를 발생하는 경우 작업 7을 참조하세요.

- [ ] **6단계: 커밋**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/backends/codex.yaml evals/backends/gemini.yaml
git commit -m "evals: drop SUPERPOWERS_ROOT from codex/gemini required_env

These backends only read SUPERPOWERS_ROOT via engine.py/setup.py's
os.environ access, which the new cli.py default helper supplies
automatically. claude*.yaml keep SUPERPOWERS_ROOT in required_env
because they interpolate \${SUPERPOWERS_ROOT} into --plugin-dir args."
```

---

## 작업 7: 새 계약에 맞게 drill의 pytest 스위트 업데이트

**파일:**
- 수정: `evals/tests/test_backend.py` (작업 6의 5단계에서 실패가 발생한 경우 테스트별 업데이트 진행)

- [ ] **1단계: 테스트 스위트 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_backend.py -v 2>&1 | tail -30
```

모든 테스트가 통과하면 5단계로 건너뜁니다(아무것도 커밋하지 않고 작업 8로 이동). 그렇지 않으면:

- [ ] **2단계: 실패한 테스트 읽기**

실패할 때마다 `evals/tests/test_backend.py`에서 테스트를 열고 단정문(assertion)을 읽습니다.

- [ ] **3단계: 단정문 업데이트**

`codex.yaml` 또는 `gemini.yaml`의 `required_env`에서 `SUPERPOWERS_ROOT` 멤버십을 단정하는 테스트의 경우: 단정문을 반전하여 부재를 확인합니다. 예:

```python
# 기존:
def test_codex_requires_superpowers_root():
    backend = load_backend("codex")
    assert "SUPERPOWERS_ROOT" in backend.required_env

# 변경 후:
def test_codex_does_not_require_superpowers_root():
    """codex.yaml dropped SUPERPOWERS_ROOT from required_env;
    the cli.py helper supplies the default."""
    backend = load_backend("codex")
    assert "SUPERPOWERS_ROOT" not in backend.required_env
```

- [ ] **4단계: 테스트 스위트 재실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest -x 2>&1 | tail -10
```

예상: 모든 테스트 통과.

- [ ] **5단계: 커밋 (1단계에서 실패가 있었던 경우에만)**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/tests/test_backend.py
git commit -m "evals: update test_backend.py for relaxed required_env contract"
```

---

## 작업 8: evals/README.md 및 evals/CLAUDE.md 업데이트

**파일:**
- 수정: `evals/README.md` (SUPERPOWERS_ROOT 설정 단계 제거)
- 수정: `evals/CLAUDE.md` (SUPERPOWERS_ROOT 설정 단계 제거)

- [ ] **1단계: evals/README.md 수정**

다음과 같이 생긴 섹션을 찾습니다:

```markdown
Required environment:
```bash
export SUPERPOWERS_ROOT=/path/to/superpowers
export ANTHROPIC_API_KEY=sk-...
```
```

다음으로 교체:

```markdown
Required environment:
```bash
export ANTHROPIC_API_KEY=sk-...
```

`SUPERPOWERS_ROOT` defaults to the parent of `evals/` (the superpowers repo root) and only needs to be set if you're running drill against a different superpowers checkout.
```

- [ ] **2단계: evals/CLAUDE.md 수정**

섹션을 찾습니다:

```markdown
## Required env

```
SUPERPOWERS_ROOT=/path/to/superpowers
ANTHROPIC_API_KEY=sk-...
```
```

다음으로 교체:

```markdown
## Required env

```
ANTHROPIC_API_KEY=sk-...
```

`SUPERPOWERS_ROOT` defaults to the parent of `evals/` (the superpowers repo root). Override only if running drill against a different superpowers checkout.
```

- [ ] **3단계: 커밋**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/README.md evals/CLAUDE.md
git commit -m "evals: drop SUPERPOWERS_ROOT setup step from README/CLAUDE

The cli.py helper now defaults the env var. Mention as override only."
```

---

## 작업 9: 새 위치에서 검증

**파일:** 없음 (검증 전용 — 수정을 요하는 항목이 없으면 커밋하지 않음)

- [ ] **1단계: drill 전체 pytest 스위트 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run pytest 2>&1 | tail -5
```

예상: 모든 테스트 통과. `unset`을 통해 상속된 환경 변수가 아닌 헬퍼를 테스트하도록 합니다.

- [ ] **2단계: drill list 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill list 2>&1 | head -10
```

예상: 누락된 SUPERPOWERS_ROOT에 대한 오류 없이 시나리오 목록 출력.

- [ ] **3단계: env 파일 로드**

```bash
set -a
source /Users/jesse/Documents/GitHub/prime-radiant-inc/sprout/.env
set +a
echo "ANTHROPIC_API_KEY set: ${ANTHROPIC_API_KEY:+yes}"
```

예상: `ANTHROPIC_API_KEY set: yes`.

- [ ] **4단계: 비용이 적은 drill 시나리오 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill run triggering-test-driven-development -b claude 2>&1 | tail -3
```

예상: `claude: 1 passed, 0 failed, 0 errors`.

FAIL인 경우 계속 진행하기 전에 디버깅하세요. 경로 기본값 변경이 가장 유력한 원인입니다. 헬퍼가 실제 실행되었는지 임시로 헬퍼 호출 뒤에 `print(os.environ["SUPERPOWERS_ROOT"])`를 추가하여 확인해 보세요.

---

## 작업 10: Bash 테스트 삭제 단계 — 하위 에이전트 게이트가 포함된 파일별 진행

각 후보 삭제 파일은 자체 하위 에이전트 검증 + 커밋을 거치게 됩니다. 후보 목록은 스펙의 커버리지 맵에서 가져옵니다. 아래 각 항목에 대해:

1. bash 테스트 파일 읽기.
2. 후보 drill 시나리오 YAML 읽기.
3. 두 내용과 비교 프롬프트를 사용하여 하위 에이전트 디스패치.
4. 하위 에이전트가 단정문 일치 표 보고.
5. 모든 bash 단정문이 일치하는 경우: bash 테스트 삭제 및 커밋.
6. 일치하지 않는 것이 있는 경우: 중단, 이관(escalate), 삭제하지 않음.

**하위 에이전트 프롬프트 템플릿 (모든 삭제 작업 시 사용):**

```
You are gating a bash test deletion. The bash test is allegedly
covered by a drill scenario; your job is to verify that claim.

BASH TEST: <paste full contents of bash test>

DRILL SCENARIO: <paste full contents of drill scenario YAML>

Output a markdown table with columns: BASH ASSERTION, DRILL CHECK,
STATUS. List EVERY assertion the bash test makes (every grep, every
[ ], every test command, every PASS/FAIL emit). For each, find a
matching drill check (in verify.assertions or verify.criteria) or
mark as UNMATCHED.

After the table, output "VERDICT: SAFE TO DELETE" if every bash
assertion has a match, otherwise "VERDICT: KEEP — N unmatched
assertions". Be conservative: if you are uncertain about a match,
mark as UNMATCHED.
```

### 작업 10a: 스킬 트리거링 프롬프트 (6개 파일)

**파일:**
- 삭제: `tests/skill-triggering/prompts/dispatching-parallel-agents.txt`
- 삭제: `tests/skill-triggering/prompts/executing-plans.txt`
- 삭제: `tests/skill-triggering/prompts/requesting-code-review.txt`
- 삭제: `tests/skill-triggering/prompts/systematic-debugging.txt`
- 삭제: `tests/skill-triggering/prompts/test-driven-development.txt`
- 삭제: `tests/skill-triggering/prompts/writing-plans.txt`
- 유지: `tests/skill-triggering/run-test.sh`, `run-all.sh`

이 프롬프트 파일들은 bash 러너의 입력 파일입니다 — 자체 단정문이 없습니다. 러너 스크립트가 단정문을 처리합니다. 각 프롬프트를 해당 drill 시나리오에 매핑합니다:

| Prompt | Drill scenario |
|--------|----------------|
| dispatching-parallel-agents.txt | triggering-dispatching-parallel-agents.yaml |
| executing-plans.txt | triggering-executing-plans.yaml |
| requesting-code-review.txt | triggering-requesting-code-review.yaml |
| systematic-debugging.txt | triggering-systematic-debugging.yaml |
| test-driven-development.txt | triggering-test-driven-development.yaml |
| writing-plans.txt | triggering-writing-plans.yaml |

- [ ] **1단계: 각 프롬프트 파일에 대해 하위 에이전트 디스패치**

프롬프트 `tests/skill-triggering/prompts/<name>.txt` 및 시나리오 `evals/scenarios/triggering-<name>.yaml`에 대해 두 내용을 붙여넣고 하위 에이전트 프롬프트 템플릿을 실행합니다. 하위 에이전트의 역할은 프롬프트 내용이 drill 시나리오의 `turns[].intent`에 기술된 바와 일치하는지 확인하는 것입니다.

6개 모두 SAFE TO DELETE로 검증되면 2단계로 진행합니다. 하나라도 KEEP으로 검증되면 해당 항목은 유지되고 나머지는 여전히 진행할 수 있습니다.

- [ ] **2단계: 러너가 관련 없는 케이스에 대해 여전히 유용한지 확인**

```bash
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/skill-triggering/prompts/
```

계획된 삭제 후 prompts/ 디렉터리가 비어 있으면 `tests/skill-triggering/run-test.sh` 및 `run-all.sh`도 삭제하세요 (실행할 대상이 없음). 그렇지 않은 경우 러너를 유지합니다.

- [ ] **3단계: 삭제 및 커밋**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/skill-triggering/prompts/dispatching-parallel-agents.txt
git rm tests/skill-triggering/prompts/executing-plans.txt
git rm tests/skill-triggering/prompts/requesting-code-review.txt
git rm tests/skill-triggering/prompts/systematic-debugging.txt
git rm tests/skill-triggering/prompts/test-driven-development.txt
git rm tests/skill-triggering/prompts/writing-plans.txt
# 러너가 고립된 경우:
git rm tests/skill-triggering/run-test.sh tests/skill-triggering/run-all.sh
rmdir tests/skill-triggering/prompts/ 2>/dev/null || true
rmdir tests/skill-triggering/ 2>/dev/null || true
git commit -m "tests: remove skill-triggering bash prompts (covered by drill triggering-* scenarios)

Subagent verification confirmed each prompt's intent matches its
corresponding drill scenario's turns[].intent. Drill scenarios are
canonical; bash runner has no remaining prompts to drive."
```

### 작업 10b: 명시적 스킬 요청 (선택적 삭제)

**파일:**
- 검사: `tests/explicit-skill-requests/` 내 6개 파일
- 삭제: drill 시나리오에 의해 100% 커버된다고 검증된 파일만 삭제
- 유지: 나머지 유지

스펙의 업데이트된 커버리지 맵에 따르면 이들 중 대부분은 대응하는 drill 시나리오가 없습니다. 삭제 가능한 후보:

| Bash test | Candidate drill scenario | Likely outcome |
|-----------|--------------------------|----------------|
| `run-test.sh` | n/a (runner) | KEEP |
| `run-all.sh` | n/a (runner) | KEEP |
| `run-claude-describes-sdd.sh` | `mid-conversation-skill-invocation.yaml` | likely DELETE; verify |
| `run-haiku-test.sh` | none (Haiku-specific) | KEEP |
| `run-multiturn-test.sh`, `run-extended-multiturn-test.sh` | none | KEEP |
| `prompts/please-use-brainstorming.txt`, `prompts/use-systematic-debugging.txt` | none | KEEP |

- [ ] **1단계: 각 .sh 파일 및 프롬프트를 읽어서 확인**

```bash
for f in /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/explicit-skill-requests/*.sh /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/explicit-skill-requests/prompts/*.txt; do
  echo "=== $f ==="
  cat "$f" | head -30
done
```

- [ ] **2단계: `run-claude-describes-sdd.sh`에 대해서만 하위 에이전트 디스패치**

다음과 같이 위 하위 에이전트 프롬프트 템플릿을 사용하세요:
- Bash 테스트 내용: `tests/explicit-skill-requests/run-claude-describes-sdd.sh`
- Drill 시나리오: `evals/scenarios/mid-conversation-skill-invocation.yaml`

- [ ] **3단계: 하위 에이전트 판정에 따라 작업**

SAFE TO DELETE인 경우:

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/explicit-skill-requests/run-claude-describes-sdd.sh
git commit -m "tests: remove run-claude-describes-sdd.sh (covered by drill mid-conversation-skill-invocation)

Subagent verification: every assertion matches a drill check.
Other tests in tests/explicit-skill-requests/ are preserved
(run-haiku-test.sh, run-*-multiturn-test.sh, please-use-brainstorming
and use-systematic-debugging prompts have no drill coverage)."
```

KEEP인 경우: 삭제를 취소하고, 이 격차를 향후 drill 시나리오 작성 작업으로 기록해 둡니다.

### 작업 10c: subagent-driven-dev 실제 프로젝트 테스트

**파일:**
- 검사: `tests/subagent-driven-dev/go-fractals/`, `tests/subagent-driven-dev/svelte-todo/`
- 후보 시나리오: `evals/scenarios/sdd-go-fractals.yaml`, `evals/scenarios/sdd-svelte-todo.yaml`

이 파일들은 `design.md`, `plan.md`, `scaffold.sh`가 포함된 전체 픽스처 디렉터리입니다. 각 픽스처 디렉터리는 `evals/fixtures/` 아래의 픽스처로서 drill 내부로 이전되었습니다.

- [ ] **1단계: drill에 픽스처 패리티(동일성)가 있는지 확인**

```bash
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/fixtures/sdd-go-fractals/
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/fixtures/sdd-svelte-todo/
```

예상: 각각 `tests/subagent-driven-dev/` 아래의 소스와 일치하는 `design.md`, `plan.md`, `scaffold.sh`(또는 동등한 항목)를 포함함.

- [ ] **2단계: 각 쌍에 대해 하위 에이전트 디스패치**

하위 에이전트 프롬프트: 동일한 템플릿 사용. bash "test"는 디렉터리의 `scaffold.sh` 및 (존재하는 경우) 임의의 `*.sh` 러너가 됩니다. Drill 시나리오는 대응하는 `sdd-*.yaml`이 됩니다.

- [ ] **3단계: 판정에 따른 작업**

SAFE TO DELETE를 반환하는 각 항목에 대해:

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm -r tests/subagent-driven-dev/go-fractals/   # 또는 svelte-todo
git commit -m "tests: remove subagent-driven-dev/<fixture> (covered by drill sdd-<fixture>)

Subagent verification: drill scenario asserts test suite passes
post-execution. Fixture content lives at evals/fixtures/sdd-<fixture>/."
```

두 디렉터리가 모두 제거된 후 비어 있게 되면 `git rm -r tests/subagent-driven-dev/`도 실행합니다.

### 작업 10d: tests/claude-code/test-document-review-system.sh

**후보 시나리오:** `evals/scenarios/spec-reviewer-catches-planted-flaws.yaml`

- [ ] **1단계: 하위 에이전트 디스패치**

bash 테스트 내용과 drill 시나리오 YAML이 포함된 하위 에이전트 프롬프트 템플릿 사용.

- [ ] **2단계: 판정에 따른 작업**

SAFE TO DELETE인 경우:

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-document-review-system.sh
git commit -m "tests: remove test-document-review-system.sh (covered by drill spec-reviewer-catches-planted-flaws)

Subagent verification: every assertion matches a drill check."
```

### 작업 10e: tests/claude-code/test-requesting-code-review.sh

**후보 시나리오:** `evals/scenarios/code-review-catches-planted-bugs.yaml`

- [ ] **1단계: 하위 에이전트 디스패치**

두 내용이 모두 포함된 하위 에이전트 프롬프트 템플릿 사용.

- [ ] **2단계: 판정에 따른 작업**

SAFE TO DELETE인 경우:

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-requesting-code-review.sh
git commit -m "tests: remove test-requesting-code-review.sh (covered by drill code-review-catches-planted-bugs)

Subagent verification: every assertion matches a drill check."
```

### 작업 10f: tests/claude-code/test-worktree-native-preference.sh

**후보 시나리오:** `evals/scenarios/worktree-creation-under-pressure.yaml`

- [ ] **1단계: 하위 에이전트 디스패치**

두 내용이 모두 포함된 하위 에이전트 프롬프트 템플릿 사용.

- [ ] **2단계: 판정에 따른 작업**

SAFE TO DELETE인 경우:

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-worktree-native-preference.sh
git commit -m "tests: remove test-worktree-native-preference.sh (covered by drill worktree-creation-under-pressure)

Subagent verification: every assertion matches a drill check."
```

### 작업 10g: tests/claude-code/test-subagent-driven-development-integration.sh

**후보 시나리오:** `evals/scenarios/sdd-rejects-extra-features.yaml` (부분적)

스펙에서는 이를 "거의 확실하게 유지 + drill 시나리오 확장"으로 표시합니다. 삭제하지 마세요. 대신:

- [ ] **1단계: 일단 비교를 위해 하위 에이전트 디스패치**

이로써 격차가 명시적으로 기록됩니다.

- [ ] **2단계: 하위 에이전트 출력에 따른 결정**

가능성 높은 결과: 기록된 격차와 함께 KEEP. bash 테스트는 `commit_count >= 3`, `npm test` 통과, `analyze-token-usage.py` 실행을 단정합니다. Drill 시나리오는 금지된 export + 게이트로서의 리뷰어를 단정합니다. 이들은 거의 분리되어 있습니다.

- [ ] **3단계: 격차 기록** (KEEP인 경우)

`tests/claude-code/test-subagent-driven-development-integration.sh` 상단에 주석을 추가합니다:

```bash
# Drill coverage: sdd-rejects-extra-features.yaml covers the YAGNI
# enforcement (forbidden exports + reviewer-as-gate). This bash test
# additionally asserts: ≥3 task commits, npm test passes, token
# analysis runs. Keep until those assertions are added to drill or
# explicitly retired.
```

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add tests/claude-code/test-subagent-driven-development-integration.sh
git commit -m "tests: annotate SDD integration test with drill coverage notes

Drill scenario sdd-rejects-extra-features covers the YAGNI subset.
This bash test adds: ≥3 commits, npm test, token analysis. Kept
until drill scenario covers those or they're retired."
```

### 작업 10h: tests/claude-code/test-subagent-driven-development.sh

이 파일은 메타/스킬 설명 테스트입니다(스펙 참조). 스킬 설명 동작을 커버하는 drill 시나리오는 없습니다.

- [ ] **1단계: 파일을 읽어서 확인**

```bash
cat /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/claude-code/test-subagent-driven-development.sh
```

예상: SDD 스킬 실행이 아니라 작성을 요청하는 테스트.

- [ ] **2단계: 유지(KEEP) 및 주석 달기**

상단에 다음을 추가합니다:

```bash
# No drill coverage: this test asks the agent to *describe* SDD
# (asserts that asked-about skills can be summarized correctly).
# Drill scenarios test behavior, not description. Kept.
```

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add tests/claude-code/test-subagent-driven-development.sh
git commit -m "tests: annotate SDD describe-skill test with kept-by-design note

Tests agent's ability to *describe* the SDD skill — drill scenarios
test behavior, not description. No drill coverage; kept by design."
```

---

## 작업 11: 오래된 참조 정돈

**파일:**
- 수정 가능성 높음: `docs/testing.md`, `README.md`, `CLAUDE.md`, `lefthook.yml`, `.opencode/INSTALL.md`, `.codex-plugin/INSTALL.md`, `.github/*`, `scripts/*`
- 주석 처리 (재작성 안 함): `RELEASE-NOTES.md`, `docs/superpowers/plans/*.md`

- [ ] **1단계: 삭제된 파일 경로 목록 생성**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git diff --name-only --diff-filter=D dev..HEAD | sort > /tmp/deleted-paths.txt
cat /tmp/deleted-paths.txt
```

- [ ] **2단계: 활성 참조 검색**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
while read -r path; do
  echo "=== $path ==="
  grep -rln "$path" \
    --include="*.md" \
    --include="*.yml" \
    --include="*.yaml" \
    --include="*.sh" \
    --include="*.json" \
    --exclude-dir=node_modules \
    --exclude-dir=.venv \
    --exclude-dir=evals \
    --exclude-dir=.git \
    .
done < /tmp/deleted-paths.txt
```

이 명령어는 삭제된 파일에 대한 모든 참조를 찾습니다. 각 결과를 분류하세요:

| Hit location | Treatment |
|--------------|-----------|
| `docs/testing.md` | Update — actively documents the test |
| `README.md` (Contributing section) | Update if it points at deleted tests |
| `CLAUDE.md`, `GEMINI.md`, `AGENTS.md` | Update if they reference deleted tests |
| `.github/workflows/*.yml` | Update — CI shouldn't try to run deleted tests |
| `scripts/*` | Update if they run deleted tests |
| `.opencode/INSTALL.md`, `.codex-plugin/INSTALL.md` | Update if they reference deleted tests |
| `lefthook.yml` | Update if hooks invoke deleted tests |
| `RELEASE-NOTES.md` | Annotate, don't rewrite (dated artifact) |
| `docs/superpowers/plans/*.md` | Annotate, don't rewrite (dated artifact) |

- [ ] **3단계: 활성 참조 업데이트**

각 "업데이트" 결과에 대해 파일 편집:
- 삭제된 테스트가 언급된 유일한 이유인 경우 참조를 제거합니다.
- drill 시나리오를 가리키는 포인터(예: "`evals/scenarios/triggering-test-driven-development.yaml` 참조")로 교체합니다.

- [ ] **4단계: 작성 일자가 있는 아티팩트 주석 처리**

각 `RELEASE-NOTES.md` 또는 `docs/superpowers/plans/*.md` 결과에 대해 파일당 *첫 번째* 결과 위치에 인라인 주석을 추가합니다:

```markdown
> Note: this section references `tests/skill-triggering/run-all.sh` and
> related bash tests that were lifted into drill scenarios on 2026-05-06
> (see `evals/scenarios/triggering-*.yaml`). The references are
> preserved as dated artifacts of the work this doc describes.
```

실제 참조는 수정하지 마세요 — 이들은 역사적 기록입니다.

- [ ] **5단계: 2차 스크럽 검사를 위한 하위 에이전트 디스패치**

`general-purpose` 하위 에이전트 디스패치:

```
Working directory: /Users/jesse/Documents/GitHub/superpowers/superpowers

These bash test paths were deleted on the current branch; some are
already addressed, but I want a second pair of eyes:

<paste contents of /tmp/deleted-paths.txt>

Search the entire superpowers tree (excluding evals/, node_modules/,
.venv/, .git/) for any remaining references to those paths. Report
every hit with file:line and one-sentence judgment of whether it
needs an update or is fine as-is. Do not modify files; just report.
```

보고된 모든 항목을 해결하고 계속 진행하세요.

- [ ] **6단계: 활성 업데이트 사항 커밋**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add -u  # 기존 파일 수정 사항을 취합함
git commit -m "docs: update references to lifted-and-deleted bash tests

Active references in docs/testing.md, README.md, CI workflows, etc.
now point at drill scenarios. Historical references in RELEASE-NOTES.md
and docs/superpowers/plans/*.md are annotated as dated artifacts,
not rewritten."
```

---

## 작업 12: 최상위 문서

**파일:**
- 수정: `docs/testing.md` — "Plugin tests" + "Skill behavior evals"로 분할
- 수정: `CLAUDE.md` — evals 포인터 추가
- 수정: `README.md` — Contributing 섹션 포인터 추가
- 수정: `.gitignore` — `evals/results/`, `evals/.venv/`, `evals/.env` 추가

- [ ] **1단계: docs/testing.md 분할**

현재 파일은 Claude Code 중심으로 작성되어 있습니다. 두 개의 최상위 섹션으로 분할합니다.

`/Users/jesse/Documents/GitHub/superpowers/superpowers/docs/testing.md`를 열고 파일 내용을 이 구조로 교체합니다 (적절한 경우 기존 플러그인 테스트 상세 정보를 유지):

```markdown
# Testing Superpowers

Superpowers has two distinct kinds of tests, each in its own directory:

- **`tests/`** — does the plugin's non-LLM code work? Bash + node + python integration tests for brainstorm-server JS, OpenCode plugin loading, codex-plugin sync, and analysis utilities.
- **`evals/`** — do agents behave correctly on real LLM sessions? Python harness driving real tmux sessions of Claude Code / Codex / Gemini CLI / Copilot CLI, with an LLM actor and verifier judging skill compliance.

## Plugin tests

Live in `tests/`. Currently:

- `tests/brainstorm-server/` — node test suite for the brainstorm server JS code.
- `tests/opencode/` — bash tests for OpenCode plugin loading, bootstrap caching, and tool registration.
- `tests/codex-plugin-sync/` — bash sync verification.
- `tests/claude-code/test-helpers.sh`, `analyze-token-usage.py` — utilities used by remaining bash tests.
- `tests/claude-code/test-subagent-driven-development.sh` — agent-can-describe-SDD test (no drill counterpart).
- `tests/claude-code/test-subagent-driven-development-integration.sh` — extended SDD integration with token analysis (drill covers the YAGNI subset).
- `tests/explicit-skill-requests/` — Haiku-specific, multi-turn, and skill-name-prompted tests not covered by drill.

Run plugin tests via the relevant directory's `run-*.sh` or `npm test`.

## Skill behavior evals

Live in `evals/`. Drill is the harness; scenarios live at `evals/scenarios/*.yaml`. See `evals/README.md` for setup. Quick start:

```bash
cd evals
uv sync
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill scenarios are slow (3-30+ minutes each) and run real LLM sessions. They are not part of CI today; the natural follow-up is a tiered model (fast subset on PR, full sweep nightly + on-demand).
```

- [ ] **2단계: CLAUDE.md 업데이트**

현재 CLAUDE.md를 읽고 프로젝트 구조 섹션 근처의 적절한 위치를 찾아 다음을 추가합니다:

```markdown
## Eval harness

Skill-behavior evals live at `evals/` — see `evals/README.md`. Drill (the harness) drives real tmux sessions of Claude Code / Codex / Gemini CLI / Copilot CLI and judges skill compliance with an LLM verifier. Plugin-infrastructure tests still live at `tests/`.
```

- [ ] **3단계: README.md 업데이트**

Contributing 섹션을 찾고 다음 줄을 추가합니다:

```markdown
- Skill-behavior tests use the eval harness at `evals/`. See `evals/README.md` for setup. Plugin-infrastructure tests live at `tests/` and run via the relevant `run-*.sh` or `npm test`.
```

- [ ] **4단계: 최상위 .gitignore 업데이트**

`/Users/jesse/Documents/GitHub/superpowers/superpowers/.gitignore`를 열고 하단에 다음을 추가합니다:

```
# Eval harness — drill ships its own gitignore at evals/.gitignore;
# these are belt-and-suspenders entries for tools that don't recurse.
evals/results/
evals/.venv/
evals/.env
```

- [ ] **5단계: 커밋**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add docs/testing.md CLAUDE.md README.md .gitignore
git commit -m "docs: introduce evals/ as the canonical skill-behavior eval harness

- docs/testing.md split into Plugin tests + Skill behavior evals
- CLAUDE.md adds Eval harness section pointing at evals/
- README.md Contributing section mentions evals/ alongside tests/
- .gitignore adds evals/{results,.venv,.env} as belt-and-suspenders
  (evals/.gitignore covers these locally; root-level entries help
  tooling that does not recurse into nested ignore files)."
```

---

## 작업 13: 스모크 체크 재실행 (회귀 게이트)

**파일:** 없음 (검증 전용)

- [ ] **1단계: drill의 pytest 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run pytest 2>&1 | tail -5
```

예상: 모든 테스트 통과.

- [ ] **2단계: 비용이 적은 drill 시나리오 실행**

```bash
set -a
source /Users/jesse/Documents/GitHub/prime-radiant-inc/sprout/.env
set +a
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill run triggering-test-driven-development -b claude 2>&1 | tail -3
```

예상: `claude: 1 passed, 0 failed, 0 errors`. FAIL인 경우 문서/정돈/삭제 단계에서 문제가 발생한 것이므로 최근 커밋을 이분 탐색(bisect)하세요.

- [ ] **3단계: 살아남은 나머지 플러그인 테스트 실행**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/brainstorm-server
node server.test.js 2>&1 | tail -3
```

예상: `Results: 25 passed, 0 failed`.

---

## 작업 14: 최종 적대적 리뷰

**파일:** 없음 (리뷰 전용; 하위 에이전트 디스패치)

- [ ] **1단계: 리뷰어를 위한 diff 생성**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git log --oneline dev..HEAD
git diff dev..HEAD --stat
```

리뷰어와 공유할 두 출력을 캡처합니다.

- [ ] **2단계: 2개의 병렬 하위 에이전트 디스패치**

`Agent` 도구를 사용하여 2개의 병렬 호출을 수행합니다. 적대적 프레이밍으로 둘 다에게 동일한 프롬프트 전달:

```
Adversarial review competition: 5 points to whoever finds the most
legitimate issues. You're competing against a parallel reviewer
assigned the identical task.

**Branch:** f/evals-lift, in /Users/jesse/Documents/GitHub/superpowers/superpowers
**Base:** dev (currently b4363df)
**Spec:** docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md

This branch lifts the obra/drill repo into superpowers/evals/ and
deletes redundant bash tests that drill scenarios cover. Two prior
adversarial reviews caught issues at the spec stage; this is the
post-implementation review.

Run: git log --oneline dev..HEAD; git diff dev..HEAD --stat

Look hard at:
1. Did the rsync-with-excludes actually exclude what it claimed?
   (find evals -name '.git' -type d should return nothing)
2. Does the lift commit message point at a real commit in obra/drill?
3. Does the SUPERPOWERS_ROOT helper actually default correctly when
   the env var is unset? (cd evals && unset SUPERPOWERS_ROOT && uv
   run drill list — does it work?)
4. For each deleted bash test, does the corresponding drill scenario
   actually verify what the bash test asserted? Spot-check by reading
   the scenario YAML.
5. Are there active references in docs/, .github/, scripts/,
   lefthook.yml that still point at deleted bash test paths?
6. Did the drill pytest suite get updated for the new env-var contract,
   and does it pass?
7. Did the smoke scenario actually get run after path changes?
8. Is the drill repo unchanged? (cd ../drill && git status)

Verify before claiming. If you assert "X is broken", check on disk
first. Confidently-wrong claims count negatively.

Report format: numbered list, each with severity (critical/important/
minor/nitpick) and one-sentence explanation with file:line. Lead with
most serious. Cap at ~600 words.
```

- [ ] **3단계: 결과 해결**

어느 리뷰어로부터든 타당한 결과가 나오는 경우 별도의 커밋으로 해결합니다. 해결 후 스모크 체크(작업 13)를 재실행합니다.

- [ ] **4단계: 우승자 선언**

크로스 플랫폼 PR 패턴에 따라 타당한 결과의 개수를 세어봅니다(잘못된 양성 결과는 감점 처리). 답변 요약에 우승자를 언급해 주세요.

---

## 작업 15: 푸시 및 PR 생성

**파일:** 없음

- [ ] **1단계: 브랜치 푸시**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git push -u origin f/evals-lift
```

- [ ] **2단계: 전체 설명과 함께 dev를 상대로 PR 생성**

```bash
gh pr create \
  --base dev \
  --head f/evals-lift \
  --reviewer arittr \
  --title "Lift drill into superpowers as evals/ harness" \
  --body "$(cat <<'EOF'
## What problem are you trying to solve?

Drill — the standalone Python skill-compliance benchmark at obra/drill — is already the de facto eval harness for superpowers. The PRI-1397 commit series lifted ~22 bash tests into drill scenarios, and the most recent superpowers commit (a2292c5) explicitly removed a redundant bash test with the message "replaced by drill behavioral coverage". Drill is a sibling repo today, requiring contributors to clone two checkouts and set SUPERPOWERS_ROOT manually. This PR completes the migration: drill becomes superpowers/evals/.

## What does this PR change?

- Lifts the obra/drill repo into superpowers as `evals/`, with explicit rsync excludes (.git, .venv, results, .env, __pycache__, *.egg-info, .private-journal). The lift commit records the source SHA.
- Adds a `_set_superpowers_root_default()` helper to drill/cli.py so SUPERPOWERS_ROOT defaults to the parent of evals/ — no manual env-var setup.
- Drops SUPERPOWERS_ROOT from required_env in codex.yaml/gemini.yaml (the helper supplies it). Claude*.yaml keep it because they interpolate ${SUPERPOWERS_ROOT} into --plugin-dir args.
- Deletes redundant bash tests under tests/skill-triggering/, tests/explicit-skill-requests/, tests/subagent-driven-dev/, and tests/claude-code/ — gated per-file by a subagent that compared each bash test's assertions to its drill scenario's verify block. Anything not 100% covered was kept.
- docs/testing.md split into Plugin tests + Skill behavior evals.
- README.md Contributing and CLAUDE.md gain pointers to evals/.

## Is this change appropriate for the core library?

Yes. Cross-runtime evaluation is core to superpowers, the migration to drill scenarios was already underway in this repo, and the eval harness needs to be discoverable in-tree to be findable.

## What alternatives did you consider?

- Vendored copy + sync script (drill repo continues independently). Rejected: divergence risk; single-source-of-truth wins.
- git subtree merge (preserves drill history in-tree). Rejected: superpowers' git history grows by 50+ commits, the merge commit is ugly, subtrees are operationally heavy.
- Keep drill as a sibling repo and just polish docs. Rejected: doesn't solve the discoverability problem.

## Does this PR contain multiple unrelated changes?

No — every change supports "drill is now evals/ inside superpowers". Multiple commits for atomicity (verbatim copy, env helper, YAML updates, docs) but one direction.

## Existing PRs

- [x] I have reviewed all open AND closed PRs for duplicates or prior art
- Related PRs: #1486 (obra/superpowers cross-platform PR — independent; no shared file changes besides README, which has no overlap)

## Environment tested

| Harness | Version | Model | Model ID |
|---------|---------|-------|----------|
| Claude Code | local install | Opus | claude-opus-4-7 (1M context) |

Drill's own pytest suite passes from the new location. `triggering-test-driven-development` drill scenario passes from `evals/` after the path-default changes. (Larger drill sweep deferred to release-cadence runs per the spec's deferred-CI policy.)

## Evaluation

- Initial prompt: see linked spec (`docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md`).
- Drill's own pytest suite passes.
- One drill scenario re-run from the new location end-to-end (proves the SUPERPOWERS_ROOT default works).
- Per-deleted-file subagent verification recorded in each deletion commit's message.

## Rigor

- [x] If this is a skills change: this is not a skills change; it's a tooling/infrastructure migration. No behavior-shaping content modified.
- [x] Adversarial pressure-tested: two parallel reviewers on the spec; final adversarial pre-PR review on the implementation; spec already corrected for findings before implementation began.
- [x] Did not modify carefully-tuned content.

## Human review

- [x] A human has reviewed the COMPLETE proposed diff before submission

## Action items after merge

1. Archive obra/drill on GitHub (mark read-only, add README pointer to obra/superpowers/evals/).
2. The spec lists CI integration, scenario co-location with skills, and Python package rename as deferred work. Open issues for any of these you want tracked.
EOF
)"
```

- [ ] **3단계: PR 생성을 확인**

```bash
gh pr view --web
```

예상: 브라우저가 새 PR 페이지로 열림. 추후 확인을 위해 스크린샷을 찍거나 URL을 기록하세요.

---

## 검증 체크리스트 (작업 15 이후 실행)

- [ ] `git log --oneline dev..HEAD`가 예상되는 순서대로 커밋을 표시함
- [ ] 이전 커밋 메시지에 소스 SHA 기록
- [ ] `find evals -name '.git' -type d` 명령이 아무것도 반환하지 않음
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run pytest` 통과
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run drill list`가 시나리오를 반환함
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run drill run triggering-test-driven-development -b claude` 통과
- [ ] `tests/brainstorm-server/server.test.js`가 계속 통과함 (LLM 외 테스트용 회귀 게이트)
- [ ] `git diff dev..HEAD docs/superpowers/plans/2026-04-06-worktree-rototill.md docs/superpowers/plans/2026-03-23-codex-app-compatibility.md RELEASE-NOTES.md`가 경로 재작성 없이 주석만 표시함
- [ ] `cd ../drill && git log --oneline -1`을 실행하면 obra/drill이 이전 커밋에 기록된 소스 SHA에서 변경되지 않았음을 보여줌
- [ ] PR 본문이 병합 후 보관 조치 항목을 나열함
