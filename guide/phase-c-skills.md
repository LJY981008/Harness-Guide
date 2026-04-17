# Phase C — 도메인 Skills + frontmatter 심화

> ← **[HARNESS_SETUP_GUIDE.md](../HARNESS_SETUP_GUIDE.md)** (상위 가이드로 돌아가기)
>
> **관련 Phase**: [A (Hooks)](phase-a-hooks.md) · [B (운영 매크로)](../HARNESS_SETUP_GUIDE.md#phase-b-운영-매크로-skills) · [E (CLAUDE.md 슬림화)](../HARNESS_SETUP_GUIDE.md#phase-e-claude.md-슬림화) · [I (모노레포)](phase-i-monorepo.md)
>
> **역할**: 프로젝트 도메인 지식(코딩 규칙·아키텍처 패턴·플로우·스키마)을 Skill로 격리. `description`·`paths` 자동 매칭으로 필요할 때만 로드되어 컨텍스트 절약.
>
> **상세 가이드**: [SKILLS_GUIDE.md](../SKILLS_GUIDE.md) — 개별 skill 작성 규격

## 목차
- [C-1. 기존 flat `.md`가 있으면 마이그레이션](#c-1-기존-flat-md가-있으면-마이그레이션)
- [C-2. 신규 skill 작성](#c-2-신규-skill-작성)
- [C-3. `paths` glob 활용](#c-3-paths-glob-활용)
- [C-4. description 작성 팁](#c-4-description-작성-팁)
- [C-4-1. Skill frontmatter 전체 필드 (2026-04 공식)](#c-4-1-skill-frontmatter-전체-필드-2026-04-공식-v22-갱신)
- [C-4-2. 문자열 치환 (v2.2)](#c-4-2-문자열-치환-v22-신규-공식화)
- [C-4-3. 인라인 쉘 실행 (v2.2)](#c-4-3-인라인-쉘-실행-v22-신규)
- [C-4-4. Live change detection & 예산 제어 (v2.2)](#c-4-4-live-change-detection--예산-제어-v22-신규)
- [C-5. Nested `.claude/skills/` 자동 발견](#c-5-nested-claudeskills-자동-발견-v21-반영)

---

## Phase C: 도메인 Skills

### C-1. 기존 flat `.md`가 있으면 마이그레이션

`.claude/skills/CODING_RULES.md` → `.claude/skills/coding-rules/SKILL.md` 로 이동 + frontmatter 추가.

일괄 마이그레이션 스크립트 샘플:

```bash
#!/usr/bin/env bash
# migrate_skills.sh
set -euo pipefail

entries=(
  'CODING_RULES.md|coding-rules|Java 코드 규칙 — import/Entity/DTO/QueryDSL.|**/*.java'
  'LOGGING_RULES.md|logging-rules|로그 포맷·이모지·금지 패턴.|**/*.java'
  'EXTERNAL_SYSTEMS.md|external-systems|서버 IP/Docker/psql/외부 API 연동.|'
)

for entry in "${entries[@]}"; do
  IFS='|' read -r OLD KEBAB DESC PATHS <<< "$entry"
  NEW=".claude/skills/$KEBAB/SKILL.md"
  mkdir -p "$(dirname "$NEW")"
  git mv ".claude/skills/$OLD" "$NEW"
  TMP=$(mktemp)
  {
    printf '%s\n' '---'
    printf 'name: %s\n' "$KEBAB"
    printf 'description: %s\n' "$DESC"
    [[ -n "$PATHS" ]] && printf 'paths: "%s"\n' "$PATHS"
    printf '%s\n\n' '---'
    cat "$NEW"
  } > "$TMP"
  mv "$TMP" "$NEW"
done
```

### C-2. 신규 skill 작성

[SKILLS_GUIDE.md](../SKILLS_GUIDE.md) 참고. 공식 포맷:

```
.claude/skills/
└── <kebab-name>/
    ├── SKILL.md           # 필수. frontmatter + 본문
    ├── references/        # 선택. 상세 문서들
    ├── examples/          # 선택. 출력 예시
    └── scripts/           # 선택. 실행 스크립트
```

템플릿: [../templates/skills/SKILL.md.template](../templates/skills/SKILL.md.template)

**실제 적용 사례 규모감**: 대형 프로젝트 기준 도메인 skill 15~20종 + 운영 매크로 5~7종 + 공용 skill 3~5종 정도가 실용적 상한. 너무 많으면 description 매칭 품질 하락.

### C-3. `paths` glob 활용

파일 작업 시 자동 로드:
```yaml
paths: "**/*.java"                    # Java 작업 시
paths: "**/*Repository.java"          # Repository만
paths: ["**/*.java", "**/*.kt"]       # 여러 패턴
paths: "modules/payment-*/**"         # 특정 모듈 묶음 (v2.1)
```

### C-4. description 작성 팁

- **앞쪽에 핵심 사용 케이스** (1536자 내에서 잘릴 수 있음)
- **트리거 문구 포함**: "로그 규약 알려줘", "DLQ 상태" 등 자연어 예시
- **`when_to_use`** 필드로 보강 가능

### C-4-1. Skill frontmatter 전체 필드 (2026-04 공식, v2.2 갱신)

공식 문서(`code.claude.com/docs/en/skills`)의 frontmatter 스키마 전체. 필수는 없고 `description`만 권장.

| 필드 | 용도 | 제약 / 기본값 |
|---|---|---|
| `name` | 디렉터리명 대신 표시명. `/name`의 이름이 됨 | 64자, lowercase+숫자+hyphen |
| `description` | Claude가 자동 호출 판단에 사용. 앞쪽에 핵심 케이스 | 최대 1,024자, 1,536자 skill listing 예산 |
| `when_to_use` | 추가 트리거 문구 (description에 append) | 1,536자 예산 공유 |
| `argument-hint` | 자동완성 힌트 | `[issue-number]` 형태 |
| `disable-model-invocation` | Claude 자동 호출 차단 (수동 `/` 전용) | default `false` |
| `user-invocable` | `false` 시 `/` 메뉴에서 숨김 (백그라운드 지식) | default `true` (v2.2 신규) |
| `allowed-tools` | skill 활성 중 무승인 허용 툴 | space/YAML 리스트 |
| `model` | skill 활성 시 사용할 모델 override | v2.2 신규 |
| `effort` | 추론 effort 레벨 | `low`\|`medium`\|`high`\|`xhigh`\|`max` (v2.2 신규) |
| `context` | `fork` 지정 시 서브에이전트에서 실행 | |
| `agent` | `context: fork` 시 사용할 subagent 타입 | 빌트인(`Explore`/`Plan`/`general-purpose`) 또는 커스텀 |
| `hooks` | 이 skill 라이프사이클 전용 훅 | PreToolUse/PostToolUse/Stop 등 (v2.2 신규) |
| `paths` | 자동 로드 glob | `"**/*.java"` 또는 YAML 리스트 |
| `shell` | 인라인 쉘 실행용 셸 | `bash`(기본) 또는 `powershell`. powershell은 `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` 필요 (v2.2 신규) |

**공식 노트**:
- **description/when_to_use 합산 1,536자 초과분은 skill listing에서 잘림** → 키워드 앞쪽 배치 필수
- description 없으면 markdown 본문 첫 문단이 대체로 사용됨
- `allowed-tools`는 **추가 허용**이지 제한 아님. 제한은 `/permissions`의 deny 규칙으로

### C-4-2. 문자열 치환 (v2.2 신규 공식화)

Skill 본문에서 사용할 수 있는 공식 치환 변수:

| 변수 | 의미 | 예시 |
|---|---|---|
| `$ARGUMENTS` | 전체 인자 문자열 | `/fix-issue 123 P0` → `"123 P0"` |
| `$ARGUMENTS[N]` | N번째 인자 (0-based, shell quoting) | `/cmd "hello world" second` → `$ARGUMENTS[0]` = `hello world` |
| `$N` | `$ARGUMENTS[N]` 축약형 | `$0`·`$1` 등 |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID | 세션별 로그 분리 |
| `${CLAUDE_SKILL_DIR}` | 이 skill의 `SKILL.md` 디렉터리 (플러그인 skill은 플러그인 서브폴더) | 번들된 스크립트 절대경로 |

**세션 로거 예시**:
```yaml
---
name: session-logger
description: Log activity for this session
---
Log the following to logs/${CLAUDE_SESSION_ID}.log:

$ARGUMENTS
```

**컴포넌트 마이그레이션 예시 ($N 축약)**:
```yaml
---
name: migrate-component
description: 컴포넌트 프레임워크 이관
---
$0 컴포넌트를 $1에서 $2로 이관. 기존 동작·테스트 보존.
```
→ `/migrate-component SearchBar React Vue` → `$0=SearchBar, $1=React, $2=Vue`

**⚠️ `$ARGUMENTS` 누락 시**: 인자가 있는데 본문에 치환자가 없으면 Claude Code가 `ARGUMENTS: <값>`을 skill 끝에 자동 append.

### C-4-3. 인라인 쉘 실행 (v2.2 신규)

Skill 본문의 `` !`command` `` 백틱은 **Claude가 보기 전에 쉘에서 실행**되어 결과로 치환된다. Skill에 동적 컨텍스트를 주입하는 공식 방법.

**PR 요약 예시**:
```yaml
---
name: pr-summary
description: Summarize current PR
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## PR 컨텍스트
- Diff: !`gh pr diff`
- 코멘트: !`gh pr view --comments`
- 변경 파일: !`gh pr diff --name-only`

## 작업
위 컨텍스트 기반으로 PR 요약...
```

**멀티라인 블록**:
````markdown
## Environment
```!
node --version
npm --version
git status --short
```
````

**정책적 비활성화**: `settings.json`의 `"disableSkillShellExecution": true` → `[shell command execution disabled by policy]`로 치환. Managed settings에서 조직 전체 차단 가능. **bundled/managed skills는 영향 없음**.

### C-4-4. Live change detection & 예산 제어 (v2.2 신규)

**Live change detection**:
- `~/.claude/skills/`, 프로젝트 `.claude/skills/`, `--add-dir`의 `.claude/skills/` 파일 추가/수정/삭제 → **세션 재시작 없이 반영**
- 단, 세션 시작 시 **존재하지 않았던** top-level skills 디렉터리를 새로 만들면 **재시작 필요** (파일 감시 대상에 추가되지 않음)

**토큰 예산 제어** (`SLASH_COMMAND_TOOL_CHAR_BUDGET`):
- Skill 디스크립션은 컨텍스트에 상주하지만, 개수가 많으면 예산에 맞춰 축약
- 기본 **8,000자 또는 컨텍스트 윈도우의 1%** 중 큰 쪽
- 축약 시 description 앞쪽이 보존되므로 **핵심 키워드 앞 배치** 필수
- 환경변수로 확대: `export SLASH_COMMAND_TOOL_CHAR_BUDGET=20000`

**컴팩션 동작**:
- 호출된 skill의 본문은 대화 1메시지로 삽입되어 **세션 내 재참조 안 됨** → skill은 "standing instruction"으로 작성
- auto-compaction 시 최근 호출 skill들을 각 5,000토큰까지 보존, 합산 25,000토큰 예산. 초과분은 오래된 것부터 제거
- skill 영향력이 희미해지면 **재호출**로 갱신

### C-5. Nested `.claude/skills/` 자동 발견 (v2.1 반영)

**공식 지원** (2025 후반 추가): 루트뿐 아니라 **서브디렉터리의 `.claude/skills/`도 자동 발견**됨.

```
<repo>/
├── .claude/skills/               ← 루트 (항상 로드)
│   └── coding-rules/SKILL.md
├── modules/payment/
│   └── .claude/skills/           ← payment 파일 작업 시에만 발견
│       └── payment-deploy/SKILL.md
└── modules/web/
    └── .claude/skills/
        └── next-conventions/SKILL.md
```

공식 문서 원문:
> *When you work with files in subdirectories, Claude Code automatically discovers skills from nested `.claude/skills/` directories.*

**활용 패턴**:
- 대형 모노레포에서 **모듈별 Skill 분산 배치** → 루트 skills 비대화 방지
- `paths` glob보다 **물리적 분리가 명확** → 팀 소유권·CODEOWNERS 매핑 쉬움
- 루트 skills에는 전역 규칙, 모듈 skills에는 도메인 규칙

**주의사항**:
- Subagents(`.claude/agents/`), Commands, Output styles는 **여전히 루트만 로드**. Skills 예외적으로만 nested 지원.
- `--add-dir` 추가 디렉터리의 `.claude/skills/`도 자동 로드 (다른 하위 설정은 제외)
- [Phase I](phase-i-monorepo.md)에서 대규모 활용 전략 상세.
