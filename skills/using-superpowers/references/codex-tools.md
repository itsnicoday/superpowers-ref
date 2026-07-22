## 서브에이전트 디스패치에는 멀티 에이전트 지원이 필요합니다

Codex 설정 파일(`~/.codex/config.toml`)에 다음 내용을 추가하세요:

```toml
[features]
multi_agent = true
```

이렇게 하면 `dispatching-parallel-agents` 및 `subagent-driven-development`와 같은 스킬을 위한 `spawn_agent`, `wait_agent`, `close_agent` 기능이 활성화됩니다. subagent-driven-development 스킬을 사용할 때, 구현자(implementer) 및 검토자(reviewer) 서브에이전트가 모든 작업을 마친 후에는 항상 이를 종료(close)해야 합니다.

## 환경 감지 (Environment Detection)

워크트리를 생성하거나 브랜치를 완료하는 스킬은 진행하기 전에 읽기 전용 git 명령으로 실행 환경을 감지해야 합니다:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 이미 연결된 워크트리에 있음 (생성 건너뛰기)
- `BRANCH` 비어 있음 → detached HEAD 상태 (샌드박스에서 branch/push/PR 실행 불가)

각 스킬이 이 시그널들을 어떻게 활용하는지에 대해서는 `using-git-worktrees` Step 0 및 `finishing-a-development-branch` Step 1을 참조하세요.

## Codex App 작업 마무리 (Codex App Finishing)

샌드박스로 인해 브랜치 생성/푸시 연산이 차단될 때(외부 관리 워크트리의 detached HEAD 상태), 에이전트는 모든 작업을 커밋하고 앱의 네이티브 컨트롤을 사용하도록 사용자에게 알립니다:

- **"Create branch"** — 브랜치 이름을 정한 뒤 앱 UI를 통해 커밋/푸시/PR 진행
- **"Hand off to local"** — 사용자의 로컬 체크아웃으로 작업 이전

에이전트는 여전히 테스트를 실행하고, 파일을 스테이징하며, 사용자가 복사하여 사용할 수 있도록 제안된 브랜치 이름, 커밋 메시지, PR 설명을 출력할 수 있습니다.
