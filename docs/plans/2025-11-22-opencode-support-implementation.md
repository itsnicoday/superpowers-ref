# OpenCode 지원 구현 계획

> **에이전트 작업자 참고:** 필수 서브 스킬: 이 계획을 작업별로 구현하려면 `superpowers:executing-plans`를 사용하세요.

**목표:** 기존 Codex 구현과 핵심 기능을 공유하는 네이티브 JavaScript 플러그인으로 OpenCode.ai에 대한 전체 superpowers 지원을 추가합니다.

**아키텍처:** 공통 스킬 검색/파싱 로직을 `lib/skills-core.js`로 추출하고, Codex가 이를 사용하도록 리팩터링한 다음, 커스텀 도구 및 세션 훅이 포함된 네이티브 플러그인 API를 사용하여 OpenCode 플러그인을 구축합니다.

**기술 스택:** Node.js, JavaScript, OpenCode Plugin API, Git worktrees

---

## Phase 1: 공유 핵심 모듈 생성

### Task 1: Frontmatter 파싱 추출

**파일:**
- 생성: `lib/skills-core.js`
- 참조: `.codex/superpowers-codex` (lines 40-74)

**Step 1: extractFrontmatter 함수가 포함된 lib/skills-core.js 생성**

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

/**
 * Extract YAML frontmatter from a skill file.
 * Current format:
 * ---
 * name: skill-name
 * description: Use when [condition] - [what it does]
 * ---
 *
 * @param {string} filePath - Path to SKILL.md file
 * @returns {{name: string, description: string}}
 */
function extractFrontmatter(filePath) {
    try {
        const content = fs.readFileSync(filePath, 'utf8');
        const lines = content.split('\n');

        let inFrontmatter = false;
        let name = '';
        let description = '';

        for (const line of lines) {
            if (line.trim() === '---') {
                if (inFrontmatter) break;
                inFrontmatter = true;
                continue;
            }

            if (inFrontmatter) {
                const match = line.match(/^(\w+):\s*(.*)$/);
                if (match) {
                    const [, key, value] = match;
                    switch (key) {
                        case 'name':
                            name = value.trim();
                            break;
                        case 'description':
                            description = value.trim();
                            break;
                    }
                }
            }
        }

        return { name, description };
    } catch (error) {
        return { name: '', description: '' };
    }
}

module.exports = {
    extractFrontmatter
};
```

**Step 2: 파일 생성 확인**

실행: `ls -l lib/skills-core.js`
예상 결과: 파일이 존재함

**Step 3: 커밋**

```bash
git add lib/skills-core.js
git commit -m "feat: create shared skills core module with frontmatter parser"
```

---

### Task 2: 스킬 검색 로직 추출

**파일:**
- 수정: `lib/skills-core.js`
- 참조: `.codex/superpowers-codex` (lines 97-136)

**Step 1: skills-core.js에 findSkillsInDir 함수 추가**

`module.exports` 앞에 추가:

```javascript
/**
 * Find all SKILL.md files in a directory recursively.
 *
 * @param {string} dir - Directory to search
 * @param {string} sourceType - 'personal' or 'superpowers' for namespacing
 * @param {number} maxDepth - Maximum recursion depth (default: 3)
 * @returns {Array<{path: string, name: string, description: string, sourceType: string}>}
 */
function findSkillsInDir(dir, sourceType, maxDepth = 3) {
    const skills = [];

    if (!fs.existsSync(dir)) return skills;

    function recurse(currentDir, depth) {
        if (depth > maxDepth) return;

        const entries = fs.readdirSync(currentDir, { withFileTypes: true });

        for (const entry of entries) {
            const fullPath = path.join(currentDir, entry.name);

            if (entry.isDirectory()) {
                // Check for SKILL.md in this directory
                const skillFile = path.join(fullPath, 'SKILL.md');
                if (fs.existsSync(skillFile)) {
                    const { name, description } = extractFrontmatter(skillFile);
                    skills.push({
                        path: fullPath,
                        skillFile: skillFile,
                        name: name || entry.name,
                        description: description || '',
                        sourceType: sourceType
                    });
                }

                // Recurse into subdirectories
                recurse(fullPath, depth + 1);
            }
        }
    }

    recurse(dir, 0);
    return skills;
}
```

**Step 2: module.exports 업데이트**

내보내기 줄을 다음으로 교체:

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir
};
```

