# 스킬 작성 모범 사례 (Skill authoring best practices)

> 에이전트가 성공적으로 탐색하고 사용할 수 있는 효과적인 스킬 작성법을 알아보세요.

좋은 스킬은 간결하고 구조화가 잘 되어 있으며, 실제 사용을 통해 테스트된 스킬입니다. 이 가이드는 에이전트가 스킬을 쉽게 발견하고 효과적으로 사용할 수 있도록 돕는 실용적인 작성 지침을 제공합니다.

스킬 작동 방식에 대한 개념적 배경은 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)를 참조하세요.

## 핵심 원칙 (Core principles)

### 간결함이 핵심 (Concise is key)

[컨텍스트 윈도우](https://platform.claude.com/docs/en/build-with-claude/context-windows)는 공공의 자산입니다. 작성한 스킬은 에이전트가 알아야 하는 다음 요소들과 컨텍스트 윈도우를 공유합니다:

* 시스템 프롬프트
* 대화 히스토리
* 다른 스킬의 메타데이터
* 실제 요청 내용

스킬의 모든 토큰이 즉각적인 비용을 발생시키는 것은 아닙니다. 시작 시에는 모든 스킬의 메타데이터(이름 및 설명)만 미리 로드됩니다. 에이전트는 스킬이 관련성을 가질 때만 SKILL.md를 읽고, 추가 파일은 필요할 때만 읽습니다. 그럼에도 불구하고 SKILL.md를 간결하게 유지하는 것은 매우 중요합니다. 에이전트가 스킬을 로드하면 모든 토큰이 대화 히스토리 및 다른 컨텍스트와 경쟁하기 때문입니다.

**기본 전제**: 에이전트는 이미 매우 똑똑합니다.

에이전트가 아직 가지고 있지 않은 컨텍스트만 추가하세요. 각 정보에 대해 스스로 질문해 보세요:

* "에이전트에게 이 설명이 정말로 필요한가?"
* "에이전트가 이미 이것을 알고 있다고 전제할 수 있는가?"
* "이 단락이 토큰 비용만큼의 가치를 지니는가?"

**좋은 예: 간결함** (약 50 토큰):

````markdown  theme={null}
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**나쁜 예: 지나치게 장황함** (약 150 토큰):

```markdown  theme={null}
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available for PDF processing, but we
recommend pdfplumber because it's easy to use and handles most cases well.
First, you'll need to install it using pip. Then you can use the code below...
```

간결한 버전은 에이전트가 PDF가 무엇인지, 라이브러리가 어떻게 작동하는지 이미 알고 있다고 전제합니다.

### 적절한 자유도 설정 (Set appropriate degrees of freedom)

작업의 취약성과 변동성에 맞춰 구체성의 수준을 조절하세요.

**높은 자유도** (텍스트 기반 지침):

사용 시점:

* 여러 접근 방식이 모두 유효할 때
* 결정이 컨텍스트에 따라 달라질 때
* 휴리스틱이 접근 방식을 안내할 때

예시:

```markdown  theme={null}
## Code review process

1. Analyze the code structure and organization
2. Check for potential bugs or edge cases
3. Suggest improvements for readability and maintainability
4. Verify adherence to project conventions
```

**중간 자유도** (파라미터가 포함된 의사코드 또는 스크립트):

사용 시점:

* 선호하는 패턴이 존재할 때
* 약간의 변형이 허용될 때
* 설정이 동작에 영향을 미칠 때

예시:

````markdown  theme={null}
## Generate report

Use this template and customize as needed:

```python
def generate_report(data, format="markdown", include_charts=True):
    # Process data
    # Generate output in specified format
    # Optionally include visualizations
```
````

**낮은 자유도** (특정 스크립트, 파라미터가 거의 없거나 없음):

사용 시점:

* 작업이 취약하고 오류가 발생하기 쉬울 때
* 일관성이 결정적일 때
* 특정 순서를 반드시 따라야 할 때

예시:

````markdown  theme={null}
## Database migration

Run exactly this script:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
````

**비유**: 에이전트를 경로를 탐험하는 로봇으로 생각해 보세요:

* **양쪽이 절벽인 좁은 다리**: 안전한 길은 단 하나뿐입니다. 구체적인 안전장치와 정확한 지침을 제공하세요 (낮은 자유도). 예: 정확한 순서로 실행되어야 하는 데이터베이스 마이그레이션.
* **위험 요소가 없는 탁 트인 들판**: 성공으로 이어지는 길은 많습니다. 대략적인 방향만 제시하고 에이전트가 최선의 경로를 찾도록 신뢰하세요 (높은 자유도). 예: 컨텍스트에 따라 최선의 접근 방식이 결정되는 코드 검토.

### 사용하려는 모든 모델로 테스트 (Test with all models you plan to use)

스킬은 모델의 확장 요소로 작동하므로 그 효과는 기반 모델에 따라 달라집니다. 사용할 예정인 모든 모델에서 스킬을 테스트하세요.

**모델별 테스트 고려사항**:

* **Claude Haiku** (빠르고 경제적): 스킬이 충분한 지침을 제공하는가?
* **Claude Sonnet** (균형잡힘): 스킬이 명확하고 효율적인가?
* **Claude Opus** (강력한 추론 능력): 스킬이 과도한 설명을 피하고 있는가?

Opus에서 완벽하게 작동하는 내용이 Haiku에는 더 많은 세부사항이 필요할 수 있습니다. 여러 모델에서 스킬을 사용할 계획이라면 모든 모델에서 잘 작동하는 지침을 목표로 하세요.

## 스킬 구조 (Skill structure)

<Note>
  **YAML Frontmatter**: SKILL.md의 frontmatter에는 두 개의 필수 필드가 필요합니다:

  * `name` - 스킬의 사람이 읽을 수 있는 이름 (최대 64자)
  * `description` - 스킬이 수행하는 작업과 사용 시점에 대한 한 줄 설명 (최대 1024자)

  전체 스킬 구조에 대한 상세 내용은 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)를 참조하세요.
