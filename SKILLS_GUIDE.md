# 스킬(Skills) 작성 가이드

> **v2.2 (2026-04-17) 공식 갱신**: Commands → Skills 통합, frontmatter 필드 확장(`user-invocable`·`effort`·`hooks`·`shell`·`model`), 치환자(`$N`·`${CLAUDE_SESSION_ID}`·`${CLAUDE_SKILL_DIR}`), 인라인 쉘(`` !`command` ``), Live change detection, `SLASH_COMMAND_TOOL_CHAR_BUDGET` 예산 제어.

## 2026-04 공식 변경점 (먼저 숙지)

- **Commands → Skills 통합**: `.claude/commands/foo.md`와 `.claude/skills/foo/SKILL.md`는 **둘 다 `/foo`를 만든다**. 이름 충돌 시 **skill 우선**. 신규 기능(supporting files, `paths:`, `disable-model-invocation`, `context: fork`, `hooks:` 등)은 skills로만 제공되므로 **신규는 skills로**. 기존 `.claude/commands/*`는 그대로 동작.
- **Open 표준**: Claude Code skills는 [agentskills.io](https://agentskills.io) 공개 표준을 따른다. Claude Code는 추가 기능(invocation 제어, subagent 실행, 동적 컨텍스트 주입)을 확장.
- **번들 skills**: `/simplify`, `/batch`, `/debug`, `/loop`, `/claude-api` 등은 **프롬프트 기반 번들 skill**. fixed logic 아니라 Claude가 자체 툴로 오케스트레이트.

## 스킬이란?

`.claude/skills/<kebab>/SKILL.md` 파일 **폴더**. CLAUDE.md에서 BLOCKING 규칙으로 참조되거나 `description`·`paths` 자동 매칭으로 **필요할 때만** 컨텍스트에 로드된다. CLAUDE.md를 짧게 유지하면서 도메인 디테일을 깊게 문서화할 수 있는 핵심 장치.

## 언제 스킬을 만드는가

- CLAUDE.md가 200줄을 넘으려 할 때 → 일부를 스킬로 분리
- 같은 설명을 두 번 이상 한 적 있을 때 → 스킬로 박제
- 도메인 특유 규칙(외부 시스템, 비즈니스 플로우 등)이 있을 때
- 신규 작업자가 헷갈려할 만한 부분

## 스킬 분류 (검증된 카테고리)

실전 멀티모듈 Spring Boot 프로젝트에서 적용된 15~20종 스킬의 카테고리:

| 카테고리 | 예시 파일 | 용도 |
|---|---|---|
| **코딩 규칙** | `coding-rules/SKILL.md` | import, naming, API 작성 규칙 |
| **아키텍처 패턴** | `patterns/SKILL.md` | 모듈 경계, TX 규칙, 의존성 방향 |
| **외부 시스템** | `external-systems/SKILL.md` | 외부 API/DB 접근법, 서버 IP |
| **데이터 플로우** | `<domain>-flow/SKILL.md`, `flow-diagrams/SKILL.md` | 핵심 비즈니스 플로우 |
| **상태 머신** | `status-lifecycle/SKILL.md` | enum 전이도, 복구 로직 |
| **장애 대응** | `consumer-dlq/SKILL.md`, `integrity-check/SKILL.md` | DLQ, 복구 절차 |
| **체크리스트** | `checklists/SKILL.md` | 신규 기능 추가 시 누락 방지 |
| **빠른 참조** | `class-index/SKILL.md`, `quick-reference/SKILL.md` | 클래스 위치, 자주 쓰는 컴포넌트 |
| **명세서** | `<message>-spec/SKILL.md`, `jsonb-schema/SKILL.md` | 메시지 포맷, 스키마 |
| **시나리오** | `scenarios/SKILL.md` | 자주 발생하는 케이스별 대응 |

새 프로젝트는 처음부터 15+개 다 만들 필요 없다. **3개부터 시작:**
1. `CODING_RULES.md`
2. `EXTERNAL_SYSTEMS.md`
3. `PATTERNS.md`

## Frontmatter 전체 필드 (2026-04 공식, v2.2)

`SKILL.md` 상단 `---` 블록. `description`만 권장, 나머지는 선택.

| 필드 | 용도 | 제약 / 기본값 |
|---|---|---|
| `name` | 디렉터리명 대신 표시. `/name` 슬래시 커맨드 이름 | 최대 64자, lowercase+숫자+hyphen |
| `description` | Claude의 자동 호출 판단 근거. **앞쪽에 키워드** | 최대 1,024자. skill listing에서 1,536자 cap |
| `when_to_use` | 추가 트리거 문구 (description에 append) | 1,536자 예산 공유 |
| `argument-hint` | 자동완성 힌트 | `[issue-number]` 등 |
| `disable-model-invocation` | Claude 자동 호출 차단 (`/name` 수동만) | default `false` |
| `user-invocable` | `false` 시 `/` 메뉴에서 숨김 (백그라운드 지식용) | default `true` (v2.2) |
| `allowed-tools` | skill 활성 중 무승인 허용 툴 | space/YAML 리스트 |
| `model` | 이 skill 활성 시 사용할 모델 override | v2.2 신규 |
| `effort` | 추론 effort | `low`/`medium`/`high`/`xhigh`/`max` (v2.2 신규) |
| `context` | `fork` 지정 시 서브에이전트에서 실행 | |
| `agent` | `context: fork` 시 subagent 타입 | `Explore`/`Plan`/`general-purpose` 또는 커스텀 |
| `hooks` | 이 skill 라이프사이클 전용 훅 | `PreToolUse`/`PostToolUse`/`Stop` (v2.2) |
| `paths` | 자동 로드 glob | `"**/*.java"` 또는 YAML 리스트 |
| `shell` | 인라인 쉘 실행 셸 | `bash`(기본)/`powershell` (v2.2 신규. PS는 `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` 필요) |

### invocation 제어 매트릭스

| frontmatter | 사용자 호출 | Claude 호출 | 컨텍스트 로드 시점 |
|---|---|---|---|
| (기본) | ✅ | ✅ | description 상시, 전체 본문은 호출 시 |
| `disable-model-invocation: true` | ✅ | ❌ | description 비포함, 전체 본문은 사용자 호출 시 |
| `user-invocable: false` | ❌ | ✅ | description 상시, 전체 본문은 Claude 호출 시 |

> **주의**: `user-invocable: false`는 메뉴 숨김이지 Skill 툴 호출 차단 아님. Claude의 프로그래매틱 호출 차단은 `disable-model-invocation: true`.

## 문자열 치환 (v2.2 공식)

| 변수 | 의미 |
|---|---|
| `$ARGUMENTS` | 전체 인자 문자열. 본문에 없으면 `ARGUMENTS: <값>`이 끝에 자동 append |
| `$ARGUMENTS[N]` / `$N` | N번째 인자 (0-based). shell-style quoting — `"hello world"`는 1개 인자 |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID. 세션별 로그 분리에 활용 |
| `${CLAUDE_SKILL_DIR}` | 이 SKILL.md 디렉터리 절대경로. 번들 스크립트 참조 시 CWD 무관 |

**컴포넌트 이관 예시 (`$N` 축약)**:
```yaml
---
name: migrate-component
description: 컴포넌트 프레임워크 이관
---
$0 컴포넌트를 $1에서 $2로 이관. 기존 동작·테스트 보존.
```
→ `/migrate-component SearchBar React Vue` → `$0=SearchBar $1=React $2=Vue`

## 인라인 쉘 실행 (v2.2 공식)

`` !`command` `` 백틱 또는 ` ```! ` 블록은 **Claude가 보기 전에 쉘에서 실행**되어 결과로 치환된다. 동적 컨텍스트 주입용 공식 preprocessing.

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

**멀티라인**:
````markdown
## Environment
```!
node --version
npm --version
git status --short
```
````

**정책 비활성화**: `settings.json`의 `"disableSkillShellExecution": true` → `[shell command execution disabled by policy]`로 치환. Managed settings로 조직 전체 차단 가능. **번들/managed skills는 영향 없음**.

## Live Change Detection & 예산 제어 (v2.2 공식)

- **Live reload**: `~/.claude/skills/`, 프로젝트 `.claude/skills/`, `--add-dir`의 `.claude/skills/`에서 파일 **추가/수정/삭제 → 세션 중 반영** (재시작 불필요).
- **단, top-level skills 디렉터리 신규 생성**은 재시작 필요 (파일 감시 대상에 포함되지 않음).

**토큰 예산** (`SLASH_COMMAND_TOOL_CHAR_BUDGET`):
- skill 이름은 항상 포함. description은 예산에 맞춰 **앞쪽부터 보존하고 뒤를 자름**
- 기본 8,000자 또는 컨텍스트 윈도우의 1% 중 큰 쪽
- 확장: `export SLASH_COMMAND_TOOL_CHAR_BUDGET=20000`
- 키워드가 뒤에 있으면 cut-off로 매칭 실패 → **핵심은 앞쪽**

**Auto-compaction**:
- 호출된 skill 본문은 대화에 **1메시지로 삽입 후 재로드 안 됨** → "standing instruction"으로 작성
- 압축 시 최근 호출 skill을 각 5,000토큰까지 보존, 합산 25,000토큰 예산. 초과분은 오래된 것부터 제거
- 영향력이 약해지면 **재호출로 갱신**

## Subagent와의 결합 (v2.2 공식화)

두 방향이 모두 공식 지원됨:

| 패턴 | System prompt | Task | 추가 로드 |
|---|---|---|---|
| **Skill + `context: fork`** | agent 타입 기본 (`Explore` 등) | SKILL.md 본문 | CLAUDE.md |
| **Subagent + `skills:`** | subagent markdown 본문 | Claude 위임 메시지 | preloaded skills 본문 + CLAUDE.md |

- `context: fork`는 **skill이 드라이버**. 격리된 agent에서 SKILL 실행
- `skills:` (subagent frontmatter)은 **subagent가 드라이버**. 시작 시 skill 본문 주입

## 스킬 작성 원칙

### 1. 한 파일 = 한 주제
파일이 여러 주제를 다루면 BLOCKING 표에서 매핑하기 어렵다. 분리하라.

### 2. 코드 예시는 실측 기반
가짜 예시(`foo`, `bar`) 대신 실제 프로젝트 코드 인용. 변경 시 같이 업데이트하기 쉽다.

### 3. 안티패턴 명시
"X를 하지 마라" 만 적지 말고, **왜** 안 되는지 사례로 적어둔다. Claude가 엣지 케이스에서 판단할 수 있게.

```markdown
## ❌ 하지 말아야 할 것

### SSH로 운영 DB 접근
**금지 이유**: 운영 서버는 SSH 비활성. 무리하게 시도하면 보안 알림 발생.
**대신**: Docker 원격 API로 접근 (`DOCKER_HOST=tcp://...`)
```

### 4. 명령어는 복붙 가능 형태
`# user/password를 채우세요` 같은 placeholder는 OK. 하지만 명령어 구조 자체는 그대로 실행 가능해야 함.

### 5. 길이 제한 없음
스킬은 필요할 때만 로드되므로 길어도 OK. 다만 한 파일이 1000줄을 넘으면 분리 검토.

## CLAUDE.md와의 연결

스킬을 만들면 반드시 CLAUDE.md의 BLOCKING 표에 추가:

```markdown
| **외부 DB 작업** | [external-systems](.claude/skills/external-systems/SKILL.md) | SSH 금지, Docker API |
```

표에 없으면 Claude가 그 스킬의 존재를 모른다. v2.2부터는 `description`·`paths` 기반 자동 로드도 있으므로 BLOCKING 표는 **명시적 트리거**(사용자가 언급하지 않아도 강제 참조) 용도로 사용.

## 유지보수

- **스킬은 자주 부패한다**: 코드/구조가 변하면 스킬도 같이 갱신해야 함
- **거짓 정보는 차라리 삭제**: 오래된 정보는 추측 근거가 되어 더 위험
- **PR마다 영향받는 스킬 갱신**: 새 패턴 도입 시 PATTERNS.md 같이 업데이트

## 템플릿

- [templates/skills/CODING_RULES.md.template](templates/skills/CODING_RULES.md.template)
- [templates/skills/EXTERNAL_SYSTEMS.md.template](templates/skills/EXTERNAL_SYSTEMS.md.template)
- [templates/skills/PATTERNS.md.template](templates/skills/PATTERNS.md.template)
