# CLAUDE.md 작성 가이드

## CLAUDE.md란?

프로젝트 루트의 `CLAUDE.md`는 **모든 대화에 자동으로 컨텍스트로 로드**되는 영구 지시서다. Claude가 그 프로젝트에서 어떻게 행동해야 하는지를 정의한다.

## 핵심 원칙

1. **짧게 유지**: 200줄 이내가 이상적. 길어지면 토큰을 매번 소모한다.
2. **상세는 스킬로 위임**: 디테일한 가이드는 `.claude/skills/*.md`로 빼고 CLAUDE.md에는 "X 작업 전에 Y 문서를 읽어라"는 BLOCKING 지시만 둔다.
3. **명령형으로 작성**: "~하면 좋다" 대신 "~한다", "~하지 않는다"로.
4. **실행 가능한 항목 위주**: 추상적 원칙(예: "코드 품질을 중시")보다 구체적 규칙(예: "wildcard import 금지")이 효과적.

## 권장 섹션 구성

실전 프로젝트 적용에서 검증된 패턴.

```markdown
# CLAUDE.md

## Claude 역할 및 소통 규칙
- 언어 (한국어/영어)
- 역할 (예: 시니어 백엔드 개발자)
- 작업 원칙 (정확성 vs 속도 트레이드오프)
- 작업 완료 후 자동 실행 절차 (테스트 → 빌드 → 커밋 등)
- 커밋 메시지 컨벤션

## ⛔ 필수 문서 참조 규칙 (BLOCKING)
| 작업 | 필수 참조 문서 | 핵심 내용 |
|------|---------------|-----------|
| 코드 작성 | CODING_RULES.md | import 규칙, naming |
| 운영 DB 조회 | EXTERNAL_SYSTEMS.md | SSH 금지, Docker API |
| ... | ... | ... |

> "이 표의 작업을 할 때는 해당 문서를 Read 툴로 먼저 읽어야 한다"고 명시.
> 위반 시 사용자가 거부한다는 점도 적어둔다.

## Project Overview
- 프로젝트 한 줄 설명
- 기술 스택
- 베이스 패키지

## Build & Development Commands
- 자주 쓰는 gradle/npm/docker 명령어 (복붙 가능 형태)

## Module Architecture
- 모듈/패키지 구성표 (모듈 → 역할 → 패턴)
- 핵심 규칙 (예: "REST Controller는 X 모듈에만 존재")

## Event-Driven Architecture (해당 시)
- Consumer ↔ Orchestrator 매핑
- 상태 전이도

## 디버깅 & 로그 분석 규칙
- 근본 원인 분석 절차
- 로그 분석 명령어 패턴

## Skills 문서 (상세 가이드)
| 문서 | 상황 |
| ... | ... |
```

## 안티패턴

- ❌ **장황한 아키텍처 설명**: 5페이지짜리 다이어그램은 스킬로 빼라. CLAUDE.md엔 "아키텍처는 PATTERNS.md 참조" 한 줄만.
- ❌ **추상적 원칙 나열**: "코드는 깨끗하게 짠다" → 행동 변화 없음. "wildcard import 금지" → 실효성 있음.
- ❌ **자주 변하는 정보 박제**: 진행 중 작업, 임시 마이그레이션 메모 등은 plan 파일이나 메모리에.
- ❌ **중복**: 스킬 문서에 있는 내용을 CLAUDE.md에 또 적지 않는다. 링크만.
- ❌ **60개 모듈 전부 열거**: 아키타입(5~8개)으로 묶고 개별 모듈 매핑은 `@docs/modules-index.md`로 분리. (v2.1, 모노레포 대응)

## 멀티모듈/모노레포 배치 (v2.1)

대형 모노레포(10+ 모듈)에서 CLAUDE.md·Rules·Skills 역할 분담:

| 정보 종류 | 어디에 | 이유 |
|---|---|---|
| 공통 역할·커밋·빌드 규칙 | 루트 `CLAUDE.md` (150줄 이내) | 모든 세션 로드 |
| 아키타입별 짧은 규칙 | `.claude/rules/<archetype>.md` + `paths:` | 경로 매칭 시만 로드, 가벼움 |
| 모듈 도메인 설명 | `modules/<name>/CLAUDE.md` | 계층 자동 로드 (on-demand) |
| 절차·슬래시 커맨드·지원 파일 | `.claude/skills/<name>/SKILL.md` | description/paths 매칭, 지원 파일 |
| 모듈 전용 스킬 | `modules/<name>/.claude/skills/` | v2.1 nested 자동 발견 |

**타 팀 CLAUDE.md 노이즈 제거** (`settings.local.json`):

```json
{
  "claudeMdExcludes": [
    "**/modules/other-team/CLAUDE.md",
    "/absolute/path/legacy/.claude/rules/**"
  ]
}
```

자세한 설계는 [HARNESS_SETUP_GUIDE.md — Phase I](HARNESS_SETUP_GUIDE.md) 참고.

**Write 툴 경로 규칙 주의** (Issue #23478 버그): `.claude/rules/`의 `paths:` frontmatter는 Read 툴에만 트리거되고 **Write 툴에는 안 뜸**. 파일 생성 시에도 필수로 적용되어야 하는 규칙은 **CLAUDE.md 계층 로드 또는 Skill**로 병행.

## BLOCKING 표 작성 팁

이 표가 CLAUDE.md의 핵심이다. 다음 형식으로 작성:

```markdown
| 작업 | 필수 참조 문서 | 핵심 내용 |
|------|---------------|-----------|
| **운영 DB 조회/수정** | [EXTERNAL_SYSTEMS](.claude/skills/external-systems/SKILL.md) | SSH 금지, Docker API로 접근 |
```

- "작업" 컬럼: Claude가 시작하려는 행위 유형
- "핵심 내용": 한 줄 요약 (Claude가 이걸 보고 "아 이거 읽어야겠네" 판단)
- 표 위 강조 문구 (v2.2 권장): `**각 Skill은 description/paths 기반 자동 로드됩니다. 자동 로드 안 되면 Skill 툴로 로드하거나 /<skill-name>으로 수동 호출**`
  - 구형 표현 "Read 툴로 먼저 읽어야 합니다"는 **지양** — 2026-04 공식은 Skill 툴 호출 또는 `/<name>`을 표준으로 함

## 자동 실행 절차

작업 완료 후 Claude가 사용자 확인 없이 자동으로 수행할 단계를 명시.

```markdown
## 작업 완료 후
1. 통합 테스트 (`./gradlew :api:test ...`)
2. 빌드 (`./gradlew build -x test`)
3. 커밋 (한글 메시지)
4. **여기서 멈춤** (push는 사용자 요청 시에만)
```

→ "여기서 멈춤"이 중요. 명시하지 않으면 Claude가 멋대로 push할 수 있다.

## 검증

CLAUDE.md를 작성한 뒤:

1. 짧은 작업 한 가지를 시켜본다 (예: 파일 한 줄 수정)
2. Claude가 BLOCKING 표의 skill을 **Skill 툴 또는 `/<name>`으로 호출**하는지 확인 (v2.2: Read 툴이 아님)
3. 자동 호출 안 되면 → skill `description`·`paths` 수정, 또는 CLAUDE.md 표의 트리거 문구 보강
4. 자동 실행 절차가 의도대로 동작하는지 확인 (테스트 → 빌드 → 커밋 → STOP)
5. (v2.2 권장) `What skills are available?` 질의로 로드된 skill 전수 확인 — `SLASH_COMMAND_TOOL_CHAR_BUDGET` 넘으면 description이 잘려 자동 매칭 실패 가능

## 템플릿

[templates/CLAUDE.md.template](templates/CLAUDE.md.template) — 빈 프로젝트용 시작점