</Note>

### 명명 규칙 (Naming conventions)

스킬을 더 쉽게 참조하고 논의할 수 있도록 일관된 명명 패턴을 사용하세요. 스킬 이름에는 **동명사 형태** (동사 + -ing)를 사용할 것을 권장합니다. 이는 스킬이 제공하는 활동이나 기능을 명확하게 설명하기 때문입니다.

**좋은 명명 예시 (동명사 형태)**:

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**수용 가능한 대안**:

* 명사구: "PDF Processing", "Spreadsheet Analysis"
* 작업 지향형: "Process PDFs", "Analyze Spreadsheets"

**피해야 할 형태**:

* 모호한 이름: "Helper", "Utils", "Tools"
* 지나치게 일반적인 이름: "Documents", "Data", "Files"
* 스킬 컬렉션 내에서 일관성이 없는 패턴

일관된 명명법을 통해 다음이 용이해집니다:

* 문서 및 대화에서 스킬 참조
* 한눈에 스킬이 수행하는 작업 파악
* 여러 스킬 정리 및 검색
* 전문적이고 결합력 있는 스킬 라이브러리 유지 관리

### 효과적인 설명 작성 (Writing effective descriptions)

`description` 필드는 스킬 탐색을 가능하게 하며, 스킬이 무엇을 하는지와 언제 사용해야 하는지를 모두 포함해야 합니다.

<Warning>
  **항상 3인칭으로 작성하세요**. 설명은 시스템 프롬프트에 주입되므로 시점이 일치하지 않으면 탐색에 문제가 발생할 수 있습니다.

  * **좋음:** "Processes Excel files and generates reports"
  * **피할 것:** "I can help you process Excel files"
  * **피할 것:** "You can use this to process Excel files"
</Warning>

**구체적이어야 하며 핵심 용어를 포함하세요**. 스킬이 수행하는 작업과 이를 사용할 특정 트리거/컨텍스트를 모두 포함하세요.

각 스킬에는 정확히 하나의 description 필드가 있습니다. 설명은 스킬 선택에 매우 결정적입니다. 에이전트는 이용 가능한 100개 이상의 스킬 중에서 올바른 스킬을 선택하는 데 이 설명을 사용합니다. 설명에는 에이전트가 언제 이 스킬을 선택해야 할지 알 수 있는 충분한 세부정보가 포함되어야 하며, SKILL.md의 나머지 부분은 구현 세부사항을 제공합니다.

효과적인 예시:

**PDF Processing skill:**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel Analysis skill:**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git Commit Helper skill:**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

다음과 같이 모호한 설명은 피하세요:

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 단계적 공개 패턴 (Progressive disclosure patterns)

SKILL.md는 온보딩 가이드의 목차처럼 에이전트에 필요한 세부 자료를 안내하는 개요 역할을 합니다. 단계적 공개 방식에 대한 설명은 개요 문서의 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)를 참조하세요.

**실용적인 지침**:

* 최적의 성능을 위해 SKILL.md 본문은 500줄 미만으로 유지하세요.
* 이 제한에 도달하면 내용을 별도 파일로 분할하세요.
* 지침, 코드 및 리소스를 효과적으로 정리하려면 아래 패턴을 사용하세요.

#### 시각적 개요: 단순함에서 복잡함으로

기본 스킬은 메타데이터와 지침이 포함된 하나의 SKILL.md 파일로 시작합니다:

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

스킬이 커짐에 따라 에이전트가 필요할 때만 로드하는 추가 콘텐츠를 함께 번들링할 수 있습니다:

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

전체 스킬 디렉토리 구조는 다음과 같은 모습일 수 있습니다:

```
pdf/
├── SKILL.md              # Main instructions (loaded when triggered)
├── FORMS.md              # Form-filling guide (loaded as needed)
├── reference.md          # API reference (loaded as needed)
├── examples.md           # Usage examples (loaded as needed)
└── scripts/
    ├── analyze_form.py   # Utility script (executed, not loaded)
    ├── fill_form.py      # Form filling script
    └── validate.py       # Validation script
```

#### 패턴 1: 참조 자료가 포함된 고수준 가이드

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns
````

에이전트는 필요할 때만 FORMS.md, REFERENCE.md 또는 EXAMPLES.md를 로드합니다.

#### 패턴 2: 도메인별 구조화

여러 도메인을 다루는 스킬의 경우, 무관한 컨텍스트가 로드되는 것을 방지하기 위해 도메인별로 내용을 정리하세요. 사용자가 영업 지표에 대해 물을 때, 에이전트는 영업 관련 스키마만 읽으면 되며 재무나 마케팅 데이터는 읽을 필요가 없습니다. 이는 토큰 사용량을 낮추고 컨텍스트를 집중적으로 유지해 줍니다.

```
bigquery-skill/
├── SKILL.md (overview and navigation)
└── reference/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    ├── product.md (API usage, features)
    └── marketing.md (campaigns, attribution)
```