**Step 3: 구문 검증**

실행: `node -c lib/skills-core.js`
예상 결과: 출력 없음 (성공)

**Step 4: 커밋**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill discovery function to core module"
```

---

### Task 3: 스킬 확인 로직 추출

**파일:**
- 수정: `lib/skills-core.js`
- 참조: `.codex/superpowers-codex` (lines 212-280)

**Step 1: resolveSkillPath 함수 추가**

`module.exports` 앞에 추가:

```javascript
/**
 * Resolve a skill name to its file path, handling shadowing
 * (personal skills override superpowers skills).
 *
 * @param {string} skillName - Name like "superpowers:brainstorming" or "my-skill"
 * @param {string} superpowersDir - Path to superpowers skills directory
 * @param {string} personalDir - Path to personal skills directory
 * @returns {{skillFile: string, sourceType: string, skillPath: string} | null}
 */
function resolveSkillPath(skillName, superpowersDir, personalDir) {
    // Strip superpowers: prefix if present
    const forceSuperpowers = skillName.startsWith('superpowers:');
    const actualSkillName = forceSuperpowers ? skillName.replace(/^superpowers:/, '') : skillName;

    // Try personal skills first (unless explicitly superpowers:)
    if (!forceSuperpowers && personalDir) {
        const personalPath = path.join(personalDir, actualSkillName);
        const personalSkillFile = path.join(personalPath, 'SKILL.md');
        if (fs.existsSync(personalSkillFile)) {
            return {
                skillFile: personalSkillFile,
                sourceType: 'personal',
                skillPath: actualSkillName
            };
        }
    }

    // Try superpowers skills
    if (superpowersDir) {
        const superpowersPath = path.join(superpowersDir, actualSkillName);
        const superpowersSkillFile = path.join(superpowersPath, 'SKILL.md');
        if (fs.existsSync(superpowersSkillFile)) {
            return {
                skillFile: superpowersSkillFile,
                sourceType: 'superpowers',
                skillPath: actualSkillName
            };
        }
    }

    return null;
}
```

**Step 2: module.exports 업데이트**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath
};
```

**Step 3: 구문 검증**

실행: `node -c lib/skills-core.js`
예상 결과: 출력 없음

**Step 4: 커밋**

```bash
git add lib/skills-core.js
git commit -m "feat: add skill path resolution with shadowing support"
```

---

### Task 4: 업데이트 확인 로직 추출

**파일:**
- 수정: `lib/skills-core.js`
- 참조: `.codex/superpowers-codex` (lines 16-38)

**Step 1: checkForUpdates 함수 추가**

상단 `require`문 뒤에 추가:

```javascript
const { execSync } = require('child_process');
```

`module.exports` 앞에 추가:

```javascript
/**
 * Check if a git repository has updates available.
 *
 * @param {string} repoDir - Path to git repository
 * @returns {boolean} - True if updates are available
 */
function checkForUpdates(repoDir) {
    try {
        // Quick check with 3 second timeout to avoid delays if network is down
        const output = execSync('git fetch origin && git status --porcelain=v1 --branch', {
            cwd: repoDir,
            timeout: 3000,
            encoding: 'utf8',
            stdio: 'pipe'
        });

        // Parse git status output to see if we're behind
        const statusLines = output.split('\n');
        for (const line of statusLines) {
            if (line.startsWith('## ') && line.includes('[behind ')) {
                return true; // We're behind remote
            }
        }
        return false; // Up to date
    } catch (error) {
        // Network down, git error, timeout, etc. - don't block bootstrap
        return false;
    }
}
```

