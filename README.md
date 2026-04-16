# Claude Code 프로젝트 세팅 템플릿

새 프로젝트에 Claude Code 하네스 엔지니어링(CLAUDE.md / Skills / Hooks / Subagents / MCP / permissions / 정적분석 / CI 자동화)을 빠르게 도입하기 위한 가이드와 템플릿 모음.

**2026-04-16 기준 v2.1** · 공식 포맷(code.claude.com/docs) — SKILL.md 폴더 구조, Hooks 네이티브, context:fork 기반 매크로, Spotless/Checkstyle 정적분석, GitLab CI 주간 GC, 모노레포 계층 설정.

**v2 확장** (Phase F/G/H 추가, 2026-04-15):
- Phase F — Spotless + Checkstyle로 CODING_RULES 자동 강제
- Phase G — PostToolUse 포맷 + Stop 체인 자동 루프
- Phase H — GitLab schedule + Discord webhook 주간 GC

**v2.1 확장** (Phase I 추가, 2026-04-16):
- Phase I — 멀티모듈/모노레포 스케일링 (60+ 모듈 대응)
- `.claude/rules/` + `paths:` frontmatter (2025 후반 신규)
- nested `.claude/skills/` 자동 발견 (공식 지원)
- `claudeMdExcludes`로 타 팀 CLAUDE.md 노이즈 제거
- `InstructionsLoaded` 훅으로 로드 디버깅
- `CLAUDE_CODE_NEW_INIT=1` 인터랙티브 초기 세팅
- 아키타입 분류 + `detect-archetype.sh` 공용 함수

**v1.0~v2.1 전 Phase**를 실전 프로젝트(7개 모듈 Spring Boot 멀티모듈)에 적용 완료 — 영상 하네스 5대 요소 전 영역 A급 달성. 본 가이드의 모든 스크립트·설정은 그 결과물을 기반으로 일반화.

## 디렉토리 구조

```
claude-setting/
├── README.md                      # 본 문서 — 전체 개요 + 도입 절차
├── HARNESS_SETUP_GUIDE.md         # ⭐ 2026 기준 하네스 Phase A~E 도입 가이드
├── CLAUDE_MD_GUIDE.md             # CLAUDE.md 작성 가이드
├── SKILLS_GUIDE.md                # .claude/skills/ 작성 가이드
├── SUBAGENTS_GUIDE.md             # .claude/agents/ 서브에이전트 가이드
└── templates/
    ├── CLAUDE.md.template
    ├── skills/
    │   ├── SKILL.md.template               # Skill 프론트매터 템플릿
    │   ├── operational-wrapper/            # Phase B subagent 래핑 매크로 템플릿
    │   │   └── SKILL.md
    │   ├── CODING_RULES.md.template
    │   ├── EXTERNAL_SYSTEMS.md.template
    │   └── PATTERNS.md.template
    └── agents/
        └── vulnerability-analyzer.md.template
```

**주요 구성요소 요약**:
- Hooks 5종 (v2):
  - `pre-bash-guard.sh` — PreToolUse 위험 명령 11종 `exit 2` 차단 (`permissions.deny` 버그 우회)
  - `verify-commit-msg.sh` — 커밋 태그 강제
  - `format-changed-java.sh` — PostToolUse 편집 파일 spotlessApply
  - `post-work-check.sh` — Stop 변경 모듈 한정 compile + checkstyle
  - `recommend-agent-on-stop.sh` — Stop 경로 매칭으로 서브에이전트 추천 (Agent `paths` 미지원 우회)
  - `spawn-reviewer-on-stop.sh` — diff 30줄+ 시 code-reviewer 호출 유도
- `settings.json` — Hooks 등록 완료본
- `settings.local.json` — 와일드카드 압축 (200+줄 → 70줄 내외)
- `build.gradle` — Spotless(palantir 2.90.0) + Checkstyle(10.21.0) 플러그인
- `config/checkstyle/checkstyle.xml` — 최소 규칙 세트
- `scripts/weekly-gc.sh` — 주간 GC (Checkstyle·TODO·archive·legacy)
- 운영 매크로 Skill 5~7종 (예: `/dlq-check`, `/prod-log`, `/pipeline-diagnose` 등 도메인 맞춤)
- 도메인 Skill 15~20종 (코딩 규칙·패턴·플로우·스키마)

