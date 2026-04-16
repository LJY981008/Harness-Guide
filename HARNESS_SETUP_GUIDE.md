# Harness Setup Guide (2026-04 기준, v2.1)

> 새 프로젝트에 Claude Code 하네스 엔지니어링을 도입하는 완전 가이드. 2026 공식 포맷(code.claude.com/docs) 기준.
>
> **레퍼런스 구현**: 가이드의 모든 스크립트·설정은 실전 프로젝트에 적용해 **현재 동작 중인 것**을 기반으로 작성됨. `templates/` 디렉터리 복사 후 프로젝트에 맞춰 조정.
>
> **v2 변경점** (2026-04-15): Phase F(정적분석) · Phase G(자동 루프) · Phase H(주간 GC) 추가. Phase A Hook 3종 → 5종으로 확장. `permissions.deny` 버그와 Agent `paths` 미지원 등 **실측으로 확인된 공식 제약** 명시.
>
> **v2.1 변경점** (2026-04-16): 대형 모노레포(60+ 모듈) 대응 Phase I 추가. 2025 후반~2026초 신규 기능 반영 — `.claude/rules/` + `paths:` frontmatter, nested `.claude/skills/` 자동 발견, `claudeMdExcludes`, `InstructionsLoaded` 훅, `CLAUDE_CODE_NEW_INIT=1` 인터랙티브 /init. 하네스 4계층 → **5계층** (rules 레이어 추가).

---

## ⚠️ 알려진 공식 제약 (먼저 숙지)

이 제약은 실제 프로젝트 적용 과정에서 **실측으로 확인**됐으며, 하네스 설계 시 반드시 반영해야 한다:

