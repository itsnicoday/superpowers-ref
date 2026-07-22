# Antigravity CLI (`agy`) 도구 맵핑 (Tool Mapping)

스킬은 동작 기반 언어("서브에이전트 디스패치", "할 일 생성", "파일 읽기")로 설명되어 있습니다. Antigravity CLI (`agy`) 환경에서는 이러한 표현이 아래 도구들로 매핑되어 처리됩니다.

| 스킬이 요청하는 동작 | Antigravity CLI 매핑 도구 |
|----------------------|----------------------|
| 서브에이전트 디스패치 (`Subagent (general-purpose):` 템플릿) | 내장 `TypeName`을 사용하는 `invoke_subagent` — 전체 기능 작업용 `self`, 읽기 전용 작업용 `research` ([Subagent support](#subagent-support) 참조) |
| 작업 추적 ("할 일 목록 생성", "완료 표시") | **Task 아티팩트** 사용 — `IsArtifact: true` 및 `ArtifactType: "task"`가 설정된 `write_to_file` ([Task tracking](#task-tracking) 참조). 백그라운드 프로세스를 관리하는 `manage_task`가 **아님**. |

## 작업 추적 (Task tracking)

Antigravity에는 **todo 전용 도구가 없습니다** (`manage_task`는 백그라운드 프로세스 — `list`/`kill`/`status`/`send_input` — 를 관리하는 도구이며 체크리스트가 *아닙니다*). 스킬에서 todo 목록 생성이나 작업 추적을 지시할 때는 **task 아티팩트(task artifact)**를 유지하세요: 작업 진행 상황에 따라 `replace_file_content` / `multi_replace_file_content`로 편집되는, `write_to_file` (`IsArtifact: true`, `ArtifactMetadata.ArtifactType: "task"`)로 저장된 마크다운 체크리스트입니다.

다단계 작업을 시작할 때, 계획의 모든 단계를 나열한 task 아티팩트를 생성하세요. 각 단계를 완료할 때마다 아티팩트를 편집하여 완료 표시 (`- [x]`)를 하세요. 계획이 변경되면 체크리스트를 업데이트하세요. 체크리스트를 최신 상태로 유지하세요 — 이것은 남은 작업에 대한 진실의 단일 출처(source of truth)입니다. 대화가 길어지면 각 단계를 시작하기 전에 이 문서를 다시 읽으세요.