## 새 프로젝트 도입 절차

> **전체 절차**: [HARNESS_SETUP_GUIDE.md](HARNESS_SETUP_GUIDE.md) Phase 0~H

### 최소 도입 (30분)
1. **Phase 0** — `.claude/` 생성 + CLAUDE.md 작성 (`templates/CLAUDE.md.template`)
2. **Phase A** — Hooks 5종 복사 + `settings.json`에 등록 (pre-bash-guard 필수 — 유일한 실제 차단 수단)
3. **MCP 등록** — [../skill-setting.md](../skill-setting.md) 참고

### 확장 (여유될 때)
4. **Phase B** — 반복 분석 작업을 `/command` 매크로 skill로 (`templates/skills/operational-wrapper/`)
5. **Phase C** — 도메인 규칙을 SKILL.md 폴더로 ([SKILLS_GUIDE.md](SKILLS_GUIDE.md))
6. **Phase D** — `settings.local.json` 와일드카드 압축 (`templates/settings.local.json.template`)
7. **Phase E** — CLAUDE.md 절차성 섹션을 skill로 분리 (목표 200줄 이내)
8. **서브에이전트** — 같은 분석을 2회 이상 반복할 때만 ([SUBAGENTS_GUIDE.md](SUBAGENTS_GUIDE.md))

### 정적분석·자동화 승격 (v2 추가)
9. **Phase F** — Spotless(palantir 2.90.0) + Checkstyle(10.21.0) 도입 → 위반 제로화 → `ignoreFailures=false` 승격
10. **Phase G** — PostToolUse 자동 포맷 + Stop 체인 3단(post-work-check, recommend-agent, spawn-reviewer)
11. **Phase H** — GitLab CI `weekly-gc` job + schedule 등록 + Discord webhook 알림 (매주 월 03:00 KST 권장)

### 멀티모듈/모노레포 스케일링 (v2.1 추가, 60+ 모듈용)
12. **Phase I** — 아키타입 분류 → `.claude/rules/` 경로 스코프 규칙 → nested `.claude/skills/` 분산 → `claudeMdExcludes` 노이즈 제거 → `detect-archetype.sh` 공용 함수 → `InstructionsLoaded` 로드 디버깅 → **Agent Teams via Nested Skills** (파이프라인 캡슐화)
    - 10개 미만 모노레포는 건너뛰어도 됨
    - 60+ 모듈은 Phase 0과 **병행 설계** 권장 (초기 아키타입 결정)
    - `CLAUDE_CODE_NEW_INIT=1`로 인터랙티브 초기 세팅
    - Agent Teams 패턴은 **복잡한 재사용 파이프라인에만** — 단순 규칙은 Rules/Skills로 충분 (토큰·디버깅 비용 주의)

## 하네스 5계층 + MCP 비교 (v2.1)

| 구성요소 | 위치 | 계층 자동 발견 | 언제 쓰나 | 특징 |
|---|---|---|---|---|
| **정책** (CLAUDE.md) | 프로젝트 루트 + 하위 디렉터리 | ✅ 상위/하위 모두 | 모든 세션 system prompt 주입 | 짧고 핵심만 (150~200줄). 절차는 skill/rule로 위임. |
| **규칙** (Rules, v2.1 신규) | `.claude/rules/<name>.md` (루트/하위) | ✅ | 경로 스코프된 짧은 규칙 | `paths:` frontmatter로 특정 경로 매칭 시만 로드. Skill보다 가벼움. |
| **절차** (Skills) | `.claude/skills/<name>/SKILL.md` | ✅ nested 자동 발견 | 필요할 때만 로드 | frontmatter `description`/`paths`로 자동 매칭. 지원 파일 동반 가능. |
| **강제** (Hooks) | `.claude/hooks/*.sh` + `settings.json` | ❌ 루트만 | 결정론적 게이트 | 에이전트 자율 준수 X. 모듈 분기는 스크립트 내부에서. |
| **격리** (Subagents) | `.claude/agents/*.md` | ❌ 루트만 | 복잡한 반복 분석 | 별도 컨텍스트. description 매칭만(paths 미지원). |
| **도구** (MCP) | `claude mcp add ...` | ❌ 세션 글로벌 | 외부 시스템 연결 | 프로젝트 단위. 많으면 프롬프트 오염. |

## 운영 원칙 (반드시 지킬 것)