**Step 2: module.exports 업데이트**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath,
    checkForUpdates
};
```

**Step 3: 구문 검증**

실행: `node -c lib/skills-core.js`
예상 결과: 출력 없음

**Step 4: 커밋**

```bash
git add lib/skills-core.js
git commit -m "feat: add git update checking to core module"
```

---

## Phase 2: 공유 Core를 사용하도록 Codex 리팩터링

### Task 5: 공유 Core를 가져오도록 Codex 업데이트

**파일:**
- 수정: `.codex/superpowers-codex` (상단에 import 추가)

**Step 1: import 문 추가**

파일 상단에 기존 `require` 구문 뒤(약 6행)에 추가:

```javascript
const skillsCore = require('../lib/skills-core');
```

**Step 2: 구문 검증**

실행: `node -c .codex/superpowers-codex`
예상 결과: 출력 없음

**Step 3: 커밋**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: import shared skills core in codex"
```

---

### Task 6: extractFrontmatter를 Core 버전으로 교체

**파일:**
- 수정: `.codex/superpowers-codex` (lines 40-74)

**Step 1: 로컬 extractFrontmatter 함수 제거**

40-74행(전체 extractFrontmatter 함수 정의)을 삭제합니다.

**Step 2: 모든 extractFrontmatter 호출 업데이트**

모든 `extractFrontmatter(` 호출을 `skillsCore.extractFrontmatter(`로 찾아 교체합니다.

영향을 받는 줄 약: 90, 310

**Step 3: 스크립트 정상 동작 확인**

실행: `.codex/superpowers-codex find-skills | head -20`
예상 결과: 스킬 목록을 보여줌

**Step 4: 커밋**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared extractFrontmatter in codex"
```

---

### Task 7: findSkillsInDir를 Core 버전으로 교체

**파일:**
- 수정: `.codex/superpowers-codex` (약 97-136행)

**Step 1: 로컬 findSkillsInDir 함수 제거**

전체 `findSkillsInDir` 함수 정의(약 97-136행)를 삭제합니다.

**Step 2: 모든 findSkillsInDir 호출 업데이트**

`findSkillsInDir(` 호출을 `skillsCore.findSkillsInDir(`로 교체합니다.

**Step 3: 스크립트 정상 동작 확인**

실행: `.codex/superpowers-codex find-skills | head -20`
예상 결과: 스킬 목록을 보여줌

**Step 4: 커밋**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared findSkillsInDir in codex"
```

---

### Task 8: checkForUpdates를 Core 버전으로 교체

**파일:**
- 수정: `.codex/superpowers-codex` (약 16-38행)

**Step 1: 로컬 checkForUpdates 함수 제거**

전체 `checkForUpdates` 함수 정의를 삭제합니다.

**Step 2: 모든 checkForUpdates 호출 업데이트**

`checkForUpdates(` 호출을 `skillsCore.checkForUpdates(`로 교체합니다.

**Step 3: 스크립트 정상 동작 확인**

실행: `.codex/superpowers-codex bootstrap | head -50`
예상 결과: 부트스트랩 내용을 보여줌

**Step 4: 커밋**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: use shared checkForUpdates in codex"
```

---

## Phase 3: OpenCode 플러그인 구축

### Task 9: OpenCode 플러그인 디렉터리 구조 생성

**파일:**
- 생성: `.opencode/plugin/superpowers.js`

**Step 1: 디렉터리 생성**

실행: `mkdir -p .opencode/plugin`

**Step 2: 기본 플러그인 파일 생성**

```javascript
#!/usr/bin/env node

/**
 * Superpowers plugin for OpenCode.ai
 *
 * Provides custom tools for loading and discovering skills,
 * with automatic bootstrap on session start.
 */

const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const os = require('os');

const homeDir = os.homedir();
const superpowersSkillsDir = path.join(homeDir, '.config/opencode/superpowers/skills');
const personalSkillsDir = path.join(homeDir, '.config/opencode/skills');

/**
 * OpenCode plugin entry point
 */
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // Custom tools and hooks will go here
  };
};
```

**Step 3: 파일 생성 확인**

실행: `ls -l .opencode/plugin/superpowers.js`
예상 결과: 파일이 존재함

**Step 4: 커밋**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: create opencode plugin scaffold"
```

---

### Task 10: use_skill 도구 구현

**파일:**
- 수정: `.opencode/plugin/superpowers.js`

**Step 1: use_skill 도구 구현 추가**

