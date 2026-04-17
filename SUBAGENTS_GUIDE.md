# 서브에이전트(Sub-agents) 작성 규격

## 서브에이전트란?

`.claude/agents/*.md`. 메인 Claude 대화에서 `Agent` 툴로 호출되는 **별 컨텍스트의 작업자**. 메인 컨텍스트를 더럽히지 않고 복잡한 분석을 위임할 때 쓴다.

## 공식 발견 위치 (5곳)

Subagent로 인식되는 `.md`는 다음 5곳에만 위치할 수 있다. 이 외의 위치에 둔 `.md`는 subagent가 아니며 frontmatter가 파싱되지 않는다.

| 우선순위 | 위치 |
|---|---|
| 1 (highest) | Managed settings 의 `.claude/agents/` |
| 2 | `--agents` CLI flag (JSON, 세션 한정) |
| 3 | 프로젝트 `.claude/agents/` |
| 4 | 사용자 `~/.claude/agents/` |
| 5 (lowest) | Plugin의 `agents/` 디렉터리 |

**발견 규칙**: 프로젝트 subagent는 **CWD 기준 walking up(상위 디렉터리 순회)**으로 발견된다. 즉 CWD가 `<repo>/modules/payment/`일 때 `modules/payment/.claude/agents/` + `<repo>/.claude/agents/` 둘 다 발견되지만, CWD가 `<repo>/`이면 루트만 발견된다.

**스킬 폴더 내부 `.md`는 subagent가 아니다**: `.claude/skills/xxx/agents/*.md` 같은 위치의 파일은 Claude Code가 subagent로 등록하지 않는다. 이 파일의 frontmatter `hooks:`는 런타임 훅으로 작동하지 않으며, LLM이 텍스트로 읽고 자율 판단하는 비결정 영역이 된다. 훅 기반 추적이 필요하면 반드시 위 5곳 중 하나에 등록해야 한다.

## 언제 만드는가

다음을 모두 만족할 때:

1. **반복성**: 같은 종류의 분석을 2회 이상 한 적 있다
2. **복잡성**: 한 번 분석에 5개 이상 파일을 읽거나, 단계가 많다
3. **컨텍스트 격리 필요**: 결과만 메인에 돌려받고 중간 탐색은 격리하고 싶다
4. **재사용 가능한 절차**: 매번 같은 순서/관점으로 본다

처음부터 만들지 마라. 같은 작업을 두 번 반복하게 되면 그때 만들어라.

## 검증된 카테고리

실전 적용에서 실효성 확인된 "분석 계열" 에이전트 예시 (이벤트 드리븐 Spring Boot 백엔드 기준):

| 에이전트 예시 | 분석 대상 |
|---|---|
| `outbox-vulnerability-analyzer` | Outbox 패턴 취약점 (중복 발행, 적체) |
| `rabbitmq-consumer-analyzer` | 메시지 큐 Consumer 멱등성/장애 |
| `scheduler-vulnerability-analyzer` | 스케줄러 race condition |
| `tx-integrity-analyzer` | 트랜잭션 경계, propagation 문제 |
| `status-transition-auditor` | 상태 전이 누락/race |
| `external-api-resilience-auditor` | 외부 API 호출 회복력 |
| `infra-vulnerability-analyzer` | 인프라 설정/운영 취약점 |

→ 모두 **"X 측면에서 코드 전체를 훑고 위험 항목을 리포트"** 형태. 코드 작성 에이전트는 없음. (메인 Claude가 직접 작성하는 게 더 효율적)

프로젝트 도메인에 맞춰 조정: 결제 시스템이면 `pricing-consistency-analyzer`, 이커머스라면 `inventory-integrity-analyzer` 같은 식.

## 파일 포맷

```markdown
---
name: "agent-name-kebab-case"
description: "이 에이전트가 언제 호출되어야 하는지를 매우 상세히. 사용자의 어떤 발화/요청에 매칭되어야 하는지 예시 포함. \n\nExamples:\n- user: \"...\"\n  assistant: \"...에이전트를 실행하겠습니다.\"\n  <Agent tool call: agent-name>\n\n- user: \"...\"\n  ..."
---

# 에이전트 본문 (시스템 프롬프트)

당신의 역할 / 분석 절차 / 출력 포맷 등을 상세히 기술.
```