- **CLAUDE.md는 짧게**: 200줄 이내. 절차성 내용은 skill로 분리.
- **Skill은 공식 포맷**: `.claude/skills/<kebab>/SKILL.md` 폴더 + frontmatter. flat `.md`는 레거시.
- **Hooks는 매번 실행됨**: 가벼워야 함. 실패 시에만 출력 (성공은 exit 0 조용히).
- **Subagent description을 상세히**: Claude가 자동 호출 여부를 description으로만 판단. 모호하면 호출 안 됨.
- **MCP는 최소로**: 툴 설명이 system prompt에 주입됨. 안 쓰는 서버는 제거.
- **permissions는 와일드카드로**: `Bash(git :*)` 한 줄이 `Bash(git status:*) Bash(git diff:*)...` 14줄보다 낫다.
- **자동 생성 CLAUDE.md 지양**: ETH Zürich 연구상 성능 악화. 조건부 규칙 과다도 효과 반감.

### v2 추가 원칙 (실측으로 확인된 공식 제약 기반)

- **⚠️ `permissions.deny` 버그 회피**: v1.0.93+ 에서 deny 규칙 무시됨 (GitHub #6699, #27040). **PreToolUse + `exit 2`만이 유일한 실제 차단**. deny 리스트에 기대지 말 것.
- **⚠️ Agent `paths` 공식 미지원**: Skill은 paths 지원하지만 Agent는 description 매칭만. 자동 호출 원하면 `recommend-agent-on-stop.sh` 같은 경로 매칭 훅으로 우회.
- **⚠️ PostToolUse/Stop `decision:block`은 유도 수준**: LLM이 무시 가능. 진짜 강제는 PreToolUse exit 2, Checkstyle `ignoreFailures=false`, CI 파이프라인만 가능.
- **포맷 베이스라인은 단일 커밋 + `.git-blame-ignore-revs`**: 대규모 포맷 적용은 blame 오염 격리를 위해 SHA를 등록. GitHub/GitLab 17.10+ 자동 인식.
- **`ignoreFailures=true` → `false` 승격은 위반 0건 달성 후**: 초기 도입은 리포트만, 점진 제로화 후 엄격 모드. 한 번에 도입하면 빌드 마비.
- **주간 GC schedule 시 배포 job 차단 필수**: `$CI_PIPELINE_SOURCE == "schedule"` 시 `when: never` prepend. 없으면 매주 자동 배포 사고.
- **Discord webhook 실패는 `|| true`로 억제**: GC 본체 성공 여부와 알림 실패를 분리. 알림 실패로 GC 전체 실패 전파 금지.

### v2.1 추가 원칙 (멀티모듈/모노레포 기반)

- **⚠️ `settings.json`·hooks·subagents·MCP는 계층 자동 발견 미지원** (Issue #12962 OPEN). 루트 1개만 로드 → 모듈별 분리 **불가능**. 훅 스크립트 내부에서 경로 분기.
- **⚠️ `.claude/rules/` `paths:` 버그** (Issue #23478, #21858): Read 툴에는 트리거되나 Write 툴에는 안 뜸. user-level(`~/.claude/rules/`)은 `paths:` 아예 무시. 중요한 규칙은 CLAUDE.md 계층 로드 병행.
- **모듈 개별 CLAUDE.md는 예외 규칙에만**: 아키타입(5~8개)으로 묶이지 않는 legacy/security/외부 API만 개별. 공통 규칙 60개 파일 복제 금지.
- **Rules vs Skills 분리 기준**: 지원 파일(scripts/, examples/) 필요하면 Skill, 짧은 경로 스코프 규칙이면 Rule. 경계 흐려지면 Skill로 통합.
- **`claudeMdExcludes` 적용은 `settings.local.json` 우선**: 팀 공유 `settings.json`엔 개인 노이즈 제거 규칙 넣지 않음. 팀 공통 제외는 project settings로.
- **아키타입 명명 규칙 조기 결정**: `modules/payment-core`, `modules/payment-data` 같은 prefix 또는 suffix 패턴을 초기에 확정. 후반 rename 비용 막대.
- **`InstructionsLoaded` 훅은 디버깅 전용**: 성능 비용 있음. 세팅 검증 끝나면 제거 또는 로그 레벨 낮추기.
# Harness-Guide
# Harness-Guide