플러그인 return 문을 다음으로 교체:

```javascript
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  // Import zod for schema validation
  const { z } = await import('zod');

  return {
    tools: [
      {
        name: 'use_skill',
        description: 'Load and read a specific skill to guide your work. Skills contain proven workflows, mandatory processes, and expert techniques.',
        schema: z.object({
          skill_name: z.string().describe('Name of the skill to load (e.g., "superpowers:brainstorming" or "my-custom-skill")')
        }),
        execute: async ({ skill_name }) => {
          // Resolve skill path (handles shadowing: personal > superpowers)
          const resolved = skillsCore.resolveSkillPath(
            skill_name,
            superpowersSkillsDir,
            personalSkillsDir
          );

          if (!resolved) {
            return `Error: Skill "${skill_name}" not found.\n\nRun find_skills to see available skills.`;
          }

          // Read skill content
          const fullContent = fs.readFileSync(resolved.skillFile, 'utf8');
          const { name, description } = skillsCore.extractFrontmatter(resolved.skillFile);

          // Extract content after frontmatter
          const lines = fullContent.split('\n');
          let inFrontmatter = false;
          let frontmatterEnded = false;
          const contentLines = [];

          for (const line of lines) {
            if (line.trim() === '---') {
              if (inFrontmatter) {
                frontmatterEnded = true;
                continue;
              }
              inFrontmatter = true;
              continue;
            }

            if (frontmatterEnded || !inFrontmatter) {
              contentLines.push(line);
            }
          }

          const content = contentLines.join('\n').trim();
          const skillDirectory = path.dirname(resolved.skillFile);

          // Format output similar to Claude Code's Skill tool
          return `# ${name || skill_name}
# ${description || ''}
# Supporting tools and docs are in ${skillDirectory}
# ============================================

${content}`;
        }
      }
    ]
  };
};
```

**Step 2: 구문 검증**

실행: `node -c .opencode/plugin/superpowers.js`
예상 결과: 출력 없음

**Step 3: 커밋**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement use_skill tool for opencode"
```

---

### Task 11: find_skills 도구 구현

**파일:**
- 수정: `.opencode/plugin/superpowers.js`

**Step 1: tools 배열에 find_skills 도구 추가**

`use_skill` 도구 정의 뒤, `tools` 배열이 닫히기 전에 추가:

```javascript
      {
        name: 'find_skills',
        description: 'List all available skills in the superpowers and personal skill libraries.',
        schema: z.object({}),
        execute: async () => {
          // Find skills in both directories
          const superpowersSkills = skillsCore.findSkillsInDir(
            superpowersSkillsDir,
            'superpowers',
            3
          );
          const personalSkills = skillsCore.findSkillsInDir(
            personalSkillsDir,
            'personal',
            3
          );

          // Combine and format skills list
          const allSkills = [...personalSkills, ...superpowersSkills];

          if (allSkills.length === 0) {
            return 'No skills found. Install superpowers skills to ~/.config/opencode/superpowers/skills/';
          }

          let output = 'Available skills:\n\n';

          for (const skill of allSkills) {
            const namespace = skill.sourceType === 'personal' ? '' : 'superpowers:';
            const skillName = skill.name || path.basename(skill.path);

            output += `${namespace}${skillName}\n`;
            if (skill.description) {
              output += `  ${skill.description}\n`;
            }
            output += `  Directory: ${skill.path}\n\n`;
          }

          return output;
        }
      }
```

**Step 2: 구문 검증**

실행: `node -c .opencode/plugin/superpowers.js`
예상 결과: 출력 없음

**Step 3: 커밋**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement find_skills tool for opencode"
```

---

### Task 12: 세션 시작 훅 구현

**파일:**
- 수정: `.opencode/plugin/superpowers.js`

**Step 1: session.started 훅 추가**

`tools` 배열 다음에 추가:

```javascript
    'session.started': async () => {
      // Read using-superpowers skill content
      const usingSuperpowersPath = skillsCore.resolveSkillPath(
        'using-superpowers',
        superpowersSkillsDir,
        personalSkillsDir
      );

      let usingSuperpowersContent = '';
      if (usingSuperpowersPath) {
        const fullContent = fs.readFileSync(usingSuperpowersPath.skillFile, 'utf8');
        // Strip frontmatter
        const lines = fullContent.split('\n');
        let inFrontmatter = false;
        let frontmatterEnded = false;
        const contentLines = [];

        for (const line of lines) {
          if (line.trim() === '---') {
            if (inFrontmatter) {
              frontmatterEnded = true;
              continue;
            }
            inFrontmatter = true;
            continue;
          }

          if (frontmatterEnded || !inFrontmatter) {
            contentLines.push(line);
          }
        }

        usingSuperpowersContent = contentLines.join('\n').trim();
      }

      // Tool mapping instructions
      const toolMapping = `
**Tool Mapping for OpenCode:**
When skills reference tools you don't have, substitute OpenCode equivalents:
- \`TodoWrite\` → \`update_plan\` (your planning/task tracking tool)
- \`Task\` tool with subagents → Use OpenCode's subagent system (@mention syntax or automatic dispatch)
- \`Skill\` tool → \`use_skill\` custom tool (already available)
- \`Read\`, \`Write\`, \`Edit\`, \`Bash\` → Use your native tools

**Skill directories contain supporting files:**
- Scripts you can run with bash tool
- Additional documentation you can read
- Utilities and helpers specific to that skill

**Skills naming:**
- Superpowers skills: \`superpowers:skill-name\` (from ~/.config/opencode/superpowers/skills/)
- Personal skills: \`skill-name\` (from ~/.config/opencode/skills/)
- Personal skills override superpowers skills when names match
`;

      // Check for updates (non-blocking)
      const hasUpdates = skillsCore.checkForUpdates(
        path.join(homeDir, '.config/opencode/superpowers')
      );

      const updateNotice = hasUpdates ?
        '\n\n⚠️ **Updates available!** Run `cd ~/.config/opencode/superpowers && git pull` to update superpowers.' :
        '';

      // Return context to inject into session
      return {
        context: `<EXTREMELY_IMPORTANT>
You have superpowers.

**Below is the full content of your 'superpowers:using-superpowers' skill - your introduction to using skills. For all other skills, use the 'use_skill' tool:**

${usingSuperpowersContent}

${toolMapping}${updateNotice}
</EXTREMELY_IMPORTANT>`
      };
    }
```

**Step 2: 구문 검증**

실행: `node -c .opencode/plugin/superpowers.js`
예상 결과: 출력 없음

**Step 3: 커밋**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: implement session.started hook for opencode"
```

---

## Phase 4: 문서화

### Task 13: OpenCode 설치 가이드 작성

**파일:**
- 생성: `.opencode/INSTALL.md`

**Step 1: 설치 가이드 생성**

```markdown
# OpenCode용 Superpowers 설치 가이드

## 사전 요구 사항

- [OpenCode.ai](https://opencode.ai) 설치됨
- Node.js 설치됨
- Git 설치됨

## 설치 단계

### 1. Superpowers Skills 설치

```bash
# Clone superpowers skills to OpenCode config directory
mkdir -p ~/.config/opencode/superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. 플러그인 설치

플러그인은 방금 클론한 superpowers 리포지토리에 포함되어 있습니다.

OpenCode는 다음 위치에서 자동으로 발견합니다:
- `~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`

또는 프로젝트 로컬 플러그인 디렉터리에 심볼릭 링크를 걸 수 있습니다:

```bash
# In your OpenCode project
mkdir -p .opencode/plugin
ln -s ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js .opencode/plugin/superpowers.js
```

### 3. OpenCode 재시작

플러그인을 로드하려면 OpenCode를 재시작하세요. 다음 세션에서 다음과 같이 표시되어야 합니다:

```
You have superpowers.
```

## 사용법

### 스킬 검색

`find_skills` 도구를 사용하여 사용 가능한 모든 스킬을 나열합니다:

```
use find_skills tool
```

### 스킬 로드

`use_skill` 도구를 사용하여 특정 스킬을 로드합니다:

```
use use_skill tool with skill_name: "superpowers:brainstorming"
```