````markdown SKILL.md theme={null}
# BigQuery Data Analysis

## Available datasets

**Finance**: Revenue, ARR, billing → See [reference/finance.md](reference/finance.md)
**Sales**: Opportunities, pipeline, accounts → See [reference/sales.md](reference/sales.md)
**Product**: API usage, features, adoption → See [reference/product.md](reference/product.md)
**Marketing**: Campaigns, attribution, email → See [reference/marketing.md](reference/marketing.md)

## Quick search

Find specific metrics using grep:

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 패턴 3: 조건부 세부사항

기본 내용을 표시하고, 고급 내용은 링크로 연결합니다:

```markdown  theme={null}
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

에이전트는 사용자가 해당 기능을 필요로 할 때만 REDLINING.md 또는 OOXML.md를 읽습니다.

### 깊게 중첩된 참조 피하기 (Avoid deeply nested references)

에이전트는 참조된 다른 파일에서 다시 참조된 파일의 경우 내용을 부분적으로만 읽을 수 있습니다. 중첩된 참조를 만나면 에이전트는 전체 파일을 읽는 대신 `head -100`과 같은 명령어로 내용을 미리보기하여 정보가 누락될 수 있습니다.

**참조는 SKILL.md로부터 1단계 깊이로 유지하세요**. 모든 참조 파일은 SKILL.md에서 직접 링크되어 필요할 때 에이전트가 전체 파일을 읽을 수 있도록 해야 합니다.

**나쁜 예: 너무 깊음**:

```markdown  theme={null}
# SKILL.md
See [advanced.md](advanced.md)...

# advanced.md
See [details.md](details.md)...

# details.md
Here's the actual information...
```

**좋은 예: 1단계 깊이**:

```markdown  theme={null}
# SKILL.md

**Basic usage**: [instructions in SKILL.md]
**Advanced features**: See [advanced.md](advanced.md)
**API reference**: See [reference.md](reference.md)
**Examples**: See [examples.md](examples.md)
```

### 긴 참조 파일은 목차 구조화하기

100줄이 넘는 긴 참조 파일의 경우 상단에 목차(table of contents)를 포함하세요. 이를 통해 에이전트는 부분 읽기로 미리보기할 때도 이용 가능한 전체 정보의 범위를 파악할 수 있습니다.

**예시**:

```markdown  theme={null}
# API Reference

## Contents
- Authentication and setup
- Core methods (create, read, update, delete)
- Advanced features (batch operations, webhooks)
- Error handling patterns
- Code examples

## Authentication and setup
...

## Core methods
...
```

그러면 에이전트는 필요한 경우 전체 파일을 읽거나 특정 섹션으로 건너뛸 수 있습니다.

이 파일 시스템 기반 아키텍처가 어떻게 단계적 공개를 가능하게 하는지에 대한 세부 정보는 아래의 [실행 환경(Runtime environment)](#runtime-environment) 섹션을 참조하세요.

## 워크플로우 및 피드백 루프 (Workflows and feedback loops)

### 복잡한 작업에는 워크플로우 활용

복잡한 작업은 명확하고 순차적인 단계로 나누세요. 특히 복잡한 워크플로우의 경우 에이전트가 응답에 복사하여 진행 상황을 체크할 수 있는 체크리스트를 제공하세요.

**예시 1: 연구 종합 워크플로우** (코드가 없는 스킬의 경우):

````markdown  theme={null}
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1: Read all source documents**

Review each document in the `sources/` directory. Note the main arguments and supporting evidence.

**Step 2: Identify key themes**

Look for patterns across sources. What themes appear repeatedly? Where do sources agree or disagree?

**Step 3: Cross-reference claims**

For each major claim, verify it appears in the source material. Note which source supports each point.

**Step 4: Create structured summary**

Organize findings by theme. Include:
- Main claim
- Supporting evidence from sources
- Conflicting viewpoints (if any)

**Step 5: Verify citations**

Check that every claim references the correct source document. If citations are incomplete, return to Step 3.
````

이 예시는 코드가 필요 없는 분석 작업에 워크플로우가 어떻게 적용되는지 보여줍니다. 체크리스트 패턴은 모든 복잡한 다단계 프로세스에 작동합니다.

**예시 2: PDF 양식 작성 워크플로우** (코드가 포함된 스킬의 경우):

````markdown  theme={null}
## PDF form filling workflow

Copy this checklist and check off items as you complete them:

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1: Analyze the form**

Run: `python scripts/analyze_form.py input.pdf`

This extracts form fields and their locations, saving to `fields.json`.

**Step 2: Create field mapping**

Edit `fields.json` to add values for each field.

**Step 3: Validate mapping**

Run: `python scripts/validate_fields.py fields.json`

Fix any validation errors before continuing.

**Step 4: Fill the form**

Run: `python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5: Verify output**

Run: `python scripts/verify_output.py output.pdf`

If verification fails, return to Step 2.
````

명확한 단계 구성은 에이전트가 중요한 검증 절차를 건너뛰는 것을 방지합니다. 체크리스트는 사용자 및 에이전트 모두가 다단계 워크플로우 진행 상황을 추적하는 데 도움이 됩니다.

### 피드백 루프 구현

**일반적인 패턴**: 검증기 실행 → 에러 수정 → 반복

이 패턴은 출력 결과물의 품질을 크게 향상시킵니다.

**예시 1: 스타일 가이드 준수** (코드가 없는 스킬의 경우):

