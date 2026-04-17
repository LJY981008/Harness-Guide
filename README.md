# Claude Code 하네스 엔지니어링 규격

> **버전**: v2.2.1 (2026-04-17 기준) · 2026-04 공식 문서(code.claude.com/docs) 최신 반영 · 대형 Phase 독립 파일 분할

Claude Code 기반 프로젝트의 **하네스(harness) 구성 규격과 운영 원칙**을 정의한다. 본 저장소는 "따라하는 튜토리얼"이 아닌 **"이렇게 구성해야 한다"의 규정집**이다.

## v2.2 주요 반영 (2026-04-17)

| 영역 | v2.1 이하 | v2.2 공식 반영 |
|---|---|---|
| Hook 이벤트 | 5종 (PreToolUse·PostToolUse·Stop·InstructionsLoaded 간접) | **25+종** (SubagentStart/Stop, TaskCreated/Completed, PermissionRequest/Denied, PostToolUseFailure, PreCompact/PostCompact, ConfigChange, FileChanged, WorktreeCreate, Elicitation, StopFailure, SessionEnd, TeammateIdle, Notification, CwdChanged 등) |
| Handler 타입 | `command`만 | **4종** (`command`/`http`/`prompt`/`agent`) |
| Async hooks | 없음 | `async: true`, `asyncRewake: true` |
| Defer tool calls | 없음 | `permissionDecision: "defer"` (v2.1.89+) |
| Commands | 별개 | **Skills와 통합** (`.claude/commands/`와 `.claude/skills/` 동일 동작, 충돌 시 skill 우선) |
| Skill frontmatter | 8종 | **+5종** (`user-invocable`, `effort`, `hooks`, `shell`, `model`) |
| Skill 치환자 | `$ARGUMENTS`만 | **+`$ARGUMENTS[N]`/`$N`, `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}`** |
| Skill 인라인 쉘 | 없음 | `` !`command` `` 백틱 / ` ```! ` 블록 preprocessing |
| Live change detection | 없음 | 세션 중 skill 변경 자동 반영 |
| 토큰 예산 | 없음 | `SLASH_COMMAND_TOOL_CHAR_BUDGET` 환경변수 |
| Subagent frontmatter | 3~5종 | **+7종** (`skills`·`initialPrompt`·`memory`·`effort`·`background`·`isolation`·`color`) |
| Subagent skills preload | 없음 | `skills:` 필드로 시작 시 본문 주입 |
| 알려진 Issue | #6699·#27040·#23478·#21858·#12962 | **+#45511** (allow가 deny 덮어쓰기, 2026-04-09) / **+#22055** (Edit/Write가 ask 우회) |

## 정의

**하네스**란 Claude Code가 프로젝트에서 일관된 방식으로 동작하도록 강제·유도하는 **레일 시스템**이다. 다음 5계층과 2개의 부가 레이어로 구성된다.

## 하네스 5계층 구조

| 레이어 | 위치 | 역할 | 계층 자동 발견 |
|---|---|---|---|
| **정책** (CLAUDE.md) | 프로젝트 루트 + 하위 디렉터리 | 모든 세션 system prompt에 주입되는 영구 지시서 | ✅ 상위/하위 모두 |
| **규칙** (Rules) | `.claude/rules/<name>.md` | 경로 스코프된 짧은 규칙. `paths:` frontmatter로 특정 경로 매칭 시만 로드 | ✅ |
| **절차** (Skills) | `.claude/skills/<name>/SKILL.md` | 필요 시 로드되는 절차·매크로. 지원 파일(scripts·references) 동반 가능 | ✅ nested 자동 발견 |
| **강제** (Hooks) | `.claude/hooks/*.sh` + `settings.json` | 결정론적 게이트. 에이전트 자율 준수에 의존하지 않는 물리적 규칙 | ❌ 루트만 |
| **격리** (Subagents) | `.claude/agents/*.md` | 복잡한 반복 분석을 위한 독립 컨텍스트 | ❌ 루트 및 CWD walking up |

부가 레이어:
- **정적분석**: Gradle Spotless + Checkstyle로 규약의 물리적 강제화
- **CI 자동화**: GitLab schedule·Discord webhook 기반 주간 GC (Claude 세션 외부)

## 디렉토리 구조

```
claude-setting/
├── README.md                      # 본 문서 — 하네스 규격 정의
├── HARNESS_SETUP_GUIDE.md         # 규격 개요 + Phase 0/B/D/E/G + 검증·실측 지표 (루트)
├── CLAUDE_MD_GUIDE.md             # CLAUDE.md 정책 레이어 작성 규격
├── SKILLS_GUIDE.md                # Skills 절차 레이어 작성 규격
├── SUBAGENTS_GUIDE.md             # Subagents 격리 레이어 작성 규격
├── guide/                         # v2.2.1 — 대형 Phase 독립 파일 (Phase별 Read 최적화)
│   ├── phase-a-hooks.md           # Hooks 5종 + Hook 이벤트 25+종 레퍼런스
│   ├── phase-c-skills.md          # Skills 심화 (frontmatter·치환·인라인 쉘·live reload)
│   ├── phase-f-static-analysis.md # Spotless + Checkstyle
│   ├── phase-h-weekly-gc.md       # GitLab schedule + Discord
│   └── phase-i-monorepo.md        # 모노레포 스케일링 + Agent Teams
└── templates/
    ├── CLAUDE.md.template
    ├── skills/
    │   ├── SKILL.md.template
    │   ├── operational-wrapper/
    │   │   └── SKILL.md
    │   ├── CODING_RULES.md.template
    │   ├── EXTERNAL_SYSTEMS.md.template
    │   └── PATTERNS.md.template
    └── agents/
        └── vulnerability-analyzer.md.template
```

**분리 원칙** (v2.2.1): 100줄 초과 대형 Phase만 독립 파일로 이동. Phase 0/B/D/E/G(소규모·의미 연계)는 루트 유지 → **한 번의 Read로 루트 + 필요 Phase 1개** 접근 구조. 25,000 토큰 Read 한계 해소.

## 필수 구성요소

다음 3개 계층은 **하네스로 간주되기 위한 최소 요건**이다. 누락 시 "하네스가 있다"고 말할 수 없다.

### Phase 0 · 정책 계층

- `.claude/` 디렉터리 존재
- 루트 `CLAUDE.md` 작성 (150~200줄 이내)
- `.gitignore`에 `settings.local.json`·`hooks/.state/` 등록
- MCP 서버 등록 (prod-db, playwright, sequential-thinking 등 프로젝트 필요에 따라)

### Phase A · 강제 계층 (Hooks 5종)

다음 5개 훅이 `settings.json`에 등록되어야 한다. 각각의 역할은 **대체 불가**이며 누락 시 해당 영역의 레일이 소실된다.

| 훅 | 이벤트 | 역할 |
|---|---|---|
| `pre-bash-guard.sh` | `PreToolUse(Bash)` | 위험 명령의 **물리적 차단** (`permissions.deny` 버그의 유일한 우회 수단) |
| `verify-commit-msg.sh` | `PreToolUse(Bash git commit)` | 커밋 태그 규약 강제 |
| `format-changed-java.sh` | `PostToolUse(Edit\|Write\|MultiEdit)` | 편집 직후 포맷 적용 |
| `post-work-check.sh` | `Stop` | 변경 모듈 한정 컴파일·정적분석 검증 |
| `recommend-agent-on-stop.sh` | `Stop` | 경로 매칭 기반 서브에이전트 추천 (Agent `paths` 미지원 우회) |

### MCP 등록

외부 시스템 연결은 **최소 구성**으로 유지한다. 사용하지 않는 MCP 서버는 system prompt 오염을 유발하므로 제거한다.

## 확장 구성요소

### Phase B · 운영 매크로 (Skills)

**반복되는 분석·조사 작업**을 `/<command>` 슬래시 커맨드로 캡슐화. `context: fork` + `agent:` 조합으로 독립 컨텍스트에서 실행.

### Phase C · 도메인 Skills

코드 규칙·아키텍처 패턴·플로우·스키마 등 **도메인 특화 지식**을 SKILL.md로 정의. `paths:` frontmatter로 자동 로드 조건 지정.

### Phase D · permissions 위생

`settings.local.json`의 allowlist를 **와일드카드로 압축**. 개별 명령 누적 금지.

### Phase E · CLAUDE.md 슬림화

절차성 내용은 Skills로, 경로 스코프 규칙은 Rules로 분리. CLAUDE.md는 **정책과 인덱스**만 유지.

### Phase F · 정적분석 계층

Gradle Spotless(palantir 2.90.0) + Checkstyle(10.21.0)로 코딩 규약의 **컴파일러 레벨 강제**. 위반 0건 달성 후 `ignoreFailures=false` 승격.

### Phase G · 자동 루프 계층

PostToolUse 포맷 + Stop 체인 3단(post-work-check, recommend-agent, spawn-reviewer)으로 **편집-검증-추천 파이프라인** 자동화.

### Phase H · 주기적 청소 계층

GitLab CI schedule + Discord webhook으로 **주간 GC** 실행. Checkstyle 집계·TODO 스캔·agent-memory archive·legacy 참조 추적.

### Phase I · 멀티모듈 스케일링 (10+ 모듈)

- **아키타입 분류**: 60개 모듈도 5~8개 아키타입으로 귀결. 개별 모듈 단위 규약 복제는 금지.
- **`.claude/rules/`**: `paths:` frontmatter로 아키타입별 경로 스코프 규칙.
- **Nested `.claude/skills/`**: 모듈 폴더 내 스킬 배치. 루트 스킬 비대화 방지.
- **`claudeMdExcludes`**: 타 팀 CLAUDE.md 자동 로드 차단.
- **`detect-archetype.sh`**: 훅 스크립트 내 아키타입 판별 공용 함수. 60-way case 금지.
- **`InstructionsLoaded` 훅**: 로드 디버깅 전용. 운영 전 제거 또는 로그 레벨 축소.
- **Agent Teams via Nested Skills**: 복잡한 재사용 파이프라인에 한정. 단순 규칙은 Rules/Skills로 충분하며 토큰·디버깅 비용 주의.

## 운영 원칙

### 계층별 규약

- **CLAUDE.md는 200줄 이내 유지**. 초과 시 절차는 Skills로, 규칙은 `.claude/rules/`로 분리한다.
- **Skill은 공식 폴더 포맷만 사용**: `.claude/skills/<kebab>/SKILL.md` + frontmatter. flat `.md`는 레거시로 간주한다.
- **Hook은 매 툴 호출마다 실행**되므로 경량 유지한다. 성공은 exit 0 무출력, 실패만 표면화한다.
- **Subagent description은 트리거 예시까지 명시**한다. Claude는 description만으로 자동 호출 여부를 판단하므로 모호하면 호출되지 않는다.
- **MCP는 최소 구성**을 원칙으로 한다. 툴 설명이 system prompt를 오염시킨다.
- **permissions는 와일드카드로 압축**한다. `Bash(git :*)` 한 줄이 개별 나열 14줄보다 우선한다.
- **자동 생성 CLAUDE.md를 지양한다**. 조건부 규칙 과다는 효과를 반감시킨다.

### 공식 제약 기반 규약

- **`permissions.deny` 의존 금지** (GitHub #6699, #27040 — v1.0.93+ 버그, 2026-04-17 기준 여전히 OPEN). **PreToolUse + `exit 2`만이 유일한 물리적 차단 수단**이다.
- **`allow`가 `deny`를 덮어쓰는 버그** (GitHub #45511, 2026-04-09). 넓은 allow 패턴(`Bash(git *)`)과 좁은 deny(`Bash(git commit *)`)를 함께 쓰면 deny 무시. **allow를 좁게 설정하거나 pre-bash-guard로 직접 차단**한다.
- **Edit/Write가 `permissions.ask` 우회** (GitHub #22055). 승인 요구 경로는 `PreToolUse(Edit|Write)` 훅으로 물리 차단, ask만 의존 금지.
- **Agent frontmatter `paths` 미지원**. Skill은 `paths` 지원하나 Agent는 description 매칭만 가능. 경로 기반 자동 유도는 `recommend-agent-on-stop.sh` 같은 훅에서 처리한다.
- **PostToolUse/Stop `decision:block`은 유도 수준**이다. LLM이 무시할 수 있으므로 물리적 강제가 필요한 규칙에는 사용하지 않는다. 진짜 강제는 **PreToolUse exit 2**, Checkstyle `ignoreFailures=false`, CI 파이프라인만 가능하다.
- **포맷 베이스라인은 단일 커밋 + `.git-blame-ignore-revs` 등록**을 원칙으로 한다. GitHub/GitLab 17.10+ 자동 인식.
- **`ignoreFailures=true` → `false` 승격은 위반 0건 달성 이후에만** 수행한다. 초기 도입 시 강제 모드는 빌드 마비를 유발한다.
- **주간 GC schedule 활성 시 배포 job 차단 규칙을 필수 추가**한다. `$CI_PIPELINE_SOURCE == "schedule"` 매칭 `when: never` prepend.
- **Discord webhook 실패는 `|| true`로 분리**한다. 알림 실패가 GC 본체 실패로 전파되지 않아야 한다.

### 멀티모듈 규약

- **`settings.json`·hooks·subagents·MCP는 계층 자동 발견 미지원** (Issue #12962 OPEN). 루트 1개 로드를 전제로 설계한다. 훅 스크립트 내부 경로 분기로 모듈별 동작을 구현한다.
- **`.claude/rules/` `paths:` frontmatter는 Read 툴에만 트리거**된다 (Issue #23478 버그). Write 경로도 커버해야 할 규칙은 CLAUDE.md 계층 로드로 병행한다.
- **user-level `~/.claude/rules/`의 `paths:`는 무시**된다 (Issue #21858). user-level rules는 전역 규칙으로만 사용한다.
- **모듈 개별 CLAUDE.md는 예외 규칙 전용**이다. 아키타입으로 귀결되지 않는 legacy/security/외부 API 모듈에만 작성한다. 공통 규칙 복제는 금지한다.
- **Rules vs Skills 분리 기준**: 지원 파일(scripts/·examples/) 필요 시 Skill, 짧은 경로 스코프 규칙 시 Rule. 경계가 흐려지면 Skill로 통합한다.
- **`claudeMdExcludes`는 `settings.local.json`에 정의**한다. 팀 공유 `settings.json`에는 개인 노이즈 제거 규칙을 넣지 않는다.
- **아키타입 명명 규약은 초기에 확정**한다. `modules/payment-core`·`modules/payment-data` 같은 prefix/suffix 패턴을 사후 rename하는 비용은 막대하다.
- **`InstructionsLoaded` 훅은 디버깅 전용**이다. 운영 환경에서는 제거 또는 로그 레벨을 축소한다.

## 참조 문서

- [HARNESS_SETUP_GUIDE.md](HARNESS_SETUP_GUIDE.md) — 규격 개요 + Phase 0/B/D/E/G + 검증·실측 지표
- [guide/phase-a-hooks.md](guide/phase-a-hooks.md) — Phase A: Hooks 5종 + Hook 이벤트 25+종 레퍼런스
- [guide/phase-c-skills.md](guide/phase-c-skills.md) — Phase C: Skills 심화
- [guide/phase-f-static-analysis.md](guide/phase-f-static-analysis.md) — Phase F: Spotless + Checkstyle
- [guide/phase-h-weekly-gc.md](guide/phase-h-weekly-gc.md) — Phase H: 주간 GC
- [guide/phase-i-monorepo.md](guide/phase-i-monorepo.md) — Phase I: 모노레포 스케일링 + Agent Teams
- [CLAUDE_MD_GUIDE.md](CLAUDE_MD_GUIDE.md) — 정책 계층 작성 규격
- [SKILLS_GUIDE.md](SKILLS_GUIDE.md) — 절차 계층 작성 규격
- [SUBAGENTS_GUIDE.md](SUBAGENTS_GUIDE.md) — 격리 계층 작성 규격
- `templates/` — 각 계층별 시작 템플릿

## 공식 문서

- [Hooks](https://code.claude.com/docs/en/hooks)
- [Skills](https://code.claude.com/docs/en/skills)
- [Subagents](https://code.claude.com/docs/en/sub-agents)
- [Settings](https://code.claude.com/docs/en/settings)
- [Memory (CLAUDE.md)](https://code.claude.com/docs/en/memory)