### 개인 스킬

`~/.config/opencode/skills/`에 나만의 스킬을 만드세요:

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

`~/.config/opencode/skills/my-skill/SKILL.md` 생성:

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

개인 스킬은 이름이 같은 경우 superpowers 스킬보다 우선 적용됩니다.

## 업데이트

```bash
cd ~/.config/opencode/superpowers
git pull
```

## 문제 해결

### 플러그인이 로드되지 않음

1. 플러그인 파일 존재 여부 확인: `ls ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`
2. 오류에 대해 OpenCode 로그 확인
3. Node.js 설치 여부 확인: `node --version`

### 스킬을 찾을 수 없음

1. 스킬 디렉터리 존재 여부 확인: `ls ~/.config/opencode/superpowers/skills`
2. `find_skills` 도구를 사용하여 발견된 내용 확인
3. 파일 구조 확인: 각 스킬에는 `SKILL.md` 파일이 있어야 합니다.

### 도구 매핑 이슈

스킬이 가지고 있지 않은 Claude Code 도구를 참조하는 경우:
- `TodoWrite` → `update_plan` 사용
- 서브에이전트가 포함된 `Task` → `@mention` 구문을 사용하여 OpenCode 서브에이전트 호출
- `Skill` → `use_skill` 도구 사용
- 파일 작업 → 네이티브 도구 사용

## 도움받기

- 이슈 제보: https://github.com/obra/superpowers/issues
- 문서: https://github.com/obra/superpowers
```

**Step 2: 파일 생성 확인**

실행: `ls -l .opencode/INSTALL.md`
예상 결과: 파일이 존재함

**Step 3: 커밋**

```bash
git add .opencode/INSTALL.md
git commit -m "docs: add opencode installation guide"
```

---

### Task 14: 메인 README 업데이트

**파일:**
- 수정: `README.md`

**Step 1: OpenCode 섹션 추가**

지원되는 플랫폼에 대한 섹션을 찾아서(파일에서 "Codex" 검색), 그 뒤에 추가합니다:

```markdown
### OpenCode