```markdown  theme={null}
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

이는 스크립트 대신 참조 문서를 사용하는 검증 루프 패턴을 보여줍니다. 여기서 "검증기"는 STYLE_GUIDE.md이며 에이전트는 읽기 및 비교를 통해 점검을 수행합니다.

**예시 2: 문서 편집 프로세스** (코드가 포함된 스킬의 경우):

```markdown  theme={null}
## Document editing process

1. Make your edits to `word/document.xml`
2. **Validate immediately**: `python ooxml/scripts/validate.py unpacked_dir/`
3. If validation fails:
   - Review the error message carefully
   - Fix the issues in the XML
   - Run validation again
4. **Only proceed when validation passes**
5. Rebuild: `python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. Test the output document
```

검증 루프는 에러를 조기에 잡아냅니다.

## 콘텐츠 지침 (Content guidelines)

### 시간에 민감한 정보 피하기

시간이 지나면 구식이 되는 정보는 포함하지 마세요:

**나쁜 예: 시간에 민감함** (시간이 지나면 틀린 정보가 됨):

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**좋은 예** ("구 패턴" 섹션 활용):

```markdown  theme={null}
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

구 패턴 섹션은 메인 콘텐츠를 어지럽히지 않으면서 역사적 컨텍스트를 제공합니다.

### 일관된 용어 사용

하나의 용어를 선택하고 스킬 전반에 걸쳐 사용하세요:

**좋음 - 일관됨**:

* 항상 "API endpoint"
* 항상 "field"
* 항상 "extract"

**나쁨 - 불일치**:

* "API endpoint", "URL", "API route", "path" 혼용
* "field", "box", "element", "control" 혼용
* "extract", "pull", "get", "retrieve" 혼용

일관성은 에이전트가 지침을 이해하고 따르는 데 도움이 됩니다.

## 공통 패턴 (Common patterns)

### 템플릿 패턴 (Template pattern)

출력 포맷을 위한 템플릿을 제공하세요. 엄격함의 수준을 필요에 맞게 조절하세요.

**엄격한 요구사항의 경우** (API 응답 또는 데이터 포맷 등):

````markdown  theme={null}
## Report structure

ALWAYS use this exact template structure:

```markdown
# [Analysis Title]

## Executive summary
[One-paragraph overview of key findings]

## Key findings
- Finding 1 with supporting data
- Finding 2 with supporting data
- Finding 3 with supporting data

## Recommendations
1. Specific actionable recommendation
2. Specific actionable recommendation
```
````

**유연한 지침의 경우** (변형 적용이 유용한 경우):

````markdown  theme={null}
## Report structure

Here is a sensible default format, but use your best judgment based on the analysis:

```markdown
# [Analysis Title]

## Executive summary
[Overview]

## Key findings
[Adapt sections based on what you discover]

## Recommendations
[Tailor to the specific context]
```

Adjust sections as needed for the specific analysis type.
````

### 예시 패턴 (Examples pattern)

출력 품질이 예시를 보는 것에 의존하는 스킬의 경우 일반 프롬프팅처럼 입/출력 쌍을 제공하세요:

````markdown  theme={null}
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**Example 2:**
Input: Fixed bug where dates displayed incorrectly in reports
Output:
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**Example 3:**
Input: Updated dependencies and refactored error handling
Output:
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

Follow this style: type(scope): brief description, then detailed explanation.
````

예시는 설명만 제공하는 것보다 에이전트가 원하는 스타일과 상세 수준을 훨씬 더 명확하게 이해할 수 있도록 돕습니다.

### 조건부 워크플로우 패턴

의사 결정 지점을 통해 에이전트를 안내하세요:

```markdown  theme={null}
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

<Tip>
  워크플로우가 여러 단계로 구성되어 커지거나 복잡해지는 경우, 별도 파일로 분리하고 에이전트에게 수행할 작업에 따라 적절한 파일을 읽도록 지시하는 것을 고려하세요.
</Tip>

## 평가 및 반복 개선 (Evaluation and iteration)

### 평가 체계(Evaluation)를 먼저 구축하기

**광범위한 문서를 작성하기 전에 평가 체계를 먼저 만드세요.** 이를 통해 작성한 스킬이 상상 속의 문제가 아닌 실제 문제를 해결하도록 할 수 있습니다.

**평가 중심 개발 (Evaluation-driven development):**

1. **격차 식별**: 스킬 없이 에이전트에게 대표적인 작업을 실행시켜 봅니다. 특정 실패나 누락된 컨텍스트를 문서화합니다.
2. **평가 체계 작성**: 이러한 격차를 테스트하는 3가지 시나리오를 구축합니다.
3. **베이스라인 확립**: 스킬이 없을 때의 에이전트 성능을 측정합니다.
4. **최소한의 지침 작성**: 격차를 해결하고 평가를 통과하기에 충분한 내용만 작성합니다.
5. **반복 개선**: 평가를 실행하고 베이스라인과 비교하여 개선합니다.

이러한 접근 방식은 현실화되지 않을 수도 있는 요구사항을 지레짐작하는 대신 실제 문제를 해결할 수 있도록 해줍니다.

**평가 구조**:

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  이 예시는 간단한 테스트 루브릭이 포함된 데이터 기반 평가 체계를 보여줍니다. 현재 이러한 평가를 실행하는 기본 내장 도구는 제공되지 않으므로 사용자가 자체 평가 시스템을 구축할 수 있습니다. 평가는 스킬의 효과성을 측정하는 진실의 원천입니다.
</Note>