## 전체 frontmatter 필드 (2026-04 공식, v2.2 갱신)

공식 문서(`code.claude.com/docs/en/sub-agents`)의 frontmatter 스키마. **필수는 `name`·`description` 2개**. 나머지는 선택.

| 필드 | 용도 | 기본값 / 제약 |
|---|---|---|
| `name` | 고유 식별자 | lowercase + hyphen |
| `description` | Claude가 위임 판단에 사용 | 트리거 예시 포함 권장 |
| `tools` | allowlist. 미지정 시 **전체 툴 상속** | `Read, Grep, Glob, Bash` 등 |
| `disallowedTools` | denylist. 상속된 툴에서 제거 | `tools`와 동시 사용 가능 (deny 먼저 적용) |
| `model` | `sonnet`·`opus`·`haiku` 또는 full ID(`claude-opus-4-7`) 또는 `inherit` | 기본 `inherit` |
| `permissionMode` | `default`·`acceptEdits`·`auto`·`dontAsk`·`bypassPermissions`·`plan` | 부모가 auto/bypass/acceptEdits면 **override 불가** |
| `maxTurns` | 최대 agentic 턴 | |
| `skills` | **시작 시 skill 본문을 시스템 프롬프트에 주입** (v2.2 공식화) | subagents는 parent skills 상속 안 함. 명시 필수 |
| `mcpServers` | MCP 서버 목록 (인라인 또는 참조) | 이 subagent 한정 연결 |
| `hooks` | subagent 라이프사이클 전용 훅 | `PreToolUse`/`PostToolUse`/`Stop`(→ `SubagentStop`으로 변환) |
| `memory` | `user` / `project` / `local` 영속 디렉터리 | `~/.claude/agent-memory/<name>/` 또는 `.claude/agent-memory[-local]/<name>/` (v2.2 공식화) |
| `background` | `true` 시 항상 백그라운드 실행 | default `false` |
| `effort` | `low`·`medium`·`high`·`xhigh`·`max` 추론 레벨 (v2.2 신규) | 세션 effort override |
| `isolation` | `worktree` 지정 시 임시 git worktree에 격리 실행 (v2.2 공식화) | 변경 없으면 자동 정리 |
| `color` | UI 표시 색상 | `red`·`blue`·`green`·`yellow`·`purple`·`orange`·`pink`·`cyan` |
| `initialPrompt` | `--agent`·`agent` 설정으로 **메인 세션 에이전트**로 띄울 때 자동 제출되는 첫 유저 턴 | commands·skills 처리됨 |

### 공식 주의사항

- **Plugin subagents**: `hooks`·`mcpServers`·`permissionMode` 필드 **무시** (보안). 사용 필요 시 `.claude/agents/` 또는 `~/.claude/agents/`로 복사.
- **v2.1.63 rename**: 기존 `Task` 툴이 **`Agent`로 개명**. 기존 `Task(...)` 참조는 alias로 유지.
- **Subagent는 subagent를 스폰할 수 없음**. Agent Teams가 필요하면 `/en/agent-teams` 참고.

## Skills Preloading (v2.2 공식화)

서브에이전트는 **parent 세션의 skill을 상속하지 않는다**. 필요한 skill은 `skills:` 필드로 명시 → 시작 시 **본문 전체가 시스템 프롬프트에 주입**됨 (invocation이 아닌 inject).

```yaml
---
name: api-developer
description: API 엔드포인트 구현 전담
skills:
  - api-conventions
  - error-handling-patterns
---

팀 컨벤션·에러 핸들링 패턴을 따르는 API 엔드포인트를 구현.
```

**skill `context: fork`와의 차이**:

| 방향 | 시스템 프롬프트 | Task | 추가 주입 |
|---|---|---|---|
| **Skill (`context: fork`)** | agent 타입의 기본 프롬프트 (`Explore` 등) | SKILL.md 본문 | CLAUDE.md |
| **Subagent (`skills:`)** | subagent markdown 본문 | Claude 위임 메시지 | preloaded skills 본문 + CLAUDE.md |

