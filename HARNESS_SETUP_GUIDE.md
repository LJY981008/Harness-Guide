# Harness Setup Guide (2026-04 기준, v2.2.2)

> 새 프로젝트에 Claude Code 하네스 엔지니어링을 도입하는 완전 가이드. 2026 공식 포맷(code.claude.com/docs) 기준.
>
> **레퍼런스 구현**: 가이드의 모든 스크립트·설정은 실전 프로젝트에 적용해 **현재 동작 중인 것**을 기반으로 작성됨. `templates/` 디렉터리 복사 후 프로젝트에 맞춰 조정.
>
> **v2 변경점** (2026-04-15): Phase F(정적분석) · Phase G(자동 루프) · Phase H(주간 GC) 추가. Phase A Hook 3종 → 5종으로 확장. `permissions.deny` 버그와 Agent `paths` 미지원 등 **실측으로 확인된 공식 제약** 명시.
>
> **v2.1 변경점** (2026-04-16): 대형 모노레포(60+ 모듈) 대응 Phase I 추가. 2025 후반~2026초 신규 기능 반영 — `.claude/rules/` + `paths:` frontmatter, nested `.claude/skills/` 자동 발견, `claudeMdExcludes`, `InstructionsLoaded` 훅, `CLAUDE_CODE_NEW_INIT=1` 인터랙티브 /init. 하네스 4계층 → **5계층** (rules 레이어 추가).
>
> **v2.2.2 변경점** (2026-04-20): 공식 문서 및 GitHub 이슈 재검증 반영.
> - **신규 반영**: `/hooks` 메뉴 (읽기 전용 훅 브라우저, 출처 라벨 표시) — "적용 후 검증" 섹션에 추가
> - **신규 반영**: `CLAUDE_ENV_FILE` 환경변수 영속화 (SessionStart/CwdChanged/FileChanged 전용) — `guide/phase-a-hooks.md` A-6-6 신설
> - **이슈 상태 재확인**: #21858 (user-level `paths:` 무시)이 **2026-03-24 completed 종료** → 제약 표에서 "영구 버그" 표기 제거
> - **이슈 상태 재확인**: #23478 (Write에 `paths:` 미적용)이 **closed as not planned** → "영구 제약으로 확정" 명시
> - **이슈 상태 재확인**: #6699(2025-09-02 completed 종료) → #27040(2026-02-20 오픈)에서 재발. 제약 표 출처를 #27040 단독 + #6699는 선행/재발 주석으로 정리
>
> **v2.2.1 변경점** (2026-04-17, 후속): 대형 Phase 독립 파일 분할 — **루트 가이드 1,847줄 → 약 561줄**.
> - `guide/phase-a-hooks.md` (Hook 5종 + 이벤트 레퍼런스, ~326줄)
> - `guide/phase-c-skills.md` (Skill 심화, ~234줄)
> - `guide/phase-f-static-analysis.md` (Spotless+Checkstyle, ~186줄)
> - `guide/phase-h-weekly-gc.md` (주간 GC, ~151줄)
> - `guide/phase-i-monorepo.md` (모노레포+Agent Teams, ~569줄)
> - Phase 0/B/D/E/G·검증·실측 지표는 루트 유지 (소규모·의미 연계)
> - **Claude Read 25,000 토큰 한계 해소** + Phase별 독립 도입 워크플로우와 1:1 매칭
>
> **v2.2 변경점** (2026-04-17): 2026-04 공식 문서 최신화 반영.
> - **Hook 이벤트 25+종 레퍼런스 표** 추가 (기존 5종에서 SubagentStart/Stop, TaskCreated/Completed, PermissionRequest/Denied, PostToolUseFailure, PreCompact/PostCompact, ConfigChange, FileChanged, WorktreeCreate/Remove, Elicitation, StopFailure, SessionEnd, Notification, TeammateIdle, CwdChanged 등)
> - **Handler 4종 정식화**: `command` / `http` / `prompt` / `agent` (이전에는 `command`만 사용)
> - **Async hooks**: `async: true` / `asyncRewake: true` — 장시간 작업 비차단 실행
> - **Defer tool calls** (v2.1.89+): `permissionDecision: "defer"` — SDK·커스텀 UI용 일시 보류
> - **Commands → Skills 통합**: `.claude/commands/`와 `.claude/skills/`가 동일하게 동작. 충돌 시 skill 우선. 신규 기능은 skills로만 제공되므로 **skills 우선 사용 원칙**.
> - **Skill frontmatter 확장**: `user-invocable` (/ 메뉴 숨김), `effort` (low~max), `hooks:` (skill 라이프사이클 전용), `shell` (bash/powershell), `model` (skill별 모델 override)
> - **문자열 치환 확장**: `$ARGUMENTS[N]`·`$N` 인덱스 액세스, `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}`
> - **인라인 쉘 실행**: `` !`command` `` 백틱 / ` ```! ` 블록 — skill 전달 전 preprocessing
> - **Live change detection**: 세션 중 skill 파일 변경 자동 반영 (재시작 불필요)
> - **`SLASH_COMMAND_TOOL_CHAR_BUDGET`**: skill 디스크립션 예산 환경변수 (기본 8,000자 또는 컨텍스트의 1%)
> - **Subagent frontmatter 확장**: `skills` (preload), `initialPrompt`, `memory`, `effort`, `background`, `isolation`, `color`
> - **신규 공식 Issue 반영**: [#45511](https://github.com/anthropics/claude-code/issues/45511) (allow가 deny 덮어쓰기), [#22055](https://github.com/anthropics/claude-code/issues/22055) (Edit/Write가 ask 우회)

---

## ⚠️ 알려진 공식 제약 (먼저 숙지)

이 제약은 실제 프로젝트 적용 과정에서 **실측으로 확인**됐으며, 하네스 설계 시 반드시 반영해야 한다:

| 제약 | 현상 | 우회 방법 |
|---|---|---|
| **`permissions.deny` 필드 버그** (GitHub [#27040](https://github.com/anthropics/claude-code/issues/27040) OPEN. 선행 [#6699](https://github.com/anthropics/claude-code/issues/6699)는 2025-09-02 completed 종료됐으나 #27040(2026-02-20 오픈)에서 재발 확인) | `deny` 규칙 무시됨 — 차단되어야 할 명령이 실행됨 | **PreToolUse hook + `exit 2`**가 유일한 실제 차단 수단 (Phase A-0 참고) |
| **Agent frontmatter `paths` 공식 미지원** | Skill은 `paths` glob으로 자동 로드되나 Agent는 `description` 매칭만 지원 | `recommend-agent-on-stop.sh`로 경로 키워드 기반 추천 주입 (Phase G) |
| **PostToolUse/Stop `decision:block`은 피드백만** | 차단이 아닌 "Claude에게 메시지 주입" 수준. LLM이 무시 가능 | 진짜 물리적 강제는 **PreToolUse + exit 2**, CI 파이프라인, Checkstyle `ignoreFailures=false`만 가능 |
| **모노레포 계층 설정 지원 불완전** (GitHub [#12962](https://github.com/anthropics/claude-code/issues/12962) OPEN, [#37344](https://github.com/anthropics/claude-code/issues/37344) 2026-03-25 중복 종료) | 계층 자동 발견 지원 여부가 요소별 상이. `settings.json`/hooks/MCP는 **루트만**, `CLAUDE.md`/`skills`/`rules`는 **양방향 계층 지원**, subagents는 **CWD 기준 walking up만** | Phase I 참고 — hooks는 루트 몰빵 + 경로 분기 스크립트, skills·rules는 모듈 폴더 분산, subagents는 루트 등록 원칙 (모듈 단위 cd 워크플로우일 때만 모듈 `.claude/agents/` 발견) |
| **`paths:` frontmatter Write 미지원** (GitHub [#23478](https://github.com/anthropics/claude-code/issues/23478) **closed as not planned** — Anthropic이 공식적으로 수정 계획 없음 → 영구 제약으로 확정) | `.claude/rules/`의 `paths:` 매칭이 **Read 툴에만 트리거, Write에는 안 됨**. (참고: user-level `~/.claude/rules/` `paths:` 무시 버그 [#21858](https://github.com/anthropics/claude-code/issues/21858)은 2026-03-24 completed 종료) | 중요한 모듈 규칙은 Write 경로도 커버되도록 CLAUDE.md 계층 로드 병행 |
| **Subagent 공식 발견 위치 5곳만 유효** | managed settings / `--agents` CLI / `.claude/agents/` (루트 또는 CWD 상위) / `~/.claude/agents/` / plugin `agents/`. **스킬 폴더 내부 `.md`는 subagent로 인식 안 됨** | 스킬 폴더 안 `.md`는 "프롬프트 파일"로만 사용 가능. frontmatter `hooks:` 등을 정규 동작시키려면 **루트 `.claude/agents/`에 등록 필수** |
| **`allow` 패턴이 `deny`를 덮어쓰는 버그** (GitHub [#45511](https://github.com/anthropics/claude-code/issues/45511), 2026-04-09) | `Bash(git *)` allow + `Bash(git commit *)` deny 공존 시 deny 무시, `git commit`이 자동 허용됨 (duplicate로 종료됐으나 현상 확인됨) | **allow 패턴을 최대한 좁게** (`Bash(git status:*)` 등 세분화), 또는 **pre-bash-guard로 직접 차단** (PreToolUse + exit 2) |
| **Edit/Write가 `permissions.ask` 우회 버그** (GitHub [#22055](https://github.com/anthropics/claude-code/issues/22055)) | [#11226](https://github.com/anthropics/claude-code/issues/11226)의 regression. Edit/Write 툴이 ask 규칙을 건너뛰고 실행 | 사용자 승인이 필수인 경로는 `PreToolUse(Edit|Write)` 훅으로 물리 차단, **ask만 의존 금지** |

**설계 함의**: "실제 차단이 필요한 규칙"과 "유도 수준이면 충분한 규칙"을 명확히 구분하라. 위험 명령·빌드 실패는 전자, 에이전트 호출 추천·리뷰 유도는 후자.

**계층 자동 발견 요약 (2026-04-16 기준)**:

| 요소 | 계층 자동 발견 | 비고 |
|---|---|---|
| `CLAUDE.md` | ✅ 양방향 | 상위/하위 디렉터리 모두. 하위는 파일 접근 시 on-demand |
| `.claude/skills/` | ✅ 양방향 | 2025 후반 추가. `packages/xxx/.claude/skills/` 자동 발견 |
| `.claude/rules/` | ✅ 양방향 | 2025 후반 추가. `paths:` frontmatter로 경로 스코핑 |
| `.claude/agents/` | ⚠️ CWD 기준 walking up | CWD에서 **상위 디렉터리로만** 순회. 루트에서 Claude 시작 시 루트만, `cd modules/xxx/` 후 시작 시 모듈+루트 둘 다 발견 |
| `.claude/settings.json` (hooks/permissions/env) | ❌ | 루트만. Issue #12962 OPEN |
| MCP 서버 | ❌ | 세션 글로벌 |

**Subagent 정규 위치 5곳** (이 외 위치의 `.md`는 subagent로 인식 안 됨):
1. Managed settings (`.claude/agents/`)
2. `--agents` CLI flag
3. `.claude/agents/` (프로젝트, walking up)
4. `~/.claude/agents/` (사용자)
5. Plugin `agents/` 디렉터리

**주의**: `.claude/skills/xxx/agents/*.md`처럼 스킬 폴더 내부에 둔 `.md`는 **subagent가 아니다**. 단순 프롬프트 파일로 취급되며, frontmatter `hooks:`를 정의해도 **Claude Code 런타임이 파싱하지 않고** 그냥 텍스트로 LLM에 전달된다. 이 경우 LLM이 해당 지시를 따를 수도, 따르지 않을 수도 있는 **비결정적 상태**가 된다. Agent Teams 패턴 적용 시 Phase I-10 반드시 참조.

---

## 하네스 5계층 구조

```
정책 (CLAUDE.md)       ← 모든 세션 system prompt에 주입. 짧게. 계층 로드 ✅
규칙 (.claude/rules)   ← paths: frontmatter로 경로 스코프. 2025 후반 신규. 계층 로드 ✅
절차 (.claude/skills)  ← SKILL.md 폴더. frontmatter 자동 로드. 계층 로드 ✅
강제 (.claude/hooks)   ← 결정론적 shell 스크립트. 루트만. 에이전트 자율 준수 X.
격리 (.claude/agents)  ← 컨텍스트 방화벽. 루트만. 복잡한 분석 분리.
정적분석 (Gradle)      ← 컴파일러 레벨 규칙 강제 (Phase F 추가)
자동화 (CI schedule)   ← 주간 GC 등 배경 작업 (Phase H 추가)
```

**v2.1 변경**: `.claude/rules/` 레이어 추가. Skills보다 **가볍고 path-scope 명확**해서 모듈 규칙 배치에 우선 고려. Skills는 여러 파일 구조·스크립트·예시가 필요할 때.

**설계 원칙**:
- CLAUDE.md: **사실/정책만** (200줄 이내). 절차는 Skills로, 경로 스코프 규칙은 `.claude/rules/`로.
- **Rules**: **경로 스코프 규칙 전용**. `paths:` frontmatter + 단일 `.md`. 지식은 Skill로, 경로별 짧은 규칙은 Rule로.
- Skills: **필요할 때만 로드**. `description` + `paths` glob 활용. 절차·템플릿·보조 스크립트 동반 시.
- Hooks: **성공은 침묵, 실패만 표면화**. 매번 실행되므로 가벼워야 함. 루트만 로드되므로 모듈 분기는 스크립트 내부에서.
- Subagents: **반복되는 복잡한 분석**에만. 과하면 오히려 UX 저하. 루트만 로드됨.
- **정적분석**: CODING_RULES.md의 자동화 가능 규칙은 Checkstyle로 물리화.
- **CI 자동화**: 주기적 청소·감사는 CI schedule로 (Claude 세션 밖).

### Rules vs Skills 선택 기준

| 상황 | 선택 | 이유 |
|---|---|---|
| 특정 디렉터리 파일에만 적용되는 짧은 규칙 | **Rule** | `paths:` 스코핑 명확, 단일 .md로 가볍다 |
| 여러 지원 파일(scripts/, examples/, references/) 필요 | **Skill** | Skill만 폴더 구조 지원 |
| `/slash-command`로 수동 호출 | **Skill** | Rule은 슬래시 커맨드 불가 |
| `description` 기반 자동 매칭 로드 | **Skill** | Rule은 `paths:` 또는 전역 로드만 |
| 경로별 자동 로드 + 내용만 필요 | **Rule** | 가장 가벼움 |
| Write 툴로 파일 생성 시에도 규칙 적용 필요 | **Skill + CLAUDE.md 계층** | Rule의 `paths:` 버그 (Issue #23478: Read만 트리거) 회피 |

---

## Phase 도입 순서 (권장)

**Phase 0** · 기본 골격 — `.claude/` 디렉터리와 CLAUDE.md·MCP 등록
**Phase A** · Hooks 5종 — Bash 가드 / 커밋 태그 / 포맷 / 빌드 / 리뷰 유도
**Phase B** · 운영 매크로 Skills — 자주 쓰는 분석을 `/command`로 표준화
**Phase C** · 도메인 Skills — 코드 규칙·플로우·스키마 등을 skill로
**Phase D** · permissions 위생 — allowlist 와일드카드 압축
**Phase E** · CLAUDE.md 슬림화 — 절차 내용을 skill로 이관
**Phase F** · 정적분석 승격 — Spotless + Checkstyle로 선언 규칙 물리화
**Phase G** · 자동 루프 — PostToolUse 포맷 + Stop 체인 + 에이전트 추천
**Phase H** · 주간 GC — GitLab schedule + Discord 알림으로 영구 청소 봇
**Phase I** · 멀티모듈/모노레포 스케일링 (v2.1) — `.claude/rules/` + nested skills + `claudeMdExcludes` + 아키타입 분기

처음부터 전부 만들 필요 없음. 실제 불편을 겪을 때 그 축을 보강.

**Phase I는 언제**:
- 모듈이 10개 이상이고 모듈마다 고유 규칙이 있을 때
- 루트 CLAUDE.md가 계속 비대해지는 게 멈추지 않을 때
- 여러 팀이 한 모노레포를 공유해서 타 팀 CLAUDE.md가 노이즈가 될 때
- 60개+ 모듈 프로젝트는 **Phase 0과 병행**해도 됨 (초기부터 아키타입 설계 필요)

---

## Phase 0: 기본 골격

```bash
cd <project-root>
mkdir -p .claude/{hooks,skills,agents}
```

1. **CLAUDE.md 작성** — [CLAUDE_MD_GUIDE.md](CLAUDE_MD_GUIDE.md). `templates/CLAUDE.md.template` 복사.
2. **MCP 등록** — [../skill-setting.md](../skill-setting.md) 참고 (prod-postgres, playwright, sequential-thinking).
3. **.gitignore에 추가**:
   ```
   .claude/settings.local.json
   .claude/hooks/.state/
   ```

---

## Phase A: Hooks 5종 → **[guide/phase-a-hooks.md](guide/phase-a-hooks.md)**

> **강제 계층**. 편집-빌드-제약을 훅으로 물리화. `pre-bash-guard`(차단) / `verify-commit-msg`(커밋 규약) / `format-changed-java`(포맷) / `post-work-check`(빌드 검증) / `recommend-agent-on-stop`(추천) / `spawn-reviewer-on-stop`(리뷰 유도) **5종** + [A-6 Hook 이벤트 25+종 레퍼런스](guide/phase-a-hooks.md#a-6-hook-이벤트-레퍼런스-2026-04-공식-v22-신규) + [Handler 4종(command/http/prompt/agent)](guide/phase-a-hooks.md#a-6-2-handler-4종-v22-신규) + [async hooks](guide/phase-a-hooks.md#a-6-3-async-hooks-v22-신규) + [defer tool calls](guide/phase-a-hooks.md#a-6-4-defer-tool-calls-v2189-v22-정식화).
>
> **핵심 요점**:
> - `permissions.deny` 버그로 **PreToolUse + exit 2가 유일한 물리 차단**. 이게 없으면 나머지 하네스는 종이 호랑이.
> - `decision:block`은 유도 수준 (LLM이 무시 가능)
> - 2026-04 공식: 이벤트 25+종 중 v2.2에서 추가 고려할 것 → `SubagentStop`·`InstructionsLoaded`·`PreCompact`·`PermissionDenied`·`FileChanged`
>
> **전체 상세**: **[guide/phase-a-hooks.md](guide/phase-a-hooks.md)** (약 326줄 — A-0 차단 레이어부터 A-6-5 PreToolUse 출력 스키마까지)

---

## Phase B: 운영 매크로 Skills

> **v2.2 공식화**: `.claude/commands/`와 `.claude/skills/`는 **동일하게 동작**한다. `.claude/commands/deploy.md`와 `.claude/skills/deploy/SKILL.md` 둘 다 `/deploy`를 만든다. 이름 충돌 시 **skill이 우선**. 신규 기능(supporting files, `paths:`, `disable-model-invocation`, `context: fork` 등)은 skills로만 제공되므로 **신규 작성은 항상 skills**. 기존 `.claude/commands/` 파일은 그대로 동작하므로 마이그레이션은 선택.

자주 쓰는 진단 작업을 `/command` 슬래시 커맨드로 표준화. Subagent를 `context: fork`로 래핑.

### B-1. Subagent 먼저 정의

각 운영 매크로는 `.claude/agents/<name>.md`에 대응 subagent가 있어야 함. 없으면 먼저 [SUBAGENTS_GUIDE.md](SUBAGENTS_GUIDE.md) 참고해 생성.

Subagent 템플릿: `templates/agents/vulnerability-analyzer.md.template`

### B-2. 매크로 Skill 작성

```bash
mkdir -p .claude/skills/<command-name>
cp templates/skills/operational-wrapper/SKILL.md \
   .claude/skills/<command-name>/SKILL.md
```

`SKILL.md` 편집 — `name`, `description`, `agent`(subagent 이름) 수정.

### B-3. 표준 포맷

```yaml
---
name: dlq-check
description: <언제 쓰는지 한 줄. 트리거 문구 예시 포함>
argument-hint: [optional-arg?]
disable-model-invocation: true
context: fork
agent: ops-incident-responder
---

$ARGUMENTS 에 대해 <작업 내용> 해줘.

## 수행 절차
1. ...
2. ...

## 필수 참조
- [../related-skill/SKILL.md](../related-skill/SKILL.md)
```

**핵심 frontmatter**:
- `disable-model-invocation: true` — Claude 자동 호출 차단. `/name`으로만 실행 (사이드 이펙트 방지)
- `context: fork` — 격리된 subagent 컨텍스트에서 실행
- `agent: <subagent-name>` — `.claude/agents/<name>.md` 의 이름과 일치

### B-4. 실제 구현 예시 (도메인별 예시)

실전 적용 시 자주 등장하는 운영 매크로 패턴 (프로젝트마다 이름·대상은 다름):
- `/dlq-check` — 메시지 큐 DLQ 현황·원인 분류
- `/prod-log` — 운영 로그 시간·키워드 필터 조회
- `/pipeline-diagnose` — 파이프라인 장애 진단 (AI·결제·리스팅 등 도메인 맞춤)
- `/stuck-check` — 상태 체류·복구 경로 진단

**원칙**: "자주 치는 3단계 이상의 조사·진단"을 하나의 슬래시 커맨드로 캡슐화. 단일 질문·조회는 매크로 만들 가치 없음.

---

## Phase C: 도메인 Skills → **[guide/phase-c-skills.md](guide/phase-c-skills.md)**

> **절차 계층**. 프로젝트 도메인 지식을 Skill로 격리하고 `description`·`paths` 자동 매칭으로 필요할 때만 로드. CLAUDE.md를 짧게 유지하는 핵심 장치.
>
> **핵심 요점**:
> - **flat `.md` → 폴더 구조**: `CODING_RULES.md` → `coding-rules/SKILL.md` (C-1 일괄 마이그레이션 스크립트)
> - **v2.2 frontmatter 14필드**: `user-invocable`·`effort`·`hooks`·`shell`·`model` 등 신규 5종 (C-4-1)
> - **v2.2 치환자**: `$ARGUMENTS[N]`·`$N`·`${CLAUDE_SESSION_ID}`·`${CLAUDE_SKILL_DIR}` (C-4-2)
> - **v2.2 인라인 쉘**: `` !`command` `` preprocessing (C-4-3)
> - **Live change detection + `SLASH_COMMAND_TOOL_CHAR_BUDGET`**: 세션 중 skill 변경 자동 반영, 디스크립션 예산 확장 (C-4-4)
> - **Nested skills**: 모듈 폴더 내 `.claude/skills/` 자동 발견 (C-5 / Phase I에서 심화)
>
> **전체 상세**: **[guide/phase-c-skills.md](guide/phase-c-skills.md)** (약 234줄 — C-1 마이그레이션부터 C-5 nested까지)
>
> **Skill 작성 규격은 별도 가이드**: [SKILLS_GUIDE.md](SKILLS_GUIDE.md)

---

## Phase D: permissions 위생

### D-1. 증상

`.claude/settings.local.json` 200줄+로 비대. 대부분 사용자가 일일이 승인한 개별 명령이 누적됨.

### D-2. 압축 전략

1. `Bash(git status:*)`, `Bash(git diff:*)` ... → `Bash(git :*)` 1줄
2. `Bash(DOCKER_HOST=tcp://10.0.0.51:2375 docker ps:*)` ... → `Bash(DOCKER_HOST=tcp://10.0.0.* docker:*)` 1줄 (서브넷 와일드카드)
3. `Bash(PGPASSWORD=xxx psql:*)` ... → `Bash(PGPASSWORD=* psql:*)` 1줄
4. 구식 경로(예: 이전 repo 경로) 전수 삭제
5. `__NEW_LINE__*`, `for i in ...`, `do ...`, `done` 같은 파편 엔트리 삭제

### D-3. 기대 효과

실측: 200+줄 → 70줄 내외 (−60~70%). 압축 후에도 실사용 명령은 모두 allow 매칭됨. 템플릿: `templates/settings.local.json.template`

### D-4. 안전장치

```bash
cp .claude/settings.local.json .claude/settings.local.json.bak
# 1~3일 실사용 후 문제없으면 .bak 삭제. 문제 있으면 .bak로 롤백.
```

---

## Phase E: CLAUDE.md 슬림화

### E-1. 기준

- **남길 것**: 역할/소통 규칙, 프로젝트 Overview, 빌드 명령, Module Architecture, Event-Driven 기본 구조
- **분리할 것**: 디버깅 절차, 로그 분석 규칙, 코드 작성 규칙 요약(→ coding-rules skill), 서버 인프라 포인터(→ external-systems skill)

### E-2. 분리 대상 예시

**CLAUDE.md에서 제거**:
```markdown
## 디버깅 & 로그 분석 규칙   ← 절차성
## 코드 작성 규칙 (요약)      ← skill에 이미 있음
## 서버 인프라 및 운영 DB     ← external-systems skill 포인터만
```

**새 skill로 이동**: `.claude/skills/debugging-discipline/SKILL.md`

이 skill은 "근본 원인 분석 절차, 로그 분석 명령어 패턴, 3회 실패 시 중단 원칙" 등 절차성 내용을 묶는다. CLAUDE.md에서는 "디버깅 시 해당 skill 필수 참조"만 명시.

### E-3. "Skills 인덱스" 섹션 추가

CLAUDE.md 하단에:

```markdown
## Skills 인덱스

### 도메인 참조 (자동 로드)
- `coding-rules` — Java 규칙 (`**/*.java`)
- `logging-rules` — 로그 포맷 (`**/*.java`)
- ...

### 운영 매크로 (수동 호출)
- `/dlq-check` — DLQ 진단
- `/pipeline-diagnose [id]` — 파이프라인 진단
- ...
```

**실측 규모**: 절차성 분리 + 자기진화 규약 포함해서도 CLAUDE.md는 **200~240줄 이내**로 유지 가능.

### E-4. BLOCKING 문구 수정

```markdown
## ⛔ 필수 문서 참조 규칙 (BLOCKING)

> 각 Skill은 description/paths 기반으로 자동 로드됩니다.
> 자동 로드 안 되면 Skill 툴로 로드하거나 /<skill-name>으로 수동 호출.
```

(이전: "Read 툴로 먼저 읽어야 합니다.")

### E-5. 하네스 자기진화 규약 섹션 (v2 신규)

Phase F~H 도입 후 CLAUDE.md 상단에 아래 블록 추가:

```markdown
## ⚙️ 하네스 자기진화 규약

**하네스 파일 변경 시 반드시 읽을 파일** (순서대로):
1. `.claude/settings.json` — PreToolUse/PostToolUse/Stop 훅 + 권한 매트릭스
2. `.claude/settings.local.json` — 로컬 allow 패턴
3. `.claude/hooks/` — pre-bash-guard, format-changed-java, post-work-check, recommend-agent-on-stop, spawn-reviewer-on-stop
4. `build.gradle` + `config/checkstyle/checkstyle.xml` — 정적분석 규칙
5. `scripts/weekly-gc.sh` + `.gitlab-ci.yml`의 `weekly-gc` job — 주간 GC
6. 이 섹션 — 변경 절차 자체

**변경 전 체크리스트 (영상 하네스 엔지니어링 3원칙)**:
- [ ] "에러 나면 시스템 고쳐라": 프롬프트 우회 말고 훅/린터 규칙으로 강제
- [ ] "점진 도입": Phase 단위 롤백 가능성 확보
- [ ] "성공 조용, 실패 시끄럽게": 신규 훅 성공 시 무출력, 실패 시만 exit 2(차단) 또는 decision:block(유도)

**알려진 제약** (기대치 관리):
- `permissions.deny`는 v1.0.93+ 버그로 비활성 — PreToolUse `pre-bash-guard.sh`가 유일한 실제 차단
- PostToolUse/Stop `decision:block`은 피드백 수준 — LLM이 무시 가능
- Agent frontmatter `paths` 미지원 — description 매칭 + `recommend-agent-on-stop.sh` 경로 키워드 주입으로만 가능
```

이 블록을 CLAUDE.md 상단(역할·소통 규칙 바로 아래)에 두면, 이후 세션에서 하네스 파일 건드릴 때 Claude가 해당 영역을 먼저 읽도록 유도 가능.

---

## Phase F: 정적분석 승격 (Spotless + Checkstyle) → **[guide/phase-f-static-analysis.md](guide/phase-f-static-analysis.md)**

> **정적분석 계층**. CODING_RULES의 자동화 가능 규칙(wildcard import·네이밍·import 순서)을 Checkstyle로 **컴파일러 레벨 강제**. Phase A 훅의 `decision:block`(유도) 위에 **빌드 실패**라는 진짜 강제 레일.
>
> **핵심 요점**:
> - **버전 확정 (실측)**: palantir-java-format 2.90.0 / Spotless 8.4.0 / Checkstyle 10.21.0
> - **점진 도입**: `ignoreFailures = true` (초기) → 위반 0건 달성 후 `false`로 승격
> - **포맷 베이스라인은 단일 커밋 + `.git-blame-ignore-revs`** (blame 오염 격리, GitHub/GitLab 17.10+ 자동 인식)
> - **실측 성과**: wildcard import 81파일 → 0 / Checkstyle 위반 1,347 → 0 / `ignoreFailures=false` 승격 완료
>
> **전체 상세**: **[guide/phase-f-static-analysis.md](guide/phase-f-static-analysis.md)** (약 186줄 — F-0 전제조건부터 F-7 실측 결과까지)

---

## Phase G: 자동 루프 (PostToolUse + Stop 체인)

> **목표**: 편집 → 포맷 → 빌드 → 리뷰 유도의 자동 파이프라인 구축. **영상 3원칙 "성공 조용, 실패 시끄럽게"** 완전 준수.

### G-1. PostToolUse 포맷 훅

편집 직후 자동 포맷 → AI 인지하지 못하는 사이 정렬 유지.

템플릿: `templates/hooks/format-changed-java.sh`

**로직**:
1. `tool_input.file_path`에서 편집 파일 추출
2. `.java`가 아니거나 `build/generated/` 경로면 exit 0 (무출력)
3. 파일 소속 모듈 탐지 → `./gradlew :module:spotlessApply --quiet --no-daemon`
4. 성공 = exit 0 무출력, 실패 = `{"decision":"block", "reason":"..."}` (피드백 수준)

**비용**: 편집당 1~3초 (ratchetFrom + 모듈 한정 덕)

### G-2. Stop 훅 체인

Stop 이벤트에 3단계 훅 체인:

```
post-work-check.sh          ← 변경 모듈 compile + checkstyle (opt-in test)
  └─ 실패 시 block + 요약 20줄

recommend-agent-on-stop.sh  ← 경로 패턴 매칭으로 관련 에이전트 추천
  └─ 매칭 시 reason에 에이전트 이름 주입

spawn-reviewer-on-stop.sh   ← diff 30줄+ 시 code-reviewer 호출 유도
  └─ 조건 충족 시 reason에 Agent(subagent_type=...) 지시
```

### G-3. 변경 모듈 한정 빌드 (post-work-check 개선)

v1의 `./gradlew build -x test`는 전체 모듈 빌드 — 느리고 과함. v2는 변경 모듈만:

```bash
CHANGED=$(git status --porcelain | awk '{print $2}' | grep '\.java$')
MODULES=$(echo "$CHANGED" | awk -F/ '{print $1}' | sort -u)
TASKS=""
for m in $MODULES; do
  TASKS="$TASKS :${m}:compileJava :${m}:checkstyleMain"
  [ "${CLAUDE_HOOK_TEST:-0}" = "1" ] && TASKS="$TASKS :${m}:test"
done
./gradlew $TASKS --no-daemon
```

**`CLAUDE_HOOK_TEST=1` opt-in 이유**: 테스트 자동 실행은 정확도↑지만 Stop 시간 30~90초 증가. 정확성 원칙과 속도 트레이드오프를 세션 단위로 선택.

실패 시 tail 20줄 요약 (FAILED/Task/stacktrace 라인 추출).

### G-4. 에이전트 자동 추천 (paths 미지원 우회)

Agent frontmatter `paths` 공식 미지원이므로 Stop 훅이 수동으로 경로 매칭:

```bash
CHANGED=$(git diff --name-only HEAD)
echo "$CHANGED" | grep -qE '(StuckRetry|PlatformListingService)' && RECS+="status-transition-auditor "
echo "$CHANGED" | grep -qE 'Outbox' && RECS+="outbox-vulnerability-analyzer "
# ...

[[ -n "$RECS" ]] && jq -n --arg r "$RECS" '{decision:"block", reason:("변경 경로 매칭 에이전트: " + $r)}'
```

**활발한 에이전트만 대상**: agent-memory에 분석 기록이 많은 서브에이전트 3~5개만. 전부 추천하면 신호 대 잡음비 저하.

### G-5. 리뷰 자동 유도

diff 30줄+ 시 `superpowers:code-reviewer` 호출 지시:

```bash
LINES=$(git diff --stat HEAD | tail -1 | grep -oE '[0-9]+ insertion' | grep -oE '[0-9]+')
[[ "$LINES" -ge 30 ]] && jq -n --arg n "$LINES" '{
  decision: "block",
  reason: ("변경 " + $n + "줄. Agent(subagent_type=\"superpowers:code-reviewer\", prompt=\"git diff HEAD 검토\") 호출 권장.")
}'
```

**주의**: `decision:block`은 유도 수준. LLM이 reason을 읽고 판단. 강제 스폰 아님.

### G-6. 모든 Stop 훅 공통 패턴

1. `stop_hook_active=true`면 즉시 exit 0 (재귀 방지)
2. `CLAUDE_HOOKS_SKIP=1`이면 즉시 exit 0
3. 관련 없는 상황은 무출력 exit 0
4. 실패/유도 시만 `exit 0 + JSON block` 또는 `exit 2`

---

## Phase H: 주간 GC (GitLab Schedule + Discord) → **[guide/phase-h-weekly-gc.md](guide/phase-h-weekly-gc.md)**

> **주기 청소 계층**. 영상 하네스 엔지니어링 ④ "가비지 컬렉션" 구현. Claude 세션 외부(CI schedule)에서 매주 Checkstyle 위반 집계·TODO 스캔·agent-memory archive·레거시 참조 수집.
>
> **핵심 요점**:
> - **GitLab schedule + Discord webhook** (cron `0 3 * * 1`, Asia/Seoul)
> - **기존 job에 schedule 차단 규칙 필수**: `$CI_PIPELINE_SOURCE == "schedule"` 매칭 `when: never` prepend 안 하면 매주 배포 자동 실행됨
> - **Discord webhook 실패는 `|| true`**: 알림 실패가 GC 본체 실패로 전파되지 않게
> - **artifacts 30일 보관**: `docs/maintenance/YYYY-MM-DD/`에 요약·로그 축적
>
> **전체 상세**: **[guide/phase-h-weekly-gc.md](guide/phase-h-weekly-gc.md)** (약 151줄 — H-1 스크립트부터 H-7 실측 시간까지)

---

## Phase I: 멀티모듈/모노레포 스케일링 (v2.1) → **[guide/phase-i-monorepo.md](guide/phase-i-monorepo.md)**

> **모노레포 스케일링 계층**. 10~100+ 모듈 대형 모노레포에서 하네스가 스케일하도록 설계. 아키타입 분류 / `.claude/rules/` / nested skills / `claudeMdExcludes` / 아키타입 기반 훅 분기 / `InstructionsLoaded` / Agent Teams 2변형.
>
> **전제**: Phase 0~E 기본 세팅 완료. 60+ 모듈 프로젝트는 Phase 0과 **동시 설계** 권장.
>
> **핵심 요점**:
> - **아키타입 5~8개**로 60+ 모듈 귀결 — 개별 모듈 규약 복제 금지 (I-0 안티패턴, I-1 분류)
> - **Rules vs Skills 배치 전략**: 짧은 경로 스코프 규칙은 `.claude/rules/` + `paths:`, 지원 파일 필요하면 Skill (I-2)
> - **루트 CLAUDE.md 150줄 유지** + 개별 모듈 CLAUDE.md는 예외 규칙만 (I-3)
> - **`claudeMdExcludes`**로 타 팀 CLAUDE.md 노이즈 제거 (I-4)
> - **`detect-archetype.sh`** 공용 함수로 60-way case 회피 (I-5)
> - **`InstructionsLoaded` 훅**으로 로드 디버깅 (I-6)
> - **`CLAUDE_CODE_NEW_INIT=1`** 서브에이전트 기반 초기 탐색 (I-7)
> - **Virtual Monorepo**: `--add-dir` + `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` (I-8)
> - **Agent Teams 패턴 A/B** — 훅 추적 필요하면 반드시 패턴 B (루트 subagent 등록) (I-10)
>
> **전체 상세**: **[guide/phase-i-monorepo.md](guide/phase-i-monorepo.md)** (약 569줄 — I-0~I-11 + Agent Teams 심화)


---

## 적용 후 검증

```bash
# 1. Hook 등록 확인
jq '.hooks | keys' .claude/settings.json
# → ["PostToolUse","PreToolUse","Stop"]
#
# 또는 Claude Code 세션 내에서 `/hooks` 입력 (v2.2 공식):
#   - 이벤트별 등록 훅 수 + 핸들러 타입([command]/[prompt]/[agent]/[http]) + 출처 표시
#   - 출처 라벨: User / Project / Local / Plugin / Session / Built-in
#   - jq 방식보다 계층 디버깅(어느 settings 파일에서 로드됐는지)에 유리

# 2. Hook 실행권한
ls -l .claude/hooks/*.sh

# 3. Skill 폴더·frontmatter
echo "folders: $(ls -d .claude/skills/*/ | wc -l)"
echo "skills: $(ls .claude/skills/*/SKILL.md | wc -l)"
echo "frontmatter: $(grep -l '^name: ' .claude/skills/*/SKILL.md | wc -l)"