### 에이전트와 함께 반복적으로 스킬 개발하기

가장 효과적인 스킬 개발 프로세스는 에이전트 자체를 참여시키는 것입니다. 한 인스턴스("Agent A")와 협력하여 다른 인스턴스("Agent B")가 사용할 스킬을 생성합니다. Agent A는 지침을 설계하고 다듬는 것을 돕고, Agent B는 실제 작업에서 이를 테스트합니다. 기반 모델은 효과적인 에이전트 지침 작성 방법과 에이전트에게 필요한 정보가 무엇인지를 모두 이해하고 있기 때문에 이 방식이 작동합니다.

**새로운 스킬 생성하기:**

1. **스킬 없이 작업 완료**: 일반 프롬프팅을 사용하여 Agent A와 함께 문제를 해결합니다. 작업하는 동안 자연스럽게 컨텍스트를 제공하고, 선호 사항을 설명하며, 절차적 지식을 공유하게 됩니다. 반복적으로 제공하는 정보가 무엇인지 주목하세요.

2. **재사용 가능한 패턴 식별**: 작업을 완료한 후, 향후 유사한 작업에 유용할 수 있는 어떤 컨텍스트를 제공했는지 식별합니다.

   **예시**: BigQuery 분석을 진행했다면 테이블 이름, 필드 정의, 필터링 규칙("테스트 계정은 항상 제외" 등), 일반적인 쿼리 패턴을 제공했을 수 있습니다.

3. **Agent A에게 스킬 생성 요청**: "방금 사용한 BigQuery 분석 패턴을 캡처하는 스킬을 생성해 줘. 테이블 스키마, 명명 규칙, 테스트 계정 필터링 규칙을 포함시켜 줘."

   <Tip>
     최신 에이전트는 스킬 포맷과 구조를 기본적으로 이해합니다. 스킬을 만드는 데 특별한 시스템 프롬프트나 "스킬 작성" 스킬이 필요하지 않습니다. 에이전트에게 스킬을 생성하도록 요청하기만 하면 적절한 frontmatter와 본문이 포함된 잘 구조화된 SKILL.md 내용을 생성합니다.
   </Tip>

4. **간결성 검토**: Agent A가 불필요한 설명을 추가하지 않았는지 확인하세요. 요청: "승률이 무엇을 의미하는지에 대한 설명은 지워 줘 - 에이전트는 이미 그걸 알고 있어."

5. **정보 아키텍처 개선**: Agent A에게 내용을 더 효과적으로 정리하도록 요청하세요. 예를 들어: "테이블 스키마가 별도의 참조 파일에 들어가도록 정리해 줘. 나중에 더 많은 테이블을 추가할 수도 있어."

6. **유사한 작업에서 테스트**: 스킬이 로드된 새로운 인스턴스인 Agent B와 함께 관련 유스케이스에서 스킬을 사용해 보세요. Agent B가 올바른 정보를 찾고, 규칙을 적절히 적용하며, 작업을 성공적으로 처리하는지 관찰하세요.

7. **관찰에 기반한 반복 개선**: Agent B가 어려움을 겪거나 무언가를 놓치면 Agent A에게 구체적인 내용을 가지고 돌아가세요: "에이전트가 이 스킬을 사용할 때 Q4 날짜 필터링을 잊어버렸어. 날짜 필터링 패턴에 대한 섹션을 추가해야 할까?"

**기존 스킬 반복 개선하기:**

동일한 계층적 패턴이 기존 스킬을 개선할 때도 지속됩니다. 다음을 번갈아 진행하게 됩니다:

* **Agent A와 작업** (스킬 다듬기를 돕는 전문가)
* **Agent B로 테스트** (실제 작업을 수행하기 위해 스킬을 사용하는 에이전트)
* **Agent B의 동작 관찰** 및 Agent A에게 인사이트 전달

1. **실제 워크플로우에 스킬 사용**: 테스트 시나리오가 아닌 실제 작업을 스킬이 로드된 Agent B에게 부여합니다.

2. **Agent B의 동작 관찰**: 어려움을 겪는 부분, 성공하는 부분, 예상치 못한 선택을 하는 부분에 주목합니다.

   **관찰 예시**: "Agent B에게 지역별 매출 보고서를 요청했을 때, 스킬에 규칙이 언급되어 있음에도 불구하고 쿼리를 작성할 때 테스트 계정 필터링을 잊어버렸다."

3. **개선을 위해 Agent A에게 돌아가기**: 현재 SKILL.md를 공유하고 관찰한 내용을 설명합니다. 질문: "지역 보고서를 요청했을 때 Agent B가 테스트 계정 필터링을 잊어버린 것을 관찰했어. 스킬에 필터링이 언급되어 있지만 눈에 잘 띄지 않는 것 같아."

4. **Agent A의 제안 검토**: Agent A는 규칙을 더 눈에 띄게 재구성하거나, "always filter" 대신 "MUST filter"와 같은 더 강력한 어조를 사용하거나, 워크플로우 섹션을 재구조화할 것을 제안할 수 있습니다.

5. **수정 사항 적용 및 테스트**: Agent A의 다듬어진 내용을 바탕으로 스킬을 업데이트한 다음, 유사한 요청으로 Agent B에서 다시 테스트합니다.

6. **사용에 기반한 반복**: 새로운 시나리오를 만남에 따라 이 관찰-다듬기-테스트 사이클을 계속하세요. 각 반복은 추측이 아닌 실제 에이전트 동작을 바탕으로 스킬을 개선합니다.