**활용 예**:
- 결제 모듈 전담 서브에이전트 → `skills: [payment-conventions, api-error-handling]` 로 필수 지식 상시 주입
- 초기 탐색 전담 → skill 없이 `description`만으로도 충분 (light)

## Permission Modes (v2.2 정리)

`permissionMode` 필드의 전체 옵션:

| 모드 | 동작 |
|---|---|
| `default` | 표준 권한 체크 + 프롬프트 |
| `acceptEdits` | 파일 편집·일반 FS 명령 자동 승인 (`additionalDirectories` 포함) |
| `auto` | 백그라운드 분류기가 명령 검토 (auto mode) |
| `dontAsk` | 프롬프트 자동 거절 (명시 allow된 툴만 통과) |
| `bypassPermissions` | ⚠️ 프롬프트 전체 스킵. `.git`/`.claude`/`.vscode`/`.idea`/`.husky`는 여전히 확인 (단, `.claude/commands`·`.claude/agents`·`.claude/skills`는 예외) |
| `plan` | 읽기 전용 탐색 (plan mode) |

**상속 우선순위**: 부모가 `bypassPermissions`·`acceptEdits`면 **subagent의 `permissionMode` 무시**. `auto` 부모도 subagent는 auto 강제 상속.

## Memory Scope 선택 가이드 (v2.2 공식화)

| 스코프 | 위치 | 언제 |
|---|---|---|
| `project` (권장 기본) | `.claude/agent-memory/<name>/` | 프로젝트 특화 지식. VCS 공유 가능 |
| `user` | `~/.claude/agent-memory/<name>/` | 모든 프로젝트 공통 |
| `local` | `.claude/agent-memory-local/<name>/` | 프로젝트 특화지만 VCS 제외 (개인 노트) |

**동작**:
- subagent 프롬프트에 `MEMORY.md`의 처음 200줄 또는 25KB (작은 쪽) 자동 주입
- Read·Write·Edit 툴 자동 활성화 (memory 관리용)
- 초과분 정리는 subagent 자신이 수행

## `initialPrompt` — Main Session Agent 패턴 (v2.2 신규)

`claude --agent <name>` 또는 `settings.json`의 `"agent": "<name>"` 로 **메인 세션 전체**를 subagent 프롬프트로 띄울 때만 동작. 첫 유저 턴으로 자동 제출됨. commands·skills도 해석됨.

```yaml
---
name: daily-standup
description: 어제 커밋 요약 + 오늘 할 일 제안
initialPrompt: "/git-log-yesterday 후 오늘 작업 계획 3가지를 추천"
---
```

```bash
claude --agent daily-standup
# → 세션 시작 즉시 initialPrompt가 첫 유저 메시지로 자동 제출
```

**⚠️ frontmatter `hooks:`는 이 모드에서 fire 안 됨** (공식 주의). Agent 툴이나 @-mention으로 spawn 될 때만 fire.

### description 필드가 가장 중요

Claude는 description만 보고 자동 호출 여부를 판단한다. 모호하면 호출 안 됨.
- ❌ "Outbox를 분석합니다"
- ✅ "Outbox 패턴의 취약점, 장애 시나리오, 운영 리스크를 분석합니다. 다음 경우에 사용: relay scheduler 동작, 트랜잭션 실패, 메시지 중복, ..."

Examples 섹션은 거의 필수. user의 한국어 발화 → assistant 응답 → 호출 형태로 3개 정도.

### name은 kebab-case
파일명과 일치시킨다. `outbox-vulnerability-analyzer.md` ↔ `name: outbox-vulnerability-analyzer`

## 본문(시스템 프롬프트) 작성

```markdown
# Outbox 패턴 취약점 분석가

## 역할
당신은 이 프로젝트의 Outbox 패턴을 분석하는 시니어 백엔드 엔지니어입니다.

## 분석 절차
1. `<module>/.../OutboxEvent*.java` 파일 모두 읽기
2. OutboxRelayScheduler의 동작 + race condition 검토
3. 트랜잭션 경계 확인
4. ...

## 분석 관점 (필수 체크리스트)
- [ ] 중복 발행 가능성
- [ ] 메시지 손실 가능성
- [ ] DEAD_LETTER 처리
- [ ] ...

## 출력 포맷
| Severity | 위치 | 문제 | 영향 | 권장 조치 |
|---|---|---|---|---|
...

## 금지사항
- 코드를 직접 수정하지 말 것 (분석만)
- 추측 금지, 코드/DB 실측 기반
```