| 제약 | 현상 | 우회 방법 |
|---|---|---|
| **`permissions.deny` 필드 버그** (GitHub [#6699](https://github.com/anthropics/claude-code/issues/6699), [#27040](https://github.com/anthropics/claude-code/issues/27040)) | v1.0.93+ 에서 `deny` 규칙 무시됨 — 차단되어야 할 명령이 실행됨 | **PreToolUse hook + `exit 2`**가 유일한 실제 차단 수단 (Phase A-0 참고) |
| **Agent frontmatter `paths` 공식 미지원** | Skill은 `paths` glob으로 자동 로드되나 Agent는 `description` 매칭만 지원 | `recommend-agent-on-stop.sh`로 경로 키워드 기반 추천 주입 (Phase G) |
| **PostToolUse/Stop `decision:block`은 피드백만** | 차단이 아닌 "Claude에게 메시지 주입" 수준. LLM이 무시 가능 | 진짜 물리적 강제는 **PreToolUse + exit 2**, CI 파이프라인, Checkstyle `ignoreFailures=false`만 가능 |
| **모노레포 계층 설정 지원 불완전** (GitHub [#12962](https://github.com/anthropics/claude-code/issues/12962) OPEN, [#37344](https://github.com/anthropics/claude-code/issues/37344) 2026-03-25 중복 종료) | 계층 자동 발견 지원 여부가 요소별 상이. `settings.json`/hooks/MCP는 **루트만**, `CLAUDE.md`/`skills`/`rules`는 **양방향 계층 지원**, subagents는 **CWD 기준 walking up만** | Phase I 참고 — hooks는 루트 몰빵 + 경로 분기 스크립트, skills·rules는 모듈 폴더 분산, subagents는 루트 등록 원칙 (모듈 단위 cd 워크플로우일 때만 모듈 `.claude/agents/` 발견) |
| **`paths:` frontmatter 부분 버그** (GitHub [#23478](https://github.com/anthropics/claude-code/issues/23478), [#21858](https://github.com/anthropics/claude-code/issues/21858)) | `.claude/rules/`의 `paths:` 매칭이 **Read 툴에만 트리거, Write에는 안 됨**. user-level(`~/.claude/rules/`)은 아예 `paths:` 무시 | 중요한 모듈 규칙은 Write 경로도 커버되도록 CLAUDE.md 계층 로드 병행, user-level은 `paths` 없는 전역 규칙으로만 사용 |
| **Subagent 공식 발견 위치 5곳만 유효** | managed settings / `--agents` CLI / `.claude/agents/` (루트 또는 CWD 상위) / `~/.claude/agents/` / plugin `agents/`. **스킬 폴더 내부 `.md`는 subagent로 인식 안 됨** | 스킬 폴더 안 `.md`는 "프롬프트 파일"로만 사용 가능. frontmatter `hooks:` 등을 정규 동작시키려면 **루트 `.claude/agents/`에 등록 필수** |

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

## Phase A: Hooks 5종

> **v2 변경점**: 이전 3종(verify-commit-msg, check-java-imports, post-work-check) → 5종. `check-java-imports.sh`는 Checkstyle(Phase F)로 흡수되며 삭제. 신규 4종 추가: `pre-bash-guard`(실제 차단), `format-changed-java`(PostToolUse), `recommend-agent-on-stop`(Stop 추천), `spawn-reviewer-on-stop`(Stop 리뷰 유도).

### A-0. 실제 차단 레이어 (PreToolUse Bash 가드) — 신규

`permissions.deny`가 버그로 비활성이므로 PreToolUse hook + `exit 2`가 **유일한 물리적 차단 수단**. 이게 없으면 나머지 하네스는 종이 호랑이.

템플릿: `templates/hooks/pre-bash-guard.sh` (프로젝트에 맞게 조정)

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

템플릿: `templates/settings.json.template`

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

## Phase B: 운영 매크로 Skills

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

[SKILLS_GUIDE.md](SKILLS_GUIDE.md) 참고. 공식 포맷:

```
.claude/skills/
└── <kebab-name>/
    ├── SKILL.md           # 필수. frontmatter + 본문
    ├── references/        # 선택. 상세 문서들
    ├── examples/          # 선택. 출력 예시
    └── scripts/           # 선택. 실행 스크립트
```

템플릿: [templates/skills/SKILL.md.template](templates/skills/SKILL.md.template)

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
- Phase I에서 대규모 활용 전략 상세.

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

## Phase F: 정적분석 승격 (Spotless + Checkstyle)

> **목표**: CODING_RULES.md의 자동화 가능 규칙(wildcard import 금지, 네이밍, import 순서)을 Checkstyle로 **물리적 강제화**. Phase A의 `decision:block`(유도)보다 강력.

### F-0. 전제 조건

- Java 21 + Gradle 8.x (Groovy DSL 또는 Kotlin DSL)
- 원격 브랜치 `origin/master`(또는 `origin/main`) 접근 가능

### F-1. build.gradle에 플러그인 추가

아래 예시는 Spring Boot 멀티모듈 Gradle 프로젝트 기준. 프로젝트 구조에 맞춰 `subprojects`/`allprojects` 블록 조정.

```groovy
plugins {
    // 기존 id들 + 아래 한 줄 추가
    id 'com.diffplug.spotless' version '8.4.0' apply false
}

subprojects {
    apply plugin: 'com.diffplug.spotless'
    apply plugin: 'checkstyle'

    spotless {
        ratchetFrom 'origin/master'  // 증분: 변경 파일만 검사
        java {
            target 'src/**/*.java'
            targetExclude 'build/generated/**/*'
            palantirJavaFormat('2.90.0')  // Lombok 친화
            removeUnusedImports()
            importOrder('java', 'javax', 'jakarta', 'org', 'com')
            formatAnnotations()
        }
    }

    checkstyle {
        toolVersion = '10.21.0'
        configFile = rootProject.file('config/checkstyle/checkstyle.xml')
        configProperties = [
            'suppressionFile': rootProject.file('config/checkstyle/suppressions.xml').toString()
        ]
        ignoreFailures = true   // 초기 도입: 리포트만, 빌드 실패 안 함
    }

    tasks.named('checkstyleTest') { enabled = false }

    // QueryDSL Q클래스가 main/test 공유 디렉토리 생성 시 implicit dep 해소
    tasks.named('checkstyleMain') {
        mustRunAfter(tasks.named('compileTestJava'))
    }
}
```

**버전 선정 근거**: 
- `palantir-java-format 2.90.0`: Java 21 공식 지원, Lombok `@Builder`/`@NoArgsConstructor` 호환
- `Spotless 8.4.0`: 최신 안정, Gradle 8.x 호환
- `Checkstyle 10.21.0`: Java 21 공식 지원
- **모두 실측 확인 버전** (추측 금지)

### F-2. Checkstyle 규칙 (최소 세트)

`config/checkstyle/checkstyle.xml`에 아래 최소 세트로 시작. 필요 시 점진 확장.

```xml
<module name="Checker">
    <property name="severity" value="error"/>
    <module name="SuppressionFilter">
        <property name="file" value="${suppressionFile}"/>
    </module>
    <module name="TreeWalker">
        <module name="AvoidStarImport"/>
        <module name="UnusedImports"><property name="processJavadoc" value="true"/></module>
        <module name="RedundantImport"/>
        <!-- Spotless importOrder와 정렬 일치 (CustomImportOrder는 그룹 분리 불일치로 오탐) -->
        <module name="ImportOrder">
            <property name="groups" value="/^java\./,/^javax\./,/^jakarta\./,/^org\./,/^com\./"/>
            <property name="ordered" value="true"/>
            <property name="separated" value="true"/>
            <property name="option" value="top"/>
            <property name="sortStaticImportsAlphabetically" value="true"/>
        </module>
        <module name="TypeName"/>
        <module name="MethodName"><property name="format" value="^[a-z][a-zA-Z0-9_]*$"/></module>
        <module name="ConstantName"/>
        <module name="PackageName"><property name="format" value="^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$"/></module>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        <module name="ParameterName"/>
    </module>
</module>
```

**FQN 검출(Regexp) 제외 이유**: 실측 48건 대부분 `new java.util.concurrent.XXX<>()` 같은 정당한 사용. 오탐 비용 > 이익.

### F-3. suppressions.xml

```xml
<suppressions>
    <suppress files="[\\/]build[\\/]generated[\\/].*" checks=".*"/>
    <suppress files=".*[\\/]Q[A-Z]\w+\.java" checks=".*"/>  <!-- QueryDSL Q클래스 -->
    <suppress files=".*[\\/]src[\\/]test[\\/].*" checks=".*"/>
    <!-- QueryDSL static final Q*Entity 필드 재익스포트 관용 -->
    <suppress files=".*QueryDslRepository\.java" checks="ConstantName"/>
</suppressions>
```

### F-4. 포맷 베이스라인 적용 (1회성 대규모 커밋)

```bash
# 1. ratchetFrom 임시 주석 처리 (전체 파일 대상)
# 2. 일괄 apply
./gradlew spotlessApply --no-daemon

# 3. 변경 규모 확인 (대형 멀티모듈 기준 500+ 파일 / 1만+ 라인 예상)
git diff --stat | tail -3

# 4. 컴파일 검증 (포맷이 기능에 영향 없는지)
./gradlew compileJava --no-daemon

# 5. 별도 커밋 (blame 오염 격리)
git commit -am "chore: spotless palantir 포맷 베이스라인 적용"

# 6. .git-blame-ignore-revs에 커밋 SHA append
echo "$(git log -1 --format=%H)" >> .git-blame-ignore-revs
git commit -am "chore: 포맷 베이스라인 커밋을 blame-ignore-revs에 등록"

# 7. ratchetFrom 재활성화
# 8. 로컬 git 설정 (1회)
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

GitHub/GitLab 17.10+는 `.git-blame-ignore-revs` 자동 인식 — UI blame에서 해당 커밋 스킵.

### F-5. wildcard import 자동 정리

실측: 대형 프로젝트에서 80+ 파일의 wildcard import를 **실제 사용 심볼만의 명시 import**로 자동 전개 가능.

**핵심 설계**:
- 프로젝트 소스 + **JDK jmods** + Gradle 캐시 jar에서 각 wildcard 패키지의 public 클래스 추출
- 본문 대문자 식별자와 교집합 → 명시 import 생성
- **이미 명시 import된 심볼은 제외** (Component 같은 모호 오탐 방지)
- 실행 후 `./gradlew spotlessApply`로 정렬·포맷 보정

### F-6. 엄격 모드 승격

위반 0건 달성 후:

```groovy
checkstyle {
    // ...
    ignoreFailures = false   // 위반 1건이라도 빌드 차단
    maxWarnings = 0
}
```

이제 `./gradlew build -x test`가 Checkstyle 위반 시 실패 → Phase A의 post-work-check가 block 반환 → 실질적 강제.

### F-7. 실측 결과 (7개 모듈 Spring Boot 프로젝트 기준)

| 항목 | Before | After |
|---|---|---|
| wildcard import | 81 파일 / 280+ 라인 | **0** |
| Checkstyle 위반 | 측정 불가 (규칙 없음) → 초기 1,347건 → 104건 (ImportOrder 정렬 교체 후) → 42건 (Spotless 적용 후) → **0** |
| ignoreFailures | n/a | `true` → `false` |
| 포맷 일관성 | 혼재 | palantir 2.90.0 통일 |
| 베이스라인 커밋 | n/a | 507 파일 (SHA는 .git-blame-ignore-revs) |

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

## Phase H: 주간 GC (GitLab Schedule + Discord)

> **목표**: 영상 하네스 엔지니어링 ④번 "가비지 컬렉션" 구현. AI가 누적된 나쁜 패턴을 모범으로 착각하는 것 방지.

### H-1. weekly-gc.sh 스크립트

템플릿: `templates/scripts/weekly-gc.sh`

4가지 점검:
1. **Checkstyle 위반 집계** — XML 리포트의 `<error>` 태그 카운트 (빌드 캐시 영향 없이 정확)
2. **TODO/FIXME/HACK/XXX 스캔** — 잊혀진 숙제 추적
3. **agent-memory 30일 경과 → archive 이동** — 컨텍스트 부패 방지
4. **레거시 참조 수집** — CLAUDE.md에서 deprecated 표시된 모듈/클래스 잔존 참조

출력: `docs/maintenance/YYYY-MM-DD/{SUMMARY.md, checkstyle.log, todos.txt, legacy-refs.txt}`

### H-2. GitLab CI job

아래 예시를 `.gitlab-ci.yml`의 기존 stages 뒤에 추가.

```yaml
weekly-gc:
  stage: test
  image: gradle:8.5-jdk21
  cache: { <<: *gradle_cache }
  before_script:
    - chmod +x ./gradlew scripts/weekly-gc.sh
  script:
    - bash scripts/weekly-gc.sh
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $GC_JOB == "weekly"'
  artifacts:
    paths: [docs/maintenance/]
    expire_in: 30 days
  allow_failure: true  # GC 실패로 다른 파이프라인 차단 금지
```

### H-3. 기존 job들에 schedule 차단 규칙 추가 (중요)

**부작용 주의**: schedule pipeline도 branch는 master라 `if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH` 규칙이 매칭됨 → **매주 월요일 GC와 함께 build/deploy 자동 실행**.

모든 배포·빌드 job의 rules에 아래 prepend:

```yaml
rules:
  - if: '$CI_PIPELINE_SOURCE == "schedule"'
    when: never
  - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

`when: manual` 포함된 job도 동일하게 추가 (실수 클릭 방지 이중화).

### H-4. GitLab Schedule 등록 (REST API)

```bash
TOKEN="your-project-access-token"
PROJECT_ID="<your-project-id>"  # GitLab 프로젝트 ID

# Schedule 생성
curl -X POST \
  --header "PRIVATE-TOKEN: $TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "description": "Weekly GC",
    "ref": "master",
    "cron": "0 3 * * 1",
    "cron_timezone": "Asia/Seoul",
    "active": true
  }' \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/pipeline_schedules"

# 응답에서 id 확인 후 변수 추가
SCHEDULE_ID="2"
curl -X POST \
  --header "PRIVATE-TOKEN: $TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"key":"GC_JOB","value":"weekly","variable_type":"env_var"}' \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/pipeline_schedules/$SCHEDULE_ID/variables"
```

### H-5. Discord webhook 알림

1. Discord 채널 → 설정 → 연동 → 웹후크 생성 → URL 복사
2. GitLab CI variable 등록 (masked):

```bash
curl -X POST \
  --header "PRIVATE-TOKEN: $TOKEN" \
  --data-urlencode "key=DISCORD_WEBHOOK_URL" \
  --data-urlencode "value=<DISCORD_WEBHOOK_URL>" \
  --data "masked=true" \
  --data "protected=false" \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/variables"
```

`weekly-gc.sh`가 마지막에 요약 메시지 전송:

```bash
if [[ -n "${DISCORD_WEBHOOK_URL:-}" ]]; then
  MSG="주간 GC: Checkstyle ${VIOLATIONS}건 · TODO ${TODO_COUNT}건 · Archived ${ARCHIVED}개 · Legacy ${LEGACY_COUNT}건"
  curl -s -X POST "$DISCORD_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{\"content\":\"$MSG\"}" > /dev/null 2>&1 || true
fi
```

`|| true`로 webhook 실패가 파이프라인 전체 실패로 전파되지 않게.

### H-6. 수동 트리거 검증

```bash
curl -X POST --header "PRIVATE-TOKEN: $TOKEN" \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/pipeline_schedules/$SCHEDULE_ID/play"

# 파이프라인 상태 확인
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "https://gitlab.example.com/api/v4/projects/$PROJECT_ID/pipelines?per_page=1"
```

Discord 채널에 알림 수신되면 성공.

### H-7. 실측 시간 (7개 모듈 Spring Boot 프로젝트 기준)

| 단계 | 시간 |
|---|---|
| Docker image pull | ~30s |
| Git clone + Gradle init | ~1min |
| checkstyleMain `--rerun-tasks` (7 모듈) | ~4~6min |
| 나머지 (TODO/archive/legacy) | 수 초 |
| **총** | **~9분** |

**최적화 여지**: `--rerun-tasks` 제거 시 3~4분으로 단축 가능 (CI는 어차피 캐시 없음).

---

## Phase I: 멀티모듈/모노레포 스케일링 (v2.1)

> **목표**: 10~100+ 모듈 대형 모노레포에서 하네스가 스케일하도록 설계. Skills·Rules의 계층 자동 발견, `claudeMdExcludes` 노이즈 제거, 아키타입 기반 훅 분기, `InstructionsLoaded` 훅으로 로드 디버깅.
>
> **전제**: Phase 0~E 기본 세팅 완료. 60+ 모듈 프로젝트는 Phase 0과 **동시 설계** 권장 (초기 아키타입 결정).

### I-0. 스케일 실패 패턴 (이렇게 하면 망한다)

| 안티패턴 | 이유 |
|---|---|
| 모듈마다 CLAUDE.md 60개 작성 | 공통 규칙 중복 → 동기화 비용 폭발. 한 규칙 바꾸려면 60군데 |
| 모듈마다 Skill 폴더 60개 | Skill 인덱스 비대화, `description` 매칭 품질 하락 |
| Hook 스크립트에 60-way case문 | 유지보수 지옥, 새 모듈 추가 시 훅 수정 필수 |
| `settings.json`을 모듈별로 분리 | **구조상 불가능** — 루트 1개만 로드됨 (Issue #12962 OPEN) |
| `~/.claude/rules/`에 `paths:` 사용 | user-level rules는 `paths:` 무시 (Issue #21858 BUG) |

### I-1. 아키타입 분류 (가장 먼저)

60개 모듈 = 개별 60개가 아니라 **5~8개 아키타입**:

```
web-layer     (Controller, REST)       — 10개
data-layer    (JPA, Repository)        — 15개
batch         (Scheduler, Job)         — 8개
shared-lib    (DTO, util)              — 12개
integration   (외부 API 클라이언트)    — 8개
legacy        (건드리면 안 됨)         — 7개
```

**판별 기준**: "이 두 모듈에 같은 규칙이 적용되는가"로 묶음. 애매하면 더 거칠게 묶는 편이 유지보수 쉬움.

Skills·Rules 작성은 **아키타입 단위**, 예외는 개별 모듈에서 override.

### I-2. Rules vs Skills 배치 전략

```
<repo>/.claude/
├── rules/                          ← 짧은 경로 스코프 규칙 (v2.1 권장)
│   ├── global.md                   ← paths 없음 → 항상 로드
│   ├── web-layer.md                ← paths: ["modules/*-web/**"]
│   ├── data-layer.md               ← paths: ["modules/*-data/**"]
│   ├── payment.md                  ← paths: ["modules/payment-*/**"]
│   └── legacy.md                   ← paths: ["modules/legacy-*/**"]
│
├── skills/                         ← 지원 파일이 필요한 절차·매크로
│   ├── coding-rules/               ← 전역 (paths 없음)
│   ├── dlq-check/                  ← 운영 매크로 (slash command)
│   └── payment-deploy/             ← paths: ["modules/payment-*/**"]
│
├── agents/                         ← 루트만 로드. description에 모듈 키워드
│   └── payment-pipeline-diag.md
│
└── settings.json                   ← 루트 1개 (hooks·permissions)

<repo>/modules/payment-core/
├── CLAUDE.md                       ← 이 모듈 작업 시 자동 로드 (on-demand)
└── .claude/skills/                 ← ✅ v2.1 nested 지원
    └── payment-migration/
        └── SKILL.md
```

**배치 결정 트리**:
```
규칙인가 절차인가?
├─ 짧은 규칙 (글로 충분) → .claude/rules/ + paths:
│  └─ 지원 파일 필요? → 아니면 Rule, 맞으면 Skill로 격상
└─ 절차/매크로/슬래시 커맨드 → .claude/skills/
   ├─ 전 모듈 공통 → 루트 .claude/skills/
   └─ 특정 모듈 전용 → modules/<name>/.claude/skills/ (nested)
```

### I-3. 루트 CLAUDE.md 경량화 (60+ 모듈 기준)

루트 CLAUDE.md는 **150줄 이내** 유지. 모듈 목록을 전부 열거하지 말고 아키타입만:

```markdown
# <Project>

## Module Archetypes
- **web-layer** (modules/*-web): REST/Controller
- **data-layer** (modules/*-data): JPA/Repository
- **batch** (modules/batch-*): Scheduler/Job
- **shared-lib** (modules/shared-*): DTO/util
- **integration** (modules/integration-*): 외부 API
- **legacy** (modules/legacy-*): 수정 금지

자세한 모듈 ↔ 팀 매핑: @docs/modules-index.md
아키타입별 규칙: .claude/rules/<archetype>.md

## Common
- Java 21, Gradle 8.x
- 빌드: `./gradlew :module:compileJava`
- 커밋 태그: feat/fix/refactor/docs/test/chore/perf

## Skills 인덱스
(생략 — 아키타입·매크로·에이전트 목록)
```

**개별 모듈 CLAUDE.md는 예외 규칙만**: legacy(수정 금지), 보안 경계(승인 필수), 외부 API(요율 제한) 같은 **아키타입으로 안 잡히는 것**만.

### I-4. `claudeMdExcludes` — 타 팀 CLAUDE.md 노이즈 제거

여러 팀이 한 모노레포 공유 시, 내 작업 폴더 위로 올라가면서 다른 팀 CLAUDE.md가 자동 로드되어 컨텍스트 오염.

`.claude/settings.json` 또는 `.claude/settings.local.json`:

```json
{
  "claudeMdExcludes": [
    "**/modules/other-team/CLAUDE.md",
    "/absolute/path/to/legacy/.claude/rules/**"
  ]
}
```

- 절대 경로 또는 glob 패턴
- 여러 settings 레이어(user/project/local/managed)에 걸쳐 **배열 merge**
- **Managed policy CLAUDE.md는 제외 불가** (조직 정책 보호)

**언제 쓰나**: 본인 작업 모듈이 `modules/my-team/**`인데 `modules/other-team/CLAUDE.md`가 계속 딸려 올라와서 토큰 낭비·규칙 혼선 유발할 때.

### I-5. Hook 스크립트 아키타입 분기 (`detect-archetype.sh`)

`settings.json`은 루트 1개라 **훅 스크립트 내부에서 분기**. 60-way case가 아닌 **아키타입 매칭**으로 유지보수 가능하게.

`.claude/hooks/lib/detect-archetype.sh`:

```bash
#!/usr/bin/env bash
# 공용 함수: 파일 경로 → 아키타입 매핑
detect_archetype() {
  case "$1" in
    modules/*-web/*|modules/gateway*) echo "web-layer" ;;
    modules/*-data/*|modules/*-repo/*) echo "data-layer" ;;
    modules/batch-*) echo "batch" ;;
    modules/shared-*) echo "shared-lib" ;;
    modules/integration-*) echo "integration" ;;
    modules/legacy-*) echo "legacy" ;;
    *) echo "default" ;;
  esac
}
```

`pre-bash-guard.sh`에서 사용:

```bash
#!/usr/bin/env bash
source "$CLAUDE_PROJECT_DIR/.claude/hooks/lib/detect-archetype.sh"

INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""')
FILES=$(echo "$INPUT" | jq -r '.tool_input.file_paths // "" | tostring')

for f in $FILES; do
  arch=$(detect_archetype "$f")
  if [[ "$arch" == "legacy" ]]; then
    echo "⛔ legacy 모듈(${f})은 직접 수정 금지. 팀 리드 승인 후 별도 브랜치 사용." >&2
    exit 2
  fi
done

# 기존 위험 명령 매칭 로직...
```

`post-work-check.sh`에서 변경 모듈 한정 빌드:

```bash
CHANGED=$(git status --porcelain | awk '{print $2}' | grep '\.java$')
MODULES=$(echo "$CHANGED" | awk -F/ '{print $1 "/" $2}' | sort -u)

TASKS=""
for m in $MODULES; do
  arch=$(detect_archetype "$m/x")
  case "$arch" in
    web-layer|data-layer|batch|integration)
      TASKS="$TASKS :${m//\//:}:compileJava :${m//\//:}:checkstyleMain" ;;
    shared-lib)
      TASKS="$TASKS :${m//\//:}:compileJava :${m//\//:}:checkstyleMain :${m//\//:}:test" ;;
    legacy)
      echo "⚠️ legacy 모듈 변경 감지: $m — 빌드 스킵, 리뷰어 확인 필수" >&2 ;;
  esac
done

[[ -n "$TASKS" ]] && ./gradlew $TASKS --no-daemon
```

**장점**:
- 새 모듈 추가 시 훅 수정 불필요 (naming convention만 맞으면 됨)
- 팀별 규칙 변경은 `detect-archetype.sh` 또는 아키타입 rule 파일 수정 한 곳에서
- 60개 모듈도 6개 case로 압축

### I-6. `InstructionsLoaded` 훅 — 로드 디버깅

대형 모노레포에서 "왜 이 규칙이 적용/미적용됐지?"를 추적하기 위한 공식 훅.

`.claude/settings.json`:

```json
{
  "hooks": {
    "InstructionsLoaded": [{
      "hooks": [{
        "type": "command",
        "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/log-loaded-instructions.sh",
        "timeout": 5
      }]
    }]
  }
}
```

`.claude/hooks/log-loaded-instructions.sh`:

```bash
#!/usr/bin/env bash
LOG="$CLAUDE_PROJECT_DIR/.claude/hooks/.state/instructions-loaded.log"
mkdir -p "$(dirname "$LOG")"
{
  echo "=== $(date -Iseconds) session=$CLAUDE_SESSION_ID ==="
  cat  # 훅 입력(로드된 파일 목록 JSON)을 그대로 기록
  echo
} >> "$LOG"
exit 0
```

**활용**: 세션 시작 후 `/memory` 명령으로 로드된 파일 목록 확인, 로그 비교로 기대와 실제 차이 파악. 특히 **`.claude/rules/` `paths:` 매칭이 Write 툴에서 안 뜨는 버그**(Issue #23478) 같은 상황 추적에 필수.

### I-7. `CLAUDE_CODE_NEW_INIT=1` — 대형 프로젝트 초기 세팅

공식 신규 인터랙티브 `/init`:

```bash
CLAUDE_CODE_NEW_INIT=1 claude
```

그 후 세션에서:

```
/init
```

**동작**:
1. 어떤 아티팩트(CLAUDE.md / skills / hooks) 설정할지 묻기
2. **서브에이전트가 코드베이스 탐색** (multi-phase)
3. 누락된 정보 follow-up 질문
4. 검토 가능한 제안서 제시 → 승인 후 파일 생성

**60+ 모듈 프로젝트에 특히 유용**: 수동으로 모든 아키타입 파악하기 벅찬 규모에서 서브에이전트가 1차 탐색 → 인간이 검토. Phase 0 직후 실행 권장.

### I-8. Virtual Monorepo 패턴 (모노레포 미확립 조직)

Owen Zanzal이 2026-03 Medium에 정리한 **멀티 repo → 가상 monorepo 패턴**. 35개 repo를 한 Claude 세션에서 참조 가능하게 묶음.

```bash
# 상위 디렉터리에 virtual monorepo 생성
mkdir ~/virtual-monorepo
cd ~/virtual-monorepo
git clone <repo-1>
git clone <repo-2>
...

# 최상위 .claude/ 설정 + CLAUDE.md로 통합 규약
```

`--add-dir` 조합:

```bash
claude --add-dir ~/other-repo
# + CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 로 해당 CLAUDE.md도 로드
```

**한계**: MCP·settings는 여전히 주 프로젝트만. Skills는 `--add-dir`의 `.claude/skills/`도 자동 로드(공식 예외).

### I-9. 적용 후 검증 (Phase I 전용)

```bash
# 1. nested skills 실제 발견 여부 — 모듈 파일 편집 중 세션에서
# "What skills are available?" → 해당 모듈 skill 포함돼야 함

# 2. claudeMdExcludes 동작 — /memory 명령으로 로드된 CLAUDE.md 목록 확인
#    excludes 패턴 매칭 파일이 빠져 있어야 함

# 3. InstructionsLoaded 훅 로그
cat .claude/hooks/.state/instructions-loaded.log | head -50

# 4. detect-archetype 함수 테스트
bash -c 'source .claude/hooks/lib/detect-archetype.sh; detect_archetype "modules/payment-core/x.java"'
# → payment (또는 아키타입 명)

# 5. 루트 CLAUDE.md 크기
wc -l CLAUDE.md
# → 150줄 이내 권장 (초과 시 @imports로 분리)

# 6. .claude/rules/ paths frontmatter 검증
for f in .claude/rules/*.md; do
  head -10 "$f" | grep -q '^paths:' && echo "$f: paths-scoped" || echo "$f: global"
done
```

### I-10. Agent Teams via Nested Skills 패턴

> **목적**: 스킬이 Agent 툴을 다단계 호출해 **모듈별 멀티스텝 파이프라인**을 캡슐화한다. 두 변형이 존재하며 각각의 용도·제약을 명확히 구분해야 한다.

#### I-10-1. 두 변형의 정의

| 변형 | 구조 | frontmatter `hooks:` 작동 | 용도 |
|---|---|---|---|
| **패턴 A** (프롬프트 파일 모듈화) | Agent `.md`를 **스킬 폴더 내부**(`.claude/skills/xxx/agents/`)에 둠 | ❌ **작동 안 함**. 단순 프롬프트 텍스트로 LLM에 전달됨 | 경량 프롬프트 분할·재사용. 훅 추적 불필요 시. |
| **패턴 B** (공식 subagent + 스킬 오케스트레이터) | Agent `.md`를 **루트 `.claude/agents/`**에 등록, 스킬은 `subagent_type:`으로 호출 | ✅ **작동**. PreToolUse/PostToolUse/SubagentStop 모두 fire | 훅 기반 자동 추적·검증이 필요한 재사용 파이프라인. |

**선택 기준**: 훅을 통한 **결정론적 추적·검증**이 요구되면 반드시 **패턴 B**를 선택한다. 패턴 A의 frontmatter는 LLM이 텍스트로 읽고 자율 판단하는 비결정 영역이다.

#### I-10-2. 패턴 A 구조 (프롬프트 파일 모듈화)

```
modules/payment/.claude/skills/payment-pipeline/
├── SKILL.md                    ← v2.1 nested 자동 발견 ✅
├── agents/                     ← ⚠️ Claude Code 공식 subagent 디렉터리 아님
│   ├── analyzer.md             ← 프롬프트 파일. hooks frontmatter 정의해도 파싱 X
│   ├── planner.md
│   ├── implementor.md
│   └── verifier.md
├── references/                 ← 공용 참조
│   └── domain-rules.md
└── temp/                       ← 중간 산출물
    ├── 01_analysis.md
    ├── 02_plan.md
    └── ...
```

스킬이 `Agent({subagent_type: "general-purpose", prompt: "analyzer.md를 Read해서..."})` 형식으로 호출하면 **general-purpose 에이전트**가 실행되어 `.md`를 입력 텍스트로 읽는다. `agents/` 폴더 명은 순수 관례일 뿐이다.

#### I-10-3. 패턴 B 구조 (공식 등록 + 오케스트레이터)

```
<repo>/.claude/agents/                        ← 공식 subagent 위치
├── payment-analyzer.md                        ← frontmatter hooks 정의 → 실제 fire
├── payment-planner.md
├── payment-implementor.md
└── payment-verifier.md

<repo>/modules/payment/.claude/skills/payment-pipeline/
├── SKILL.md                                   ← Agent({subagent_type: "payment-analyzer"}) 순차 호출
└── temp/                                      ← 중간 산출물 (agent의 PostToolUse 훅으로 검증)
```

Agent frontmatter 예시:
```yaml
---
name: payment-analyzer
description: 결제 모듈 분석 전담
hooks:
  PostToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/hooks/verify-payment-output.sh"
  Stop:
    - hooks:
        - type: command
          command: "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-analysis-result.sh"
  # 위 Stop은 런타임에 SubagentStop으로 변환되어 이 agent 종료 시만 fire
---
```

이 훅들은 **해당 subagent 라이프사이클에만** fire되며, 다른 agent·메인 세션에는 영향 없다.

#### I-10-2. SKILL.md 파이프라인 정의

```yaml
---
name: payment-pipeline
description: 결제 모듈 다단계 구현 파이프라인 (분석→계획→구현→검증)
paths: ["modules/payment-*/**"]
---

## 파이프라인 실행 순서 (Agent Teams)

4개의 에이전트를 `Agent` 도구로 순차 스폰한다.
각 에이전트 응답에 `[DONE]`이 포함되면 다음 단계로 진행.

### Step 1 — Analyzer 스폰
```
Agent({
  description: "결제 기획서 분석",
  subagent_type: "Explore",
  prompt: "프로젝트 루트: {절대경로}
           .claude/skills/payment-pipeline/agents/analyzer.md를 Read하여 지시에 따라 실행하라.
           결과는 temp/01_analysis.md에 저장."
})
```
→ `[DONE]` 수신 → `temp/01_analysis.md` 확인

### Step 2 — Planner 스폰
(분석 결과를 입력으로 계획 수립)

### Step 3 — Implementor 스폰
(계획에 따라 코드 작성)

### Step 4 — Verifier 스폰
(테스트·빌드 검증, 실패 시 롤백 지시)
```

#### I-10-4. 공통 장점

- **모듈별 파이프라인 캡슐화**: `/payment-pipeline` 단일 호출로 다단계 실행
- **컨텍스트 격리**: 각 스텝의 내부 탐색이 메인에 누적되지 않아 메인 컨텍스트 보호
- **단계별 재사용**: 단일 프롬프트·agent 수정이 전 파이프라인에 반영
- **모델 조합 최적화**: 읽기 전용 단계는 `Explore`(Haiku), 중요 단계는 `general-purpose`(Opus) 등 `subagent_type`으로 비용 조정

#### I-10-5. 공통 단점

**토큰 비용**:

| 항목 | 대략 토큰 |
|---|---|
| System prompt (툴 목록·시스템 룰) | 5~10k |
| CLAUDE.md 재로드 | 1~5k |
| Skill descriptions 리스트 | 1~3k |
| 전달 받은 프롬프트 | 0.2~1k |
| 지시받은 `.md` 파일 Read (패턴 A) 또는 agent 시스템 프롬프트 (패턴 B) | 1~10k |
| **Agent 호출 1회당 고정비** | **~8~20k 토큰** |

- 4단계 파이프라인의 고정 오버헤드만 30~80k 토큰 (실작업 토큰 별도)
- Prompt cache는 fork마다 새 세션이므로 공유되지 않는다
- 단계별 작업량이 작으면 오버헤드가 실작업을 초과한다. 메인 직접 처리가 유리한 케이스도 있다.

**레이턴시**:
- 순차 실행 시 단일 처리 대비 N배 시간
- fork 시작마다 초기화 오버헤드
- 독립 단계의 병렬화(한 메시지에 Agent 툴 다중 호출)로만 완화 가능

**상태 전달**:
- 서브에이전트는 이전 단계 컨텍스트와 완전 격리되므로 결과 전달은 반드시 파일로 기록한다
- 파일명·스키마 규약 이탈 시 다음 단계가 실패한다

**비결정성**:
- LLM 호출이 독립이므로 동일 입력도 다른 출력이 가능하다
- 포맷 규약 이탈이 후속 단계 실패를 유발하므로 CI에서 간헐 실패가 발생할 수 있다

#### I-10-6. 패턴 A 고유 단점 (프롬프트 파일 방식)

**관측성 저하**:
- 서브에이전트 내부 대화가 메인에 노출되지 않아 디버깅이 어렵다
- 실패 시 요약만 반환되므로 원인 추적이 제한된다
- 중간 산출물 파일(`temp/*.md`)이 유일한 디버깅 흔적이다

**훅 미작동**:
- `.md` frontmatter의 `hooks:` 정의는 Claude Code가 파싱하지 않는다
- LLM이 텍스트로 읽고 자율 판단하여 지시를 따르거나 무시한다 → **비결정적 동작**

**안티패턴 — 문서 위치 비결정성 사례**:
```
증상: Agent A가 temp/01.md에 결과를 써야 하는데 가끔 다른 경로에 저장.
      Agent B가 temp/01.md를 찾지 못해 파이프라인 중단.

원인: agent A의 frontmatter에 `hooks: PostToolUse: ... 경로 검증` 정의했으나,
      해당 `.md`가 스킬 폴더 내부 파일이므로 런타임 훅으로 등록되지 않음.
      Claude Code는 이 frontmatter를 무시하고 전체 내용을 LLM 프롬프트로 전달.
      LLM이 경로 규칙을 매번 다르게 해석하여 비결정 동작 발생.

해결: 훅 기반 결정론적 검증이 필요하면 패턴 B로 전환. agent를 루트 등록.
```

**에러 핸들링 취약**:
- `[DONE]` 같은 문자열 매칭으로 완료 감지 → 포맷 이탈 시 파이프라인 중단
- 중간 실패 시 재시작 메커니즘이 없어 처음부터 재실행 필요

#### I-10-7. 패턴 B 고유 이점

**결정론적 훅 fire**:
- `PostToolUse`, `SubagentStop` 등이 정의된 subagent 실행 시 **반드시** fire
- 셸 스크립트 기반 검증 → LLM 해석 여지 없음

**자연스러운 훅 분리**:
- 목적별 subagent(`search-analyzer`·`feature-implementor`·`migration-runner`)마다 각자의 frontmatter 훅을 가진다
- 전역 `settings.json` 훅의 "모든 툴 호출에 fire" 문제와 달리 **해당 agent 실행 시에만** fire되어 스크립트 내부 분기 불필요
- nigun_01 사례의 "서칭/기능 추가/수정/마이그레이션별 다른 훅" 요구가 자연스럽게 해결

**도구 제한 명시**:
- `allowed-tools`, `model` 등 frontmatter 필드로 subagent별 권한·모델 정밀 제어

#### I-10-8. 도입 판단 기준

**패턴 A 적합**: 경량 프롬프트 분할이 목적이고 훅 기반 검증이 불필요한 경우. 예: 문서 생성 파이프라인, 단발성 분석.

**패턴 B 적합**: 결정론적 추적·검증이 필요한 재사용 파이프라인. CI 통합이 필요한 워크플로우. 증상 디버깅이 필수인 운영 환경.

**둘 다 피해야 할 경우**:
- 단계별 작업이 짧음 (각 5~10 툴 호출) → 오버헤드가 실작업 초과
- 단계 간 상태 공유 과다 → 파일 전달 복잡도 폭증
- 1회성 작업 → 설계 비용 회수 불가

#### I-10-9. 완화책

토큰·레이턴시 절감:

1. **`subagent_type` 지정**으로 읽기 전용 단계는 `Explore`(Haiku) 사용 (Opus 대비 25배 저렴)
2. **파이프라인 단계 최소화** — 4단계를 2~3단계로 통합 가능한 지점 탐색
3. **프롬프트·시스템 파일 500줄 이하** — 공용 내용은 `references/`로 분리
4. **단계 간 전달은 요약만** — 전문 전달 지양, 500자 이내 요약 강제
5. **독립 단계 병렬화** — 한 메시지에 Agent 툴 다중 호출

#### I-10-10. 검증

```bash
# 1. nested skill 자동 발견 확인
# 해당 모듈 파일 편집 중 세션에서: "What skills are available?"
# → payment-pipeline 포함 여부 확인

# 2. 패턴 B — agent가 루트에 등록되었는지 확인
ls .claude/agents/payment-*.md

# 3. 패턴 A 사용 시 temp/ runtime 디렉터리 .gitignore 등록 확인
grep -q "temp/" .gitignore || echo "⚠️ temp/ gitignore 누락"

# 4. 패턴 B 훅 동작 확인
# /payment-pipeline 실행 → .claude/hooks/.state/ 로그 또는
# agent의 PostToolUse 커맨드 실행 흔적 확인

# 5. 토큰 비용 벤치마크
# /payment-pipeline 실행 후 Claude Code `/cost` 또는 세션 토큰 사용량
# → 메인 직접 처리 대비 1.5~3배 범위 예상
```

#### I-10-11. 요약

- Agent Teams 패턴은 **복잡한 재사용 파이프라인 전용**이다. 단순 모듈 규칙 적용은 nested skill + `paths:`만으로 충분하다.
- **훅 기반 결정론적 검증이 필요하면 반드시 패턴 B**를 채택한다. 패턴 A의 frontmatter 훅은 동작하지 않는다.
- 패턴 A는 경량 프롬프트 분할 목적에만 적합하며, 훅 추적이 필요하면 패턴 B로 전환해야 한다.

### I-11. 실측 권장 수치 (60+ 모듈 기준)

| 항목 | 임계선 | 초과 시 |
|---|---|---|
| 루트 CLAUDE.md | 150줄 | `@imports` 또는 `.claude/rules/`로 분리 |
| `.claude/rules/` 파일 수 | 15~25개 | 아키타입 재분류 검토 |
| 루트 `.claude/skills/` | 20개 | 모듈별 nested로 분산 |
| `.claude/agents/` | 5~10개 | 중복 description 통합 |
| 개별 모듈 CLAUDE.md | 전체 10% 이하 | 아키타입으로 이관 가능한지 재검토 |
| Hook 스크립트 라인 수 | 각 100줄 이내 | `lib/`로 공용 함수 추출 |

---

## 적용 후 검증

```bash
# 1. Hook 등록 확인
jq '.hooks | keys' .claude/settings.json
# → ["PostToolUse","PreToolUse","Stop"]

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