**팀 피드백 수집:**

1. 팀원들과 스킬을 공유하고 사용 패턴을 관찰하세요.
2. 질문하기: 스킬이 예상 시점에 활성화되는가? 지침이 명확한가? 무엇이 빠졌는가?
3. 자신만의 사용 패턴에 존재하는 사각지대를 해결하기 위해 피드백을 반영하세요.

**이 접근 방식이 작동하는 이유**: Agent A는 에이전트의 니즈를 이해하고, 당신은 도메인 전문 지식을 제공하며, Agent B는 실제 사용을 통해 격차를 드러내고, 반복적인 다듬기는 추측이 아닌 관찰된 동작을 바탕으로 스킬을 개선합니다.

### 에이전트의 스킬 탐색 방식 관찰 (Observe how agents navigate Skills)

스킬을 반복 개선할 때 에이전트가 실제로 스킬을 사용하는 방식을 세심히 관찰하세요. 다음에 주목하세요:

* **예상치 못한 탐색 경로**: 에이전트가 예상하지 못한 순서로 파일을 읽나요?이는 구조가 생각만큼 직관적이지 않음을 나타낼 수 있습니다.
* **놓친 연결 고리**: 에이전트가 중요한 파일에 대한 참조를 따라가지 못하나요? 링크를 더 명시적이거나 눈에 띄게 만들어야 할 수 있습니다.
* **특정 섹션에 대한 과도한 의존**: 에이전트가 동일한 파일을 반복해서 읽는다면 해당 내용을 메인 SKILL.md에 두어야 할지 고려해 보세요.
* **무시된 콘텐츠**: 에이전트가 번들 파일을 전혀 읽지 않는다면 필요 없거나 메인 지침에서 신호가 제대로 전달되지 않은 것일 수 있습니다.

추측이 아닌 이러한 관찰을 바탕으로 반복 개선하세요. 스킬 메타데이터의 'name'과 'description'은 특히 중요합니다. 에이전트는 현재 작업에 대응하여 스킬을 트리거할지 결정할 때 이를 사용합니다. 스킬이 무엇을 하는지, 언제 사용해야 하는지 명확하게 기술되어 있는지 확인하세요.

## 피해야 할 안티 패턴 (Anti-patterns to avoid)

### Windows 스타일 경로 피하기

Windows 환경이더라도 파일 경로에는 항상 슬래시(`/`)를 사용하세요:

* ✓ **좋음**: `scripts/helper.py`, `reference/guide.md`
* ✗ **피할 것**: `scripts\helper.py`, `reference\guide.md`

Unix 스타일 경로는 모든 플랫폼에서 작동하지만, Windows 스타일 경로는 Unix 시스템에서 에러를 유발합니다.

### 지나치게 많은 옵션 제공 피하기

필요하지 않다면 여러 접근 방식을 제시하지 마세요:

````markdown  theme={null}
**Bad example: Too many choices** (confusing):
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**Good example: Provide a default** (with escape hatch):
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 고급: 실행 가능한 코드가 포함된 스킬 (Advanced: Skills with executable code)