## Frontmatter `hooks:` 필드 (공식 지원)

Subagent frontmatter에 `hooks:` 필드를 정의하면 **해당 subagent 라이프사이클에만** 훅이 fire된다. 전역 `settings.json` 훅과 달리 다른 subagent·메인 세션에는 영향이 없다.

```yaml
---
name: payment-analyzer
description: 결제 모듈 분석 전담
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/hooks/payment-bash-guard.sh"
  PostToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/hooks/verify-payment-output.sh"
  Stop:
    - hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-analysis-result.sh"
  # 런타임에 SubagentStop으로 변환됨
---
```

### 이벤트 종류

| 이벤트 | Matcher | fire 시점 |
|---|---|---|
| `PreToolUse` | 도구 이름 | Subagent가 도구를 사용하기 전 |
| `PostToolUse` | 도구 이름 | Subagent가 도구를 사용한 후 |
| `Stop` | (없음) | Subagent가 완료될 때 (런타임에 `SubagentStop`으로 변환) |

### 활용 패턴

- **결정론적 검증**: agent가 작성한 산출물의 경로·포맷을 셸 스크립트로 물리 검증 후 실패 시 재시도 유도
- **목적별 훅 분리**: 전역 `settings.json` 훅이 "모든 툴 호출에 fire"되는 문제를 회피. 예를 들어 `search-analyzer`·`feature-implementor`·`migration-runner`가 각자 다른 훅을 가지도록 설계.
- **Agent Teams 파이프라인 추적**: 스킬이 여러 subagent를 순차 호출할 때 각 subagent의 PostToolUse·SubagentStop으로 단계별 로그·검증 자동화.

### 위치 제약

**본 파일이 공식 5곳 중 하나에 있어야만 훅이 fire된다**. 스킬 폴더 내부(`.claude/skills/xxx/agents/*.md`)에 둔 `.md`는 Claude Code가 subagent로 인식하지 않으므로 `hooks:` 정의가 무시되고 LLM이 텍스트로 읽는 비결정 영역이 된다.

## 안티패턴

- ❌ **너무 일반적인 에이전트**: "코드 리뷰 에이전트" → 매칭 모호. `tx-integrity-analyzer`처럼 특화한다.
- ❌ **에이전트 안에서 또 에이전트 호출**: 기술적으로 가능하나 컨텍스트 폭발을 유발한다. 한 단계만 허용한다.
- ❌ **메인이 할 일을 굳이 에이전트로**: 단순 grep·단일 파일 수정은 메인에서 처리한다.
- ❌ **출력 포맷 미정의**: 결과가 자유서술이면 메인이 다시 정리해야 한다. 표·체크리스트 형식을 강제한다.
- ❌ **스킬 폴더 내부에 agent `.md` 배치**: subagent로 인식되지 않아 frontmatter `hooks:`·`allowed-tools`·`model` 등이 작동하지 않는다. 훅 추적이 필요하면 반드시 `.claude/agents/`에 등록한다.
- ❌ **훅으로 처리할 규칙을 프롬프트 텍스트에만 의존**: LLM이 지시를 매번 다르게 해석하여 비결정 동작을 유발한다. 결정론이 필요하면 frontmatter `hooks:` + 셸 스크립트로 강제한다.

## 실행 예시

```
Agent(
  description="Outbox 취약점 분석",
  subagent_type="outbox-vulnerability-analyzer",  # .claude/agents/에 등록된 이름
  prompt="..."
)
```
서브에이전트가 자기 컨텍스트에서 작업하고 결과 텍스트만 메인에 반환한다. 등록된 subagent의 frontmatter 훅은 해당 실행 중 fire된다.

## 병렬 실행

독립적인 분석은 한 메시지에 여러 Agent 툴을 호출하여 병렬화할 수 있다.

## 템플릿

[templates/agents/vulnerability-analyzer.md.template](templates/agents/vulnerability-analyzer.md.template)
