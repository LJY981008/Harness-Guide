# Phase I — 멀티모듈/모노레포 스케일링 + Agent Teams

> ← **[HARNESS_SETUP_GUIDE.md](../HARNESS_SETUP_GUIDE.md)** (상위 가이드로 돌아가기)
>
> **관련 Phase**: [A (Hooks)](phase-a-hooks.md) · [C (도메인 Skills)](phase-c-skills.md) · [E (CLAUDE.md 슬림화)](../HARNESS_SETUP_GUIDE.md#phase-e-claude.md-슬림화)
>
> **관련 가이드**: [CLAUDE_MD_GUIDE.md](../CLAUDE_MD_GUIDE.md) · [SKILLS_GUIDE.md](../SKILLS_GUIDE.md) · [SUBAGENTS_GUIDE.md](../SUBAGENTS_GUIDE.md)
>
> **역할**: 10~100+ 모듈 대형 모노레포에서 하네스가 스케일하도록 설계. 아키타입 분류 · `.claude/rules/` · nested skills · `claudeMdExcludes` · `detect-archetype.sh` · `InstructionsLoaded` · Agent Teams.
>
> **전제**: Phase 0~E 기본 세팅 완료. 60+ 모듈 프로젝트는 Phase 0과 **동시 설계** 권장 (초기 아키타입 결정).

## 목차
- [I-0. 스케일 실패 패턴](#i-0-스케일-실패-패턴-이렇게-하면-망한다)
- [I-1. 아키타입 분류 (가장 먼저)](#i-1-아키타입-분류-가장-먼저)
- [I-2. Rules vs Skills 배치 전략](#i-2-rules-vs-skills-배치-전략)
- [I-3. 루트 CLAUDE.md 경량화](#i-3-루트-claudemd-경량화-60-모듈-기준)
- [I-4. `claudeMdExcludes` — 타 팀 CLAUDE.md 노이즈 제거](#i-4-claudemdexcludes--타-팀-claudemd-노이즈-제거)
- [I-5. Hook 스크립트 아키타입 분기](#i-5-hook-스크립트-아키타입-분기-detect-archetypesh)
- [I-6. `InstructionsLoaded` 훅 — 로드 디버깅](#i-6-instructionsloaded-훅--로드-디버깅)
- [I-7. `CLAUDE_CODE_NEW_INIT=1` — 대형 프로젝트 초기 세팅](#i-7-claude_code_new_init1--대형-프로젝트-초기-세팅)
- [I-8. Virtual Monorepo 패턴](#i-8-virtual-monorepo-패턴-모노레포-미확립-조직)
- [I-9. 적용 후 검증](#i-9-적용-후-검증-phase-i-전용)
- [I-10. Agent Teams via Nested Skills 패턴](#i-10-agent-teams-via-nested-skills-패턴)
- [I-11. 실측 권장 수치](#i-11-실측-권장-수치-60-모듈-기준)

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
