# Claude Code Skills 테스트

Claude Code CLI를 사용하여 superpowers 스킬을 검증하는 자동화 테스트 수트입니다.

## 개요 (Overview)

본 테스트 수트는 스킬이 올바르게 로드되고 Claude가 의도한 대로 스킬을 따르는지 검증합니다. 테스트는 헤드리스 모드(`claude -p`)로 Claude Code를 호출하고 동작을 확인합니다.

## 요구 사항 (Requirements)

- Claude Code CLI 설치 및 PATH 등록 (`claude --version` 동작 확인)
- 로컬 superpowers 플러그인 설치 (설치는 메인 README 참조)

## 테스트 실행 (Running Tests)

### 모든 빠른 테스트 실행 (권장):
```bash
./run-skill-tests.sh
```

### 통합 테스트 실행 (느림, 10-30분 소요):
```bash
./run-skill-tests.sh --integration
```

### 특정 테스트 실행:
```bash
./run-skill-tests.sh --test test-subagent-driven-development.sh
```

### 상세 출력(verbose) 포함 실행:
```bash
./run-skill-tests.sh --verbose
```

### 사용자 지정 타임아웃 설정:
```bash
./run-skill-tests.sh --timeout 1800  # 통합 테스트를 위한 30분 설정
```

## 테스트 구조 (Test Structure)

### test-helpers.sh
스킬 테스트를 위한 공통 함수:
- `run_claude "prompt" [timeout]` - 프롬프트와 함께 Claude 실행
- `assert_contains output pattern name` - 패턴 존재 여부 검증
- `assert_not_contains output pattern name` - 패턴 부재 여부 검증
- `assert_count output pattern count name` - 정확한 개수 검증
- `assert_order output pattern_a pattern_b name` - 순서 검증
- `create_test_project` - 임시 테스트 디렉터리 생성
- `create_test_plan project_dir` - 샘플 계획 파일 생성

### 테스트 파일 (Test Files)

각 테스트 파일의 역할:
1. `test-helpers.sh` 수용(source)
2. 특정 프롬프트로 Claude Code 실행
3. 어설션(assertion)을 통해 예상 동작 검증
4. 성공 시 0, 실패 시 0이 아닌 값 반환

## 테스트 예시 (Example Test)

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

echo "=== Test: My Skill ==="

# Ask Claude about the skill
output=$(run_claude "What does the my-skill skill do?" 30)

# Verify response
assert_contains "$output" "expected behavior" "Skill describes behavior"

echo "=== All tests passed ==="
```

## 현재 테스트 목록 (Current Tests)

### 빠른 테스트 (기본 실행)

#### test-subagent-driven-development.sh
스킬 내용 및 요구사항 검증 (~2분):
- 스킬 로딩 및 접근성
- 워크플로우 순서 (코드 품질 검사 전 사양 준수 검사)
- 문서화된 자체 리뷰 요구사항
- 문서화된 계획 읽기 효율성
- 문서화된 사양 준수 리뷰어의 회의적 평가
- 문서화된 리뷰 루프
- 문서화된 작업 컨텍스트 제공

### 통합 테스트 (--integration 플래그 사용)

#### test-subagent-driven-development-integration.sh
전체 워크플로우 실행 테스트 (~10-30분):
- Node.js 설정으로 실제 테스트 프로젝트 생성
- 2개 작업이 포함된 구현 계획 생성
- subagent-driven-development를 사용하여 계획 실행
- 실제 동작 검증:
  - 시작 시 한 번만 계획 읽기 (작업당 읽기 아님)
  - 서브에이전트 프롬프트에 전체 작업 텍스트 제공
  - 서브에이전트가 보고 전 자체 리뷰 수행
  - 코드 품질 검사 전에 사양 준수 리뷰 진행
  - 사양 리뷰어가 코드를 독립적으로 읽음
  - 동작하는 구현물 생성
  - 테스트 통과
  - 올바른 git 커밋 생성

**테스트 대상:**
- 워크플로우가 실제로 엔드투엔드로 동작하는지 여부
- 개선 사항이 실제로 적용되는지 여부
- 서브에이전트가 스킬을 올바르게 따르는지 여부
- 최종 코드가 정상 작동하고 테스트되는지 여부

#### test-worktree-native-preference.sh
using-git-worktrees 스킬에 대한 RED-GREEN-REFACTOR 검증 (~5분):
- RED: 1a 단계가 없는 스킬 — 에이전트가 `git worktree add`를 사용해야 함
- GREEN: 1a 단계가 있는 스킬 — 에이전트가 네이티브 EnterWorktree 도구를 사용해야 함
- PRESSURE: 기존 `.worktrees/`가 존재하는 긴급 프레이밍 상황에서 GREEN과 동일
- Drill 시나리오 `worktree-creation-under-pressure.yaml`은 PRESSURE 단계만 포함함

## 새로운 테스트 추가 방법 (Adding New Tests)

1. 새 테스트 파일 생성: `test-<skill-name>.sh`
2. test-helpers.sh 포함
3. `run_claude` 및 어설션을 사용하여 테스트 작성
4. `run-skill-tests.sh`의 테스트 목록에 추가
5. 실행 권한 부여: `chmod +x test-<skill-name>.sh`

## 타임아웃 고려 사항 (Timeout Considerations)

- 기본 타임아웃: 테스트당 5분
- Claude Code의 응답에 시간이 걸릴 수 있음
- 필요한 경우 `--timeout`으로 조정
- 긴 실행을 피하기 위해 테스트의 초점을 명확히 유지

## 실패한 테스트 디버깅 (Debugging Failed Tests)

`--verbose` 플래그를 사용하면 전체 Claude 출력을 확인할 수 있습니다:
```bash
./run-skill-tests.sh --verbose --test test-subagent-driven-development.sh
```

상세 옵션이 없으면 실패 시에만 출력이 표시됩니다.

## CI/CD 통합 (CI/CD Integration)

CI에서 실행하려면:
```bash
# CI 환경을 위한 명시적 타임아웃과 함께 실행
./run-skill-tests.sh --timeout 900

# 종료 코드 0 = 성공, 0이 아닌 값 = 실패
```

## 참고 사항 (Notes)

- 테스트는 전체 실행이 아닌 스킬 *지침*을 검증합니다.
- 전체 워크플로우 테스트는 매우 오래 걸릴 수 있습니다.
- 주요 스킬 요구사항을 검증하는 데 집중하세요.
- 테스트는 결정론적(deterministic)이어야 합니다.
- 구현 세부사항에 대한 테스트는 피하세요.
