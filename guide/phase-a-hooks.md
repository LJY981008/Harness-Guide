# Phase A — Hooks 5종 + Hook 이벤트 레퍼런스

> ← **[HARNESS_SETUP_GUIDE.md](../HARNESS_SETUP_GUIDE.md)** (상위 가이드로 돌아가기)
>
> **관련 Phase**: [B (운영 매크로)](../HARNESS_SETUP_GUIDE.md#phase-b-운영-매크로-skills) · [C (도메인 Skills)](phase-c-skills.md) · [F (정적분석)](phase-f-static-analysis.md) · [G (자동 루프)](../HARNESS_SETUP_GUIDE.md#phase-g-자동-루프-posttooluse--stop-체인) · [I (모노레포)](phase-i-monorepo.md)
>
> **역할**: Claude Code 하네스의 **강제 계층**. 에이전트 자율 준수에 의존하지 않는 물리적 규칙을 훅으로 구현한다. **성공 침묵·실패 시끄럽게** 원칙.

## 목차
- [A-0. 실제 차단 레이어 (PreToolUse Bash 가드)](#a-0-실제-차단-레이어-pretooluse-bash-가드--신규)
- [A-1. 스크립트 복사](#a-1-스크립트-복사)
- [A-2. settings.json 등록](#a-2-settingsjson-등록)
- [A-3. 프로젝트별 조정](#a-3-프로젝트별-조정)
- [A-4. 우회 방법](#a-4-우회-방법)
- [A-5. 스모크 테스트](#a-5-스모크-테스트)
- [A-6. Hook 이벤트 레퍼런스 (2026-04 공식, v2.2 신규)](#a-6-hook-이벤트-레퍼런스-2026-04-공식-v22-신규)
  - [A-6-1. Hook 이벤트 전체 표](#a-6-1-hook-이벤트-전체-표)
  - [A-6-2. Handler 4종 (v2.2 신규)](#a-6-2-handler-4종-v22-신규)
  - [A-6-3. Async hooks (v2.2 신규)](#a-6-3-async-hooks-v22-신규)
  - [A-6-4. Defer tool calls (v2.1.89+, v2.2 정식화)](#a-6-4-defer-tool-calls-v2189-v22-정식화)
  - [A-6-5. 업그레이드된 PreToolUse 출력 스키마 (v2.2)](#a-6-5-업그레이드된-pretooluse-출력-스키마-v22)

---

## Phase A: Hooks 5종

> **v2 변경점**: 이전 3종(verify-commit-msg, check-java-imports, post-work-check) → 5종. `check-java-imports.sh`는 Checkstyle(Phase F)로 흡수되며 삭제. 신규 4종 추가: `pre-bash-guard`(실제 차단), `format-changed-java`(PostToolUse), `recommend-agent-on-stop`(Stop 추천), `spawn-reviewer-on-stop`(Stop 리뷰 유도).

### A-0. 실제 차단 레이어 (PreToolUse Bash 가드) — 신규

`permissions.deny`가 버그로 비활성이므로 PreToolUse hook + `exit 2`가 **유일한 물리적 차단 수단**. 이게 없으면 나머지 하네스는 종이 호랑이.

템플릿: `../templates/hooks/pre-bash-guard.sh` (프로젝트에 맞게 조정)

차단 대상 (11종 위험 패턴, bash regex):
- `rm -rf /`, `rm -rf /etc`·`/home`·`/usr` 등 시스템 디렉토리
- `rm -rf ~`, `rm -rf *` (wildcard)
- `git push --force` / `-f` / `--force-with-lease`
- `git push origin master` / `main` (`master:dev` 같은 refspec은 통과)
- `git reset --hard origin/*`
- `git branch -D`
- `docker system prune`, `docker volume prune`
- `DROP DATABASE`, `DROP SCHEMA`, `TRUNCATE` (psql 경유 포함)
- `git commit/push --no-verify` (훅 우회)

**핵심 문법**: `exit 2` + stderr 메시지. `exit 0 + JSON`은 피드백 수준이라 차단 안 됨.

### A-1. 스크립트 복사

```bash
cp templates/hooks/*.sh .claude/hooks/
chmod +x .claude/hooks/*.sh
```

| 스크립트 | 이벤트 | 동작 | 차단 레벨 |
|---------|--------|------|----------|
| `pre-bash-guard.sh` | `PreToolUse(Bash)` (매 실행) | 위험 명령 11종 정규식 매칭 시 `exit 2` | **진짜 차단** |
| `verify-commit-msg.sh` | `PreToolUse(Bash)` if=`Bash(git commit *)` | 커밋 태그(feat/fix/refactor/docs/test/chore/perf) 강제 | deny (JSON) |
| `format-changed-java.sh` | `PostToolUse(Edit\|Write\|MultiEdit)` | 편집된 .java의 소속 모듈에 `:module:spotlessApply` | block on fail (피드백) |
| `post-work-check.sh` | `Stop` | 변경 모듈 한정 `:module:compileJava :module:checkstyleMain` (CLAUDE_HOOK_TEST=1이면 :test 추가) | block on fail (피드백) |
| `recommend-agent-on-stop.sh` | `Stop` | git diff 경로 패턴으로 활발한 서브에이전트 추천 | 유도만 |
| `spawn-reviewer-on-stop.sh` | `Stop` | diff 30줄+ 시 `superpowers:code-reviewer` 호출 유도 | 유도만 |

### A-2. settings.json 등록

템플릿: `../templates/settings.json.template`

```json
{
  "permissions": { ... },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/pre-bash-guard.sh",
            "timeout": 5
          },
          {
            "type": "command",
            "if": "Bash(git commit *)",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/verify-commit-msg.sh",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [{
          "type": "command",
          "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/format-changed-java.sh",
          "timeout": 30
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-work-check.sh",
            "timeout": 240
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/recommend-agent-on-stop.sh",
            "timeout": 5
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/spawn-reviewer-on-stop.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

### A-3. 프로젝트별 조정

- **Gradle 대신 Maven**: `post-work-check.sh`의 `./gradlew :module:compileJava :module:checkstyleMain` 을 `mvn -pl <module> -am compile` 로 교체.
- **비-Java 프로젝트**: `format-changed-java.sh` 삭제 + 그 PostToolUse 블록 제거, 또는 Prettier/ruff 등 해당 언어 포매터로 교체.
- **커밋 태그 규약 다름**: `verify-commit-msg.sh`의 정규식 `^(feat|fix|refactor|docs|test|chore|perf)` 수정.
- **pre-bash-guard 커스터마이징**: 프로젝트 기본 브랜치가 main이면 패턴 확인. 허용 refspec 추가 필요 시 regex 조정.
- **recommend-agent-on-stop 키워드**: 프로젝트 도메인 클래스명에 맞게 `(StuckRetry|Outbox|...)` 패턴 교체.

### A-4. 우회 방법

긴급 상황:
```bash
CLAUDE_HOOKS_SKIP=1 claude ...
```
(pre-bash-guard 포함 모든 훅 스킵. 단 `--no-verify` 자체는 여전히 PreToolUse에서 차단됨 — guard가 먼저 실행되어 `CLAUDE_HOOKS_SKIP` 확인 전에 종료)

### A-5. 스모크 테스트

```bash
# 위험 명령 차단
echo '{"tool_input":{"command":"rm -rf /etc"}}' | .claude/hooks/pre-bash-guard.sh
# → stderr에 차단 메시지 + exit 2

echo '{"tool_input":{"command":"git push origin master --force"}}' | .claude/hooks/pre-bash-guard.sh
# → exit 2

# 정상 명령 통과
echo '{"tool_input":{"command":"rm -rf /tmp/test-123"}}' | .claude/hooks/pre-bash-guard.sh
# → exit 0 (무출력)

echo '{"tool_input":{"command":"git push origin master:dev"}}' | .claude/hooks/pre-bash-guard.sh
# → exit 0

# 커밋 메시지 검증
echo '{"tool_input":{"command":"git commit -m \"bad\""}}' | .claude/hooks/verify-commit-msg.sh
# → deny JSON

echo '{"tool_input":{"command":"git commit -m \"feat: x\""}}' | .claude/hooks/verify-commit-msg.sh
# → exit 0

# Stop 훅 (Java 변경 없을 때)
echo '{"session_id":"s1","cwd":"'$(pwd)'","stop_hook_active":false}' | .claude/hooks/post-work-check.sh
# → exit 0
```

---

### A-6. Hook 이벤트 레퍼런스 (2026-04 공식, v2.2 신규)

> **변경 이유**: v2.1까지는 PreToolUse/PostToolUse/Stop 3종만 활용. 2026 공식 문서는 25+종 이벤트 + 4종 handler + async + defer를 표준화. 하네스 설계 시 어떤 이벤트를 선택할지 **정확한 카탈로그**가 필요하다.

#### A-6-1. Hook 이벤트 전체 표

**Cadence별 분류**:

| Cadence | 이벤트 | fire 시점 | Matcher |
|---|---|---|---|
| **Session (1회/세션)** | `SessionStart` | 세션 시작/재개 | `startup`, `resume`, `clear`, `compact` |
| | `SessionEnd` | 세션 종료 | `clear`, `resume`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, `other` |
| | `InstructionsLoaded` | CLAUDE.md·rules 로드 시 | `session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact` |
| **Turn (1회/턴)** | `UserPromptSubmit` | 유저 프롬프트 제출 직후 | (없음) |
| | `Stop` | Claude 응답 완료 | (없음) |
| | `StopFailure` | API 에러로 턴 종료 | `rate_limit`, `authentication_failed`, `billing_error`, `invalid_request`, `server_error`, `max_output_tokens`, `unknown` |
| **Tool (매 툴 호출)** | `PreToolUse` | 툴 실행 전 | 툴명 (`Bash`, `Edit`, `mcp__.*`, etc.) |
| | `PostToolUse` | 툴 성공 후 | 툴명 |
| | `PostToolUseFailure` | 툴 실패 후 (v2.2 인식) | 툴명 |
| | `PermissionRequest` | 권한 다이얼로그 표시 | 툴명 |
| | `PermissionDenied` | auto 모드가 거절 | 툴명 |
| **Subagent** | `SubagentStart` | 서브에이전트 스폰 | agent 타입 (`Explore`, `Plan`, 커스텀명) |
| | `SubagentStop` | 서브에이전트 종료 | agent 타입 |
| **Task** | `TaskCreated` | TaskCreate 호출 시 | (없음) |
| | `TaskCompleted` | 태스크 완료 표시 시 | (없음) |
| **Agent Teams** | `TeammateIdle` | 팀메이트 에이전트가 idle 직전 | (없음) |
| **Compaction** | `PreCompact` | 컨텍스트 압축 직전 | `manual`, `auto` |
| | `PostCompact` | 압축 완료 후 | `manual`, `auto` |
| **Config/Filesystem** | `ConfigChange` | 설정 파일 변경 | `user_settings`, `project_settings`, `local_settings`, `policy_settings`, `skills` |
| | `CwdChanged` | CWD 변경 | (없음) |
| | `FileChanged` | 감시 파일 디스크 변경 | 파일명 glob (`.env\|.envrc`) |
| | `WorktreeCreate` | `--worktree`로 워크트리 생성 | (없음) |
| | `WorktreeRemove` | 워크트리 제거 | (없음) |
| **MCP Elicitation** | `Elicitation` | MCP 서버가 사용자 입력 요청 | MCP 서버명 |
| | `ElicitationResult` | 유저가 MCP elicitation 응답 | MCP 서버명 |
| **Notification** | `Notification` | 알림 발송 시 | `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog` |

**설계 함의**:
- v2.1까지 하네스는 `PreToolUse(Bash)` + `PostToolUse(Edit|Write|MultiEdit)` + `Stop` 3종만 사용. **v2.2부터는 아래를 추가 고려**:
  - `SubagentStop` — 서브에이전트 산출물 검증 (단, frontmatter `hooks:`만이 해당 subagent 라이프사이클 국한. settings.json 등록 시 전체 서브에이전트 fire)
  - `InstructionsLoaded` — 멀티모듈 로드 디버깅 ([Phase I-6](phase-i-monorepo.md#i-6-instructionsloaded-훅--로드-디버깅) 참고)
  - `PreCompact` — 압축 전 중요 상태 백업 (긴 세션 안전망)
  - `PermissionDenied` — 자동 거절 기록 후 `retry: true` 응답으로 재시도 유도 가능
  - `FileChanged` — 외부 프로세스가 생성·수정한 파일 감지 (예: generated code 핫리로드)

#### A-6-2. Handler 4종 (v2.2 신규)

**이전**: `"type": "command"`만 사용. **이제**: 용도에 맞게 선택.

| Handler | 스키마 핵심 | 용도 | 비용 |
|---|---|---|---|
| **`command`** | `"command": "./hook.sh"` | 결정론적 쉘 스크립트. 가장 일반적 | 실행 시간만 |
| **`http`** | `"url": "http://..."`, `"headers": {...}`, `"allowedEnvVars": [...]` | 중앙집중 검증 서버, 원격 감사 로그 | 네트워크 왕복 |
| **`prompt`** | `"prompt": "Evaluate: $ARGUMENTS"`, `"model": "claude-haiku"` | 가벼운 LLM 1-shot 판단 (예: 커밋 메시지 품질 평가) | Haiku 호출 비용 |
| **`agent`** | `"prompt": "Verify: ..."`, `"model": "claude-opus"` | 툴 접근 가진 서브에이전트로 복잡 검증 | 풀 세션 오버헤드 |

**예시 — `prompt` handler로 sudo 명령 자동 판단**:

```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "prompt",
    "if": "Bash(sudo *)",
    "prompt": "이 sudo 명령이 프로젝트에 안전한가? 컨텍스트: $ARGUMENTS",
    "model": "claude-haiku",
    "timeout": 30
  }]
}
```

**선택 기준**:
- 정책이 셸 regex로 표현 가능 → `command` (속도·예측성 최우선)
- 외부 시스템(감사 DB·Slack·SIEM)에 이벤트 송신 → `http`
- 맥락 의존 판단(모호한 패턴) → `prompt` (Haiku로 저비용)
- 복잡한 다단계 검증 (예: diff 전체를 정책과 대조) → `agent` (드물게)

#### A-6-3. Async hooks (v2.2 신규)

장시간 작업(>30s)을 **Claude 실행을 막지 않고** 백그라운드로:

```json
{
  "type": "command",
  "command": "./slow-audit.sh",
  "async": true            // 백그라운드 실행, 응답 기다리지 않음
}
```

Exit code 2로 **Claude를 깨우는** 변형:

```json
{
  "type": "command",
  "command": "./long-verify.sh",
  "asyncRewake": true      // 백그라운드, exit 2면 Claude 재활성화
}
```

**적용 케이스**:
- Stop 훅에서 Spotless + 테스트 풀 실행 (90+초) → `async: true`
- 긴 CI 검증을 asyncRewake로 걸어두고 Claude는 후속 작업 병행 → 실패 시만 통지

**주의**: async 훅의 `decision:block`은 **이미 지나간 턴에 영향 못 줌**. 실시간 차단이 필요하면 sync (`async: false` 기본).

#### A-6-4. Defer tool calls (v2.1.89+, v2.2 정식화)

SDK·커스텀 UI에서 **툴 호출을 외부 처리기로 일시 보류**:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "defer"
  }
}
```

SDK 클라이언트에 아래와 같이 전달됨:
```json
{
  "stop_reason": "tool_deferred",
  "deferred_tool_use": {
    "id": "toolu_01abc",
    "name": "AskUserQuestion",
    "input": { "questions": [...] }
  }
}
```

**주 용도**: 커스텀 UI가 사용자 확인을 비동기로 받아 재개. 일반 대화형 CLI에서는 사용 안 함.

#### A-6-5. 업그레이드된 PreToolUse 출력 스키마 (v2.2)

기존 `permissions.ask/allow/deny` 대신 **권한 규칙 on-the-fly 추가** 지원:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedPermissions": [{
        "type": "addRules",
        "rules": [{"toolName": "Bash", "ruleContent": "npm run *"}],
        "behavior": "allow",
        "destination": "localSettings"
      }],
      "message": "이유"
    }
  }
}
```

**활용**: 한 번 승인한 명령을 **자동으로 `settings.local.json`에 영구 저장** → [Phase D "permissions 위생"](../HARNESS_SETUP_GUIDE.md#phase-d-permissions-위생) 자동화.

#### A-6-6. `CLAUDE_ENV_FILE` 환경변수 영속화 (v2.2 공식)

`SessionStart` / `CwdChanged` / `FileChanged` 훅에서만 사용 가능한 환경변수 파일 경로. 훅이 `KEY=value` 형식으로 이 파일에 쓰면, 이후 **같은 세션의 모든 Bash 호출**에서 해당 변수가 주입된다.

```bash
#!/bin/bash
# .claude/hooks/init-archetype-env.sh (SessionStart hook)
ARCHETYPE=$(bash "$CLAUDE_PROJECT_DIR/.claude/hooks/lib/detect-archetype.sh" "$PWD")
echo "CLAUDE_ARCHETYPE=$ARCHETYPE" >> "$CLAUDE_ENV_FILE"
echo "CLAUDE_MODULE_ROOT=$(git rev-parse --show-toplevel)" >> "$CLAUDE_ENV_FILE"
```

**주 용도**:
- 아키타입·모듈 루트 캐싱 — 매 훅 호출마다 `detect-archetype.sh` 재실행 회피 ([Phase I-5](phase-i-monorepo.md) 효율화)
- 세션 시작 시 프로젝트별 경로(PATH / LD_LIBRARY_PATH 등) 주입
- CWD 변경 시 모듈별 환경 재계산

**제약**: PreToolUse/PostToolUse/Stop 등 **다른 훅 타입에서는 `$CLAUDE_ENV_FILE` 미할당**. 이 훅들에서 환경변수를 쓰려면 `.state/` 디렉터리에 수동 export하고 훅 내부에서 source하는 기존 방식 유지.