아래 섹션은 실행 가능한 스크립트가 포함된 스킬에 초점을 맞춥니다. 마크다운 지침만 사용하는 스킬의 경우 [Checklist for effective Skills](#checklist-for-effective-skills)로 건너뛰세요.

### 에이전트에 미루지 말고 직접 해결 (Solve, don't punt)

스킬용 스크립트를 작성할 때 에러 조건을 에이전트에게 미루지 말고 직접 처리하세요.

**좋은 예: 에러를 명시적으로 처리**:

```python  theme={null}
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ''
```

**나쁜 예: 에라이 모르겠다 하고 에이전트에게 미룸**:

```python  theme={null}
def process_file(path):
    # Just fail and let the agent figure it out
    return open(path).read()
```

설정 파라미터 역시 "부두 상수(voodoo constants, 근거 없는 상수)"가 되지 않도록 타당성을 밝히고 문서화해야 합니다(Ousterhout 법칙). 작성자조차 올바른 값을 모른다면 에이전트가 어떻게 이를 결정하겠습니까?

**좋은 예: 자체 문서화**:

```python  theme={null}
# HTTP requests typically complete within 30 seconds
# Longer timeout accounts for slow connections
REQUEST_TIMEOUT = 30

# Three retries balances reliability vs speed
# Most intermittent failures resolve by the second retry
MAX_RETRIES = 3
```

**나쁜 예: 매직 넘버**:

```python  theme={null}
TIMEOUT = 47  # Why 47?
RETRIES = 5   # Why 5?
```

### 유틸리티 스크립트 제공

에이전트가 직접 스크립트를 작성할 수 있더라도 사전에 만들어진 스크립트는 이점을 제공합니다:

**유틸리티 스크립트의 이점**:

* 생성된 코드보다 안정적임
* 토큰 절약 (컨텍스트에 코드를 포함할 필요 없음)
* 시간 절약 (코드 생성 절차 불필요)
* 사용 간 일관성 보장

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

위 다이어그램은 지침 파일과 함께 실행 가능한 스크립트가 어떻게 작용하는지 보여줍니다. 지침 파일(forms.md)이 스크립트를 참조하면 에이전트는 해당 스크립트의 전체 내용을 컨텍스트에 로드하지 않고 실행할 수 있습니다.

**중요한 차이점**: 지침에서 에이전트가 다음 중 무엇을 해야 하는지 명확히 밝히세요:

* **스크립트 실행** (가장 일반적): "Run `analyze_form.py` to extract fields"
* **참조용으로 읽기** (복잡한 로직의 경우): "See `analyze_form.py` for the field extraction algorithm"

대부분의 유틸리티 스크립트의 경우 더 안정적이고 효율적이므로 실행하는 방식이 선호됩니다. 스크립트 실행 작동 방식에 대한 상세 내용은 아래의 [실행 환경(Runtime environment)](#runtime-environment) 섹션을 참조하세요.

**예시**:

````markdown  theme={null}
## Utility scripts

**analyze_form.py**: Extract all form fields from PDF

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

Output format:
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**: Check for overlapping bounding boxes

```bash
python scripts/validate_boxes.py fields.json
# Returns: "OK" or lists conflicts
```

**fill_form.py**: Apply field values to PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 시각적 분석 활용

입력물을 이미지로 렌더링할 수 있는 경우 에이전트가 이를 분석하도록 하세요:

````markdown  theme={null}
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. The agent can see field locations and types visually
````

<Note>
  이 예시에서는 `pdf_to_images.py` 스크립트를 직접 작성해야 합니다.
</Note>

에이전트의 비전(Vision) 기능은 레이아웃과 구조를 이해하는 데 유용합니다.

### 검증 가능한 중간 산출물 생성

에이전트가 복잡하고 개방적인 작업을 수행할 때 실수를 저지를 수 있습니다. "계획-검증-실행(plan-validate-execute)" 패턴은 에이전트가 먼저 구조화된 포맷으로 계획을 생성한 후 실행 전 스크립트로 해당 계획을 검증하여 에러를 조기에 잡아냅니다.

**예시**: 에이전트에게 스프레드시트를 기반으로 PDF의 50개 양식 필드를 업데이트하도록 요청한다고 가정해 봅시다. 검증이 없으면 존재하지 않는 필드를 참조하거나, 충돌하는 값을 생성하거나, 필수 필드를 놓치거나, 업데이트를 잘못 적용할 수 있습니다.

**해결책**: 위에서 보여준 워크플로우 패턴(PDF 양식 작성)을 사용하되, 변경 사항을 적용하기 전에 검증을 거치는 중간 `changes.json` 파일을 추가하세요. 워크플로우는: 분석 → **계획 파일 생성** → **계획 검증** → 실행 → 검증 단계가 됩니다.

**이 패턴이 효과적인 이유:**

* **에러 조기 차단**: 변경 사항을 적용하기 전에 검증기가 문제를 찾아냄
* **기계적 검증 가능**: 스크립트가 객관적인 검증 제공
* **되돌릴 수 있는 계획 수립**: 에이전트는 원본을 건드리지 않고 계획을 반복 개선할 수 있음
* **명확한 디버깅**: 에러 메시지가 특정 문제를 직접 지적함

**사용 시점**: 배치 작업, 파괴적 변경, 복잡한 검증 규칙, 중요도가 높은 작업.

**구현 팁**: 에이전트가 문제를 수정하는 데 도움이 되도록 검증 스크립트가 "Field 'signature_date' not found. Available fields: customer_name, order_total, signature_date_signed"와 같이 구체적인 에러 메시지를 출력하게 하세요.

### 패키지 의존성 패키징

스킬은 플랫폼별 제한 사항이 존재하는 코드 실행 환경에서 실행됩니다:

* **claude.ai**: npm 및 PyPI의 패키지를 설치할 수 있고 GitHub 저장소에서 끌어올 수 있음
* **Anthropic API**: 네트워크 접근 불가 및 런타임 패키지 설치 불가능

SKILL.md에 필요한 패키지 목록을 기재하고 [code execution tool documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)에서 이용 가능한지 확인하세요.

### 실행 환경 (Runtime environment)

스킬은 파일 시스템 접근, bash 명령 및 코드 실행 기능이 포함된 코드 실행 환경에서 작동합니다. 기술적 개념에 대한 설명은 개요 문서의 [The Skills architecture](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)를 참조하세요.

**이것이 스킬 작성에 미치는 영향:**

**에이전트가 스킬에 접근하는 방식:**

1. **메타데이터 사전 로드**: 시작 시 모든 스킬의 YAML frontmatter에 있는 이름과 설명이 시스템 프롬프트로 로드됨
2. **필요 시 파일 읽기**: 에이전트는 필요할 때 파일 읽기 도구를 사용하여 파일 시스템에서 SKILL.md 및 기타 파일에 접근함
3. **효율적인 스크립트 실행**: 유틸리티 스크립트는 전체 내용을 컨텍스트에 로드하지 않고 bash를 통해 실행될 수 있음. 스크립트의 출력 결과만 토큰을 소비함
4. **대용량 파일에 대한 컨텍스트 페널티 없음**: 참조 파일, 데이터 또는 문서는 실제 접근할 때까지 컨텍스트 토큰을 소비하지 않음

* **파일 경로가 중요함**: 에이전트는 파일 시스템처럼 스킬 디렉토리를 탐색합니다. 백슬래시가 아닌 슬래시(`reference/guide.md`)를 사용하세요.
* **파일 이름을 설명적으로 작성**: 내용을 나타내는 이름을 사용하세요: `doc2.md`가 아닌 `form_validation_rules.md`
* **탐색을 위한 구조화**: 도메인이나 기능별로 디렉토리를 구획하세요.
  * 좋음: `reference/finance.md`, `reference/sales.md`
  * 나쁨: `docs/file1.md`, `docs/file2.md`
* **포괄적인 리소스 포함**: 완벽한 API 문서, 풍부한 예시, 대용량 데이터셋을 포함하세요. 접근하기 전까지 컨텍스트 페널티가 없습니다.
* **결정론적 작업에는 스크립트 선호**: 에이전트에게 검증 코드를 생성하도록 요청하기보다 `validate_form.py`를 작성하세요.
* **실행 의도를 명확히 함**:
  * "Run `analyze_form.py` to extract fields" (실행)
  * "See `analyze_form.py` for the extraction algorithm" (참조용으로 읽기)
* **파일 접근 패턴 테스트**: 실제 요청으로 테스트하여 에이전트가 디렉토리 구조를 잘 탐색하는지 검증하세요.

**예시:**

```
bigquery-skill/
├── SKILL.md (overview, points to reference files)
└── reference/
    ├── finance.md (revenue metrics)
    ├── sales.md (pipeline data)
    └── product.md (usage analytics)
```

사용자가 매출에 대해 질문하면 에이전트는 SKILL.md를 읽고 `reference/finance.md` 참조를 확인한 뒤 bash를 호출하여 해당 파일만 읽습니다. sales.md 및 product.md 파일은 파일 시스템에 남아 필요할 때까지 0개의 토큰을 소비합니다. 이 파일 시스템 기반 모델이 바로 단계적 공개를 가능케 하는 핵심입니다. 에이전트는 탐색을 통해 각 작업이 정확히 요구하는 바만 선택적으로 로드할 수 있습니다.

기술 아키텍처에 대한 전체 세부 내용은 스킬 개요 문서의 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)를 참조하세요.

### MCP 툴 참조 (MCP tool references)

스킬에서 MCP (Model Context Protocol) 툴을 사용하는 경우 "tool not found" 에러를 방지하기 위해 항상 풀 네임(fully qualified tool names)을 사용하세요.

**포맷**: `ServerName:tool_name`

**예시**:

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

설명:

* `BigQuery` 및 `GitHub`는 MCP 서버 이름입니다.
* `bigquery_schema` 및 `create_issue`는 해당 서버 내부의 툴 이름입니다.

서버 접두사가 없으면 여러 MCP 서버를 이용할 수 있을 때 에이전트가 툴을 찾지 못할 수 있습니다.

### 툴이 설치되어 있다고 가정하지 말 것

패키지가 이용 가능한 상태라고 지레짐작하지 마세요:

````markdown  theme={null}
**Bad example: Assumes installation**:
"Use the pdf library to process the file."

**Good example: Explicit about dependencies**:
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 기술 노트 (Technical notes)

### YAML frontmatter 요구사항

SKILL.md frontmatter에는 `name` (최대 64자) 및 `description` (최대 1024자) 필드가 필수적입니다. 전체 구조 세부사항은 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)를 참조하세요.

