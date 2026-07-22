# OpenCode 지원 설계

**날짜:** 2025-11-22
**작성자:** Bot & Jesse
**상태:** 설계 완료, 구현 대기 중

## 개요

기존 Codex 구현과 핵심 기능을 공유하는 네이티브 OpenCode 플러그인 아키텍처를 사용하여 OpenCode.ai에 대한 전체 superpowers 지원을 추가합니다.

## 배경

OpenCode.ai는 Claude Code 및 Codex와 유사한 코딩 에이전트입니다. superpowers를 OpenCode로 이식하려는 이전 시도(PR #93, PR #116)는 파일 복사 방식을 사용했습니다. 이 설계는 다른 접근 방식을 취합니다. Codex 구현과 코드를 공유하면서 JavaScript/TypeScript 플러그인 시스템을 사용하여 네이티브 OpenCode 플러그인을 구축합니다.

### 플랫폼 간 주요 차이점

- **Claude Code**: 네이티브 Anthropic 플러그인 시스템 + 파일 기반 스킬
- **Codex**: 플러그인 시스템 없음 → 부트스트랩 마크다운 + CLI 스크립트
- **OpenCode**: 이벤트 훅 및 커스텀 도구 API를 갖춘 JavaScript/TypeScript 플러그인

### OpenCode의 에이전트 시스템

- **기본 에이전트**: Build (기본값, 전체 권한) 및 Plan (제한됨, 읽기 전용)
- **서브에이전트**: General (리서치, 검색, 다단계 작업)
- **호출**: 기본 에이전트에 의한 자동 디스패치 또는 수동 `@mention` 구문
- **설정**: `opencode.json` 또는 `~/.config/opencode/agent/` 내 커스텀 에이전트

## 아키텍처

### 상위 구조

1. **공유 핵심 모듈** (`lib/skills-core.js`)
   - 공통 스킬 검색 및 파싱 로직
   - Codex 및 OpenCode 구현 모두에서 사용

2. **플랫폼별 래퍼**
   - Codex: CLI 스크립트 (`.codex/superpowers-codex`)
   - OpenCode: 플러그인 모듈 (`.opencode/plugin/superpowers.js`)

3. **스킬 디렉터리**
   - Core: `~/.config/opencode/superpowers/skills/` (또는 설치된 위치)
   - Personal: `~/.config/opencode/skills/` (코어 스킬을 오버라이드/shadowing)

### 코드 재사용 전략

`.codex/superpowers-codex`에서 공통 기능을 추출하여 공유 모듈로 이동:

```javascript
// lib/skills-core.js
module.exports = {
  extractFrontmatter(filePath),      // Parse name + description from YAML
  findSkillsInDir(dir, maxDepth),    // Recursive SKILL.md discovery
  findAllSkills(dirs),                // Scan multiple directories
  resolveSkillPath(skillName, dirs), // Handle shadowing (personal > core)
  checkForUpdates(repoDir)           // Git fetch/status check
};
```

### 스킬 Frontmatter 형식

현재 형식 (`when_to_use` 필드 없음):

```yaml
---
name: skill-name
description: Use when [condition] - [what it does]; [additional context]
---
```

## OpenCode 플러그인 구현

### 커스텀 도구

**도구 1: `use_skill`**

특정 스킬의 내용을 대화에 로드합니다 (Claude의 Skill 도구와 동일).

```javascript
{
  name: 'use_skill',
  description: 'Load and read a specific skill to guide your work',
  schema: z.object({
    skill_name: z.string().describe('Name of skill (e.g., "superpowers:brainstorming")')
  }),
  execute: async ({ skill_name }) => {
    const { skillPath, content, frontmatter } = resolveAndReadSkill(skill_name);
    const skillDir = path.dirname(skillPath);

    return `# ${frontmatter.name}
# ${frontmatter.description}
# Supporting tools and docs are in ${skillDir}
# ============================================

${content}`;
  }
}
```

**도구 2: `find_skills`**

메타데이터와 함께 사용 가능한 모든 스킬을 목록화합니다.

```javascript
{
  name: 'find_skills',
  description: 'List all available skills',
  schema: z.object({}),
  execute: async () => {
    const skills = discoverAllSkills();
    return skills.map(s =>
      `${s.namespace}:${s.name}
  ${s.description}
  Directory: ${s.directory}
`).join('\n');
  }
}
```

### 세션 시작 훅

새 세션이 시작될 때 (`session.started` 이벤트):

1. **using-superpowers 내용 주입**
   - using-superpowers 스킬의 전체 내용
   - 필수 워크플로 정립

2. **find_skills 자동 실행**
   - 사용 가능한 전체 스킬 목록을 미라 보여줌
   - 각 스킬 디렉터리 경로 포함

3. **도구 매핑 지침 주입**
   ```markdown
   **Tool Mapping for OpenCode:**
   When skills reference tools you don't have, substitute:
   - `TodoWrite` → `update_plan`
   - `Task` with subagents → Use OpenCode subagent system (@mention)
   - `Skill` tool → `use_skill` custom tool
   - Read, Write, Edit, Bash → Your native equivalents

   **Skill directories contain:**
   - Supporting scripts (run with bash)
   - Additional documentation (read with read tool)
   - Utilities specific to that skill
   ```

4. **업데이트 확인** (비동기/non-blocking)
   - 타임아웃이 설정된 빠른 git fetch
   - 업데이트가 있는 경우 알림

### 플러그인 구조

```javascript
// .opencode/plugin/superpowers.js
const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const { z } = require('zod');

export const SuperpowersPlugin = async ({ client, directory, $ }) => {
  const superpowersDir = path.join(process.env.HOME, '.config/opencode/superpowers');
  const personalDir = path.join(process.env.HOME, '.config/opencode/skills');

  return {
    'session.started': async () => {
      const usingSuperpowers = await readSkill('using-superpowers');
      const skillsList = await findAllSkills();
      const toolMapping = getToolMappingInstructions();

      return {
        context: `${usingSuperpowers}\n\n${skillsList}\n\n${toolMapping}`
      };
    },

    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill',
        schema: z.object({
          skill_name: z.string()
        }),
        execute: async ({ skill_name }) => {
          // Implementation using skillsCore
        }
      },
      {
        name: 'find_skills',
        description: 'List all available skills',
        schema: z.object({}),
        execute: async () => {
          // Implementation using skillsCore
        }
      }
    ]
  };
};
```

## 파일 구조

```
superpowers/
├── lib/
│   └── skills-core.js           # NEW: Shared skill logic
├── .codex/
│   ├── superpowers-codex        # UPDATED: Use skills-core
│   ├── superpowers-bootstrap.md
│   └── INSTALL.md
├── .opencode/
│   ├── plugin/
│   │   └── superpowers.js       # NEW: OpenCode plugin
│   └── INSTALL.md               # NEW: Installation guide
└── skills/                       # Unchanged
```

## 구현 계획

### Phase 1: 공유 Core 리팩터링

1. `lib/skills-core.js` 생성
   - `.codex/superpowers-codex`에서 frontmatter 파싱 추출
   - 스킬 검색 로직 추출
   - 경로 처리 로직(shadowing 포함) 추출
   - `name` 및 `description`만 사용하도록 업데이트 (`when_to_use` 없음)

2. 공유 Core를 사용하도록 `.codex/superpowers-codex` 업데이트
   - `../lib/skills-core.js`에서 가져오기
   - 중복 코드 제거
   - CLI 래퍼 로직 유지

3. Codex 구현 정상 동작 여부 테스트
   - bootstrap 명령어 검증
   - use-skill 명령어 검증
   - find-skills 명령어 검증

### Phase 2: OpenCode 플러그인 구축

1. `.opencode/plugin/superpowers.js` 생성
   - `../../lib/skills-core.js`에서 공유 코어 가져오기
   - 플러그인 함수 구현
   - 커스텀 도구 정의 (use_skill, find_skills)
   - session.started 훅 구현

2. `.opencode/INSTALL.md` 생성
   - 설치 안내
   - 디렉터리 설정
   - 환경 설정 가이드

3. OpenCode 구현 테스트
   - 세션 시작 부트스트랩 검증
   - use_skill 도구 작동 검증
   - find_skills 도구 작동 검증
   - 스킬 디렉터리 접근 가능 여부 검증

### Phase 3: 문서화 및 다듬기

1. OpenCode 지원 내용으로 README 업데이트
2. 메인 문서에 OpenCode 설치 가이드 추가
3. RELEASE-NOTES 업데이트
4. Codex와 OpenCode 모두 정상 작동하는지 테스트

## 다음 단계

1. **격리된 작업 공간 생성** (git worktrees 사용)
   - 브랜치: `feature/opencode-support`

2. **가능한 경우 TDD 준수**
   - 공유 코어 함수 테스트
   - 스킬 검색 및 파싱 테스트
   - 두 플랫폼 간 통합 테스트

3. **점진적 구현**
   - Phase 1: 공유 코어 리팩터링 + Codex 업데이트
   - 계속 진행하기 전에 Codex가 여전히 작동하는지 확인
   - Phase 2: OpenCode 플러그인 구축
   - Phase 3: 문서화 및 마무리

4. **테스트 전략**
   - 실제 OpenCode 설치 환경에서 수동 테스트
   - 스킬 로딩, 디렉터리, 스크립트 동작 검증
   - Codex와 OpenCode 나란히 테스트
   - 도구 매핑이 올바르게 작동하는지 확인

5. **PR 및 병합**
   - 전체 구현 완료 후 PR 생성
   - 깨끗한 환경에서 테스트
   - main에 병합

## 기대 효과

- **코드 재사용**: 스킬 검색/파싱을 위한 단일 진실 출처(Single source of truth)
- **유지보수성**: 버그 수정이 두 플랫폼 모두에 적용됨
- **확장성**: 향후 플랫폼(Cursor, Windsurf 등) 추가 용이
- **네이티브 통합**: OpenCode의 플러그인 시스템을 적절하게 활용
- **일관성**: 모든 플랫폼에서 동일한 스킬 경험 제공
