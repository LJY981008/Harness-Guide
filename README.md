# Claude Code 프로젝트 세팅 템플릿

새 프로젝트에 Claude Code 하네스 엔지니어링(CLAUDE.md / Skills / Hooks / Subagents / MCP / permissions / 정적분석 / CI 자동화)을 빠르게 도입하기 위한 가이드와 템플릿 모음.

**2026-04-15 기준 v2** · 공식 포맷(code.claude.com/docs) — SKILL.md 폴더 구조, Hooks 네이티브, context:fork 기반 매크로, Spotless/Checkstyle 정적분석, GitLab CI 주간 GC.

**v2 확장** (Phase F/G/H 추가):
- Phase F — Spotless + Checkstyle로 CODING_RULES 자동 강제
- Phase G — PostToolUse 포맷 + Stop 체인 자동 루프
- Phase H — GitLab schedule + Discord webhook 주간 GC

레퍼런스 구현체: [tbbe-hub/.claude](../tbbe-hub/.claude) — 영상 하네스 5대 요소 전 영역 A급 달성

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

**Hooks·settings·실제 Skills 예시는 복제본 대신 tbbe-hub 실전 구현을 그대로 참조**:
- Hooks 5종 (v2): [/home/code/project/tbbe-hub/.claude/hooks/](../tbbe-hub/.claude/hooks/)
  - `pre-bash-guard.sh` — PreToolUse 위험 명령 11종 `exit 2` 차단 (`permissions.deny` 버그 우회)
  - `verify-commit-msg.sh` — 커밋 태그 강제
  - `format-changed-java.sh` — PostToolUse 편집 파일 spotlessApply
  - `post-work-check.sh` — Stop 변경 모듈 한정 compile + checkstyle
  - `recommend-agent-on-stop.sh` — Stop 경로 매칭으로 서브에이전트 추천 (Agent `paths` 미지원 우회)
  - `spawn-reviewer-on-stop.sh` — diff 30줄+ 시 code-reviewer 호출 유도
- settings.json (Hooks 등록 완료본): [/home/code/project/tbbe-hub/.claude/settings.json](../tbbe-hub/.claude/settings.json)
- settings.local.json (와일드카드 압축): [/home/code/project/tbbe-hub/.claude/settings.local.json](../tbbe-hub/.claude/settings.local.json)
- build.gradle (Spotless + Checkstyle v2): [/home/code/project/tbbe-hub/build.gradle](../tbbe-hub/build.gradle)
- Checkstyle 설정: [/home/code/project/tbbe-hub/config/checkstyle/](../tbbe-hub/config/checkstyle/)
- 주간 GC 스크립트: [/home/code/project/tbbe-hub/scripts/weekly-gc.sh](../tbbe-hub/scripts/weekly-gc.sh)
- 운영 매크로 5종: [/home/code/project/tbbe-hub/.claude/skills/{dlq-check,prod-log,ai-diagnose,listing-status,stuck-check}/](../tbbe-hub/.claude/skills/)
- 도메인 Skills 19종 + review-diff: [/home/code/project/tbbe-hub/.claude/skills/](../tbbe-hub/.claude/skills/)

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

## 하네스 4계층 + MCP 비교

| 구성요소 | 위치 | 언제 쓰나 | 특징 |
|---|---|---|---|
| **정책** (CLAUDE.md) | 프로젝트 루트 | 모든 세션 system prompt 주입 | 짧고 핵심만 (200줄 이내). 절차는 skill로 위임. |
| **절차** (Skills) | `.claude/skills/<name>/SKILL.md` | 필요할 때만 로드 | frontmatter `description`/`paths`로 자동 매칭. 토큰 효율. |
| **강제** (Hooks) | `.claude/hooks/*.sh` + `settings.json` | 결정론적 게이트 | 에이전트 자율 준수 X. 성공은 침묵, 실패만 표면화. |
| **격리** (Subagents) | `.claude/agents/*.md` | 복잡한 반복 분석 | 별도 컨텍스트. 과다 생성 금지. |
| **도구** (MCP) | `claude mcp add ...` | 외부 시스템 연결 | 프로젝트 단위. 많으면 프롬프트 오염. |

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
# Harness-Guide
# Harness-Guide