Superpowers works with [OpenCode.ai](https://opencode.ai) through a native JavaScript plugin.

**Installation:** See [.opencode/INSTALL.md](.opencode/INSTALL.md)

**Features:**
- Custom tools: `use_skill` and `find_skills`
- Automatic session bootstrap
- Personal skills with shadowing
- Supporting files and scripts access
```

**Step 2: 서식 검증**

실행: `grep -A 10 "### OpenCode" README.md`
예상 결과: 추가한 섹션을 보여줌

**Step 3: 커밋**

```bash
git add README.md
git commit -m "docs: add opencode support to readme"
```

---

### Task 15: Release Notes 업데이트

**파일:**
- 수정: `RELEASE-NOTES.md`

**Step 1: OpenCode 지원 항목 추가**

파일 상단(헤더 바로 다음)에 추가합니다:

```markdown
## [Unreleased]

### Added

- **OpenCode Support**: Native JavaScript plugin for OpenCode.ai
  - Custom tools: `use_skill` and `find_skills`
  - Automatic session bootstrap with tool mapping instructions
  - Shared core module (`lib/skills-core.js`) for code reuse
  - Installation guide in `.opencode/INSTALL.md`

### Changed

- **Refactored Codex Implementation**: Now uses shared `lib/skills-core.js` module
  - Eliminates code duplication between Codex and OpenCode
  - Single source of truth for skill discovery and parsing

---

```

**Step 2: 서식 검증**

실행: `head -30 RELEASE-NOTES.md`
예상 결과: 새 섹션을 보여줌

**Step 3: 커밋**

```bash
git add RELEASE-NOTES.md
git commit -m "docs: add opencode support to release notes"
```

---

## Phase 5: 최종 검증

### Task 16: Codex 정상 동작 여부 테스트

**파일:**
- 테스트: `.codex/superpowers-codex`

**Step 1: find-skills 명령어 테스트**

실행: `.codex/superpowers-codex find-skills | head -20`
예상 결과: 이름과 설명이 포함된 스킬 목록을 보여줌

**Step 2: use-skill 명령어 테스트**

실행: `.codex/superpowers-codex use-skill superpowers:brainstorming | head -20`
예상 결과: brainstorming 스킬 내용을 보여줌

**Step 3: bootstrap 명령어 테스트**

실행: `.codex/superpowers-codex bootstrap | head -30`
예상 결과: 지침이 포함된 부트스트랩 내용을 보여줌

**Step 4: 모든 테스트 통과 시 성공 기록**

커밋 불필요 - 검증 전용 단계입니다.

---

### Task 17: 파일 구조 검증

**파일:**
- 확인: 모든 새 파일이 존재하는지 확인

**Step 1: 모든 파일 생성 여부 확인**

실행:
```bash
ls -l lib/skills-core.js
ls -l .opencode/plugin/superpowers.js
ls -l .opencode/INSTALL.md
```

예상 결과: 모든 파일이 존재함

**Step 2: 디렉터리 구조 확인**

실행: `tree -L 2 .opencode/` (또는 tree를 사용할 수 없는 경우 `find .opencode -type f`)
예상 결과:
```
.opencode/
├── INSTALL.md
└── plugin/
    └── superpowers.js
```

**Step 3: 구조가 올바른 경우 계속 진행**

커밋 불필요 - 검증 전용 단계입니다.

---

### Task 18: 최종 커밋 및 요약

**파일:**
- 확인: `git status`

**Step 1: git status 확인**

실행: `git status`
예상 결과: 작업 트리 깨끗함, 모든 변경 사항이 커밋됨

**Step 2: 커밋 로그 검토**

실행: `git log --oneline -20`
예상 결과: 이 구현으로 인한 모든 커밋을 보여줌

**Step 3: 요약 문서 작성**

다음 내용을 포함하는 완료 요약 작성:
- 완료된 총 커밋 수
- 생성된 파일: `lib/skills-core.js`, `.opencode/plugin/superpowers.js`, `.opencode/INSTALL.md`
- 수정된 파일: `.codex/superpowers-codex`, `README.md`, `RELEASE-NOTES.md`
- 수행된 테스트: Codex 명령어 검증 완료
- 다음 단계 준비: 실제 OpenCode 설치 환경에서 테스트할 준비 완료

**Step 4: 완료 보고**

사용자에게 요약 보고서를 제시하고 다음 옵션 제안:
1. 푸시하여 원격 저장소에 반영
2. Pull Request 생성
3. 실제 OpenCode 설치 환경에서 테스트 (OpenCode 설치 필요)

---

## 테스트 가이드 (수동 - OpenCode 필요)

이 단계들은 OpenCode가 설치되어 있어야 하며 자동화된 구현의 일부가 아닙니다:

1. **스킬 설치**: `.opencode/INSTALL.md` 지침을 따르세요.
2. **OpenCode 세션 시작**: 부트스트랩 메시지가 나타나는지 확인합니다.
3. **find_skills 테스트**: 사용 가능한 모든 스킬을 목록화해야 합니다.
4. **use_skill 테스트**: 스킬을 로드하고 내용이 나타나는지 확인합니다.
5. **지원 파일 테스트**: 스킬 디렉터리 경로에 접근 가능한지 확인합니다.
6. **개인 스킬 테스트**: 개인 스킬을 생성하고 코어 스킬을 올바르게 오버라이드(shadowing)하는지 확인합니다.
7. **도구 매핑 테스트**: TodoWrite → update_plan 매핑이 제대로 작동하는지 확인합니다.

## 성공 기준

- [ ] 모든 핵심 기능이 포함된 `lib/skills-core.js` 생성 완료
- [ ] 공유 코어를 사용하도록 `.codex/superpowers-codex` 리팩터링 완료
- [ ] Codex 명령어 정상 작동 확인 (find-skills, use-skill, bootstrap)
- [ ] 도구 및 훅이 포함된 `.opencode/plugin/superpowers.js` 생성 완료
- [ ] 설치 가이드 작성 완료
- [ ] README 및 RELEASE-NOTES 업데이트 완료
- [ ] 모든 변경 사항 커밋 완료
- [ ] 작업 트리 상태 깨끗함