# 4. Hook 실제 deny 동작
echo '{"tool_input":{"command":"git commit -m \"bad msg\""}}' \
  | .claude/hooks/verify-commit-msg.sh \
  | jq -r '.hookSpecificOutput.permissionDecision'
# → deny

# 5. 새 세션 시작 후 `What skills are available?` 입력 시
#    project skill 목록이 전부 나오면 OK
```

---

## 참고: 실측 지표 (7개 모듈 Spring Boot 멀티모듈 적용 결과)

### v1 도입 (Phase 0~E) 지표
| 지표 | Before | After v1 | 변화 |
|------|--------|-------|------|
| `.claude/settings.local.json` | 219줄 | 73줄 | -66.7% |
| CLAUDE.md | 298줄 | 221줄 | -25.8% |
| Skill 자동 로드 가능 | 0개 (flat MD) | 24개 (SKILL.md) | +24 |
| Hooks | 0개 | 3개 | +3 |
| 운영 슬래시 커맨드 | 0개 | 5개 | +5 |
| 커밋 태그 강제 | 에이전트 자율 | 결정론적 | ✓ |
| 빌드 검증 | 에이전트 자율 | Stop hook | ✓ |

### v2 확장 (Phase F~H, 2026-04-15) 지표
| 지표 | v1 | v2 | 변화 |
|------|----|----|------|
| Hooks | 3개 | **5개** | +2 (pre-bash-guard, format-changed-java, recommend-agent-on-stop, spawn-reviewer-on-stop; check-java-imports 삭제) |
| 정적분석 플러그인 | 0개 | **Spotless + Checkstyle** | palantir 2.90.0 + Checkstyle 10.21.0 |
| wildcard import | 173 파일 | **0** | 81 자동 + 2 수동 (Gradle 소스세트 내) |
| Checkstyle 위반 | 측정 불가 (규칙 없음) | **0건** | ignoreFailures=false 승격 |
| 포맷 일관성 | 혼재 | **palantir 2.90.0 통일** | 507 파일 베이스라인 |
| `.git-blame-ignore-revs` | 없음 | **등록** | blame 오염 격리 |
| 위험 Bash 차단 | 없음 | **PreToolUse exit 2** | rm -rf 시스템 + force push + DROP + 등 11종 |
| 위험 차단 레이어 | permissions.allow만 | **실제 exit 2** | deny 버그 우회 |
| 에이전트 자동 추천 | 없음 | **Stop 훅 경로 매칭** | paths 미지원 우회 |
| 리뷰 자동 유도 | 없음 | **diff 30줄+ 시 유도** | superpowers:code-reviewer |
| 주간 GC | 없음 | **GitLab schedule + Discord** | 매주 월 03:00 KST |
| GC 리포트 보관 | n/a | **30일** | GitLab artifacts |

### 영상 하네스 5대 요소 달성도 변화
| 요소 | Before | v1 | v2 |
|---|---|---|---|
| ① 컨텍스트 파일 | F | A | A+ |
| ② 자동 강제 | F | C+ | **A** |
| ③ 권한 경계 | F | C | **A-** (deny 버그 우회) |
| ④ 가비지 컬렉션 | F | F | **A** |
| ⑤ 역할 분리 | F | B- | **B+** |

---

## 관련 문서

- [CLAUDE_MD_GUIDE.md](CLAUDE_MD_GUIDE.md) — CLAUDE.md 작성 가이드
- [SKILLS_GUIDE.md](SKILLS_GUIDE.md) — Skill 작성 가이드
- [SUBAGENTS_GUIDE.md](SUBAGENTS_GUIDE.md) — Subagent 작성 가이드
- [../skill-setting.md](../skill-setting.md) — MCP 서버 설정
- `templates/` — 스크립트·설정·Skill·Agent 템플릿

## 공식 docs

- [Hooks — code.claude.com](https://code.claude.com/docs/en/hooks)
- [Skills — code.claude.com](https://code.claude.com/docs/en/skills)
- [Subagents — code.claude.com](https://code.claude.com/docs/en/sub-agents)