### 토큰 예산 (Token budgets)

최적의 성능을 위해 SKILL.md 본문은 500줄 미만으로 유지하세요. 내용이 이를 초과하면 앞에서 설명한 단계적 공개 패턴을 사용하여 별도의 파일로 분할하세요. 아키텍처 세부사항은 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)를 참조하세요.

## 효과적인 스킬을 위한 체크리스트 (Checklist for effective Skills)

스킬을 공유하기 전에 다음 사항을 검증하세요:

### 핵심 품질

* [ ] 설명이 구체적이며 핵심 용어를 포함함
* [ ] 설명이 스킬이 하는 일과 사용 시점을 모두 포함함
* [ ] SKILL.md 본문이 500줄 미만임
* [ ] 추가 세부사항은 별도 파일에 분리됨 (필요한 경우)
* [ ] 시간에 민감한 정보가 없음 (또는 "구 패턴" 섹션에 포함됨)
* [ ] 전반적으로 용어가 일관됨
* [ ] 예시가 추상적이지 않고 구체적임
* [ ] 파일 참조가 1단계 깊이로 유지됨
* [ ] 단계적 공개 방식이 적절히 활용됨
* [ ] 워크플로우에 명확한 단계가 있음

### 코드 및 스크립트

* [ ] 스크립트가 에이전트에게 미루지 않고 문제를 직접 해결함
* [ ] 에러 처리가 명시적이고 도움이 됨
* [ ] "부두 상수"가 없음 (모든 설정값의 타당성 명시)
* [ ] 필요한 패키지가 지침에 명시되어 있고 사용 가능한지 검증됨
* [ ] 스크립트에 명확한 문서가 포함됨
* [ ] Windows 스타일 경로가 없음 (모두 슬래시 사용)
* [ ] 주요 작업에 대한 검증/확인 단계가 존재함
* [ ] 품질이 중요한 작업에 피드백 루프가 포함됨

### 테스트

* [ ] 3개 이상의 평가 케이스(evaluations)가 작성됨
* [ ] Haiku, Sonnet, Opus로 테스트됨
* [ ] 실제 사용 시나리오로 테스트됨
* [ ] 팀 피드백이 반영됨 (해당하는 경우)

## 다음 단계 (Next steps)

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart">
    Create your first Skill
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="https://code.claude.com/docs/en/skills">
    Create and manage Skills in Claude Code
  </Card>

  <Card title="Use Skills with the API" icon="code" href="https://platform.claude.com/docs/en/build-with-claude/skills-guide">
    Upload and use Skills programmatically
  </Card>
</CardGroup>
