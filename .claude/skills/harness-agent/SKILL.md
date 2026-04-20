---
name: harness-agent
description: "지정 경로의 프로젝트 Claude Code 하네스 세팅을 5명의 도메인 전문가(정책·규칙·절차·강제·격리)가 병렬 평가 후, summarizer가 영상 5요소로 롤업한 리포트를 생성. 타깃 프로젝트는 읽기 전용, 리포트는 현재 프로젝트 .analyze/<project_name>/YYYY-MM-DD.md에 저장."
argument-hint: <absolute-project-path>
disable-model-invocation: true
---

# /harness-agent — 하네스 감사 오케스트레이터

$ARGUMENTS 경로의 프로젝트 하네스를 평가한다. 가이드 기준은 이 프로젝트의 [HARNESS_SETUP_GUIDE.md](/home/code/project/claude-setting/HARNESS_SETUP_GUIDE.md) v2.2.2.

## 수행 절차

1. **경로 검증 및 프로젝트명 추출**
   - `test -d "$ARGUMENTS"` 로 디렉터리 존재 확인. 없으면 즉시 중단하고 사용자에게 알림.
   - `basename "$ARGUMENTS"` 로 프로젝트 이름 추출 → 이후 `<project_name>`.

2. **스택 탐지 (조건부 확장 결정)**
   Bash·Glob으로 타깃 경로 내부를 확인:
   - `build.gradle` / `pom.xml` / `package.json`+ESLint/Prettier → Phase F 플래그 활성화
   - `.gitlab-ci.yml` / `.github/workflows/` / `azure-pipelines.yml` → Phase H 플래그
   - 최상위 모듈 디렉터리 10+ + `settings.gradle`/`pnpm-workspace.yaml`/`lerna.json` → Phase I 플래그
   결과를 변수로 보관해 각 auditor와 summarizer에 전달.

3. **5명 병렬 스폰** (단일 메시지에 5개 Agent 툴 호출)
   각 호출의 `subagent_type`과 프롬프트 핵심:
   - `harness-policy-auditor` — "타깃 경로: <path>. 정책(CLAUDE.md) 계층 평가."
   - `harness-rules-auditor` — "타깃 경로: <path>. 규칙(.claude/rules/) 계층 평가."
   - `harness-skills-auditor` — "타깃 경로: <path>. 절차(.claude/skills/) 계층 평가. Phase F/H/I 플래그: <F> <H> <I>."
   - `harness-hooks-auditor` — "타깃 경로: <path>. 강제(.claude/hooks/+settings.json) 계층 평가."
   - `harness-agents-auditor` — "타깃 경로: <path>. 격리(.claude/agents/) 계층 평가. Phase I 플래그: <I>."
   각 프롬프트에 **타깃 경로는 읽기 전용. .analyze/ 바깥 어디에도 쓰지 말 것** 명시.

4. **Summarizer 호출** (6번째 Agent 툴)
   `harness-summarizer`에게 위 5개 리포트를 전체 텍스트로 전달. Summarizer는 markdown 리포트 본문을 응답으로 반환.

5. **리포트 저장**
   - Glob으로 기존 파일 확인: `.analyze/<project_name>/YYYY-MM-DD*.md`
   - 없으면 `.analyze/<project_name>/YYYY-MM-DD.md`, N개 있으면 `YYYY-MM-DD-(N+1).md`
   - 필요 시 Bash `mkdir -p .analyze/<project_name>` 실행 후 Write 툴로 저장
   - 저장 경로를 사용자에게 출력

6. **최종 응답**
   사용자에게 출력할 내용만:
   - 저장 경로 (`.analyze/<project_name>/YYYY-MM-DD.md`)
   - 영상 5요소 요약 스코어카드 표 (summarizer 결과에서 발췌)
   - 상세는 해당 파일 참조 유도

## 리포트 헤더 템플릿

Write 툴로 작성할 때 summarizer 응답 앞에 프리앰블을 붙인다:

```markdown
# Harness Audit Report: <project_name>

**감사 일시**: YYYY-MM-DD HH:MM
**가이드 버전**: v2.2.2
**감사 대상**: <absolute-path>
**감지된 스택**: <stack 요약>
**조건부 확장 활성화**: F <✓|✗> · H <✓|✗> · I <✓|✗>

---

<summarizer 응답 본문 그대로>
```

## 제약

- 타깃 프로젝트에는 **절대 쓰지 않는다** — Read·Glob·Grep·읽기성 Bash만 사용
- `.analyze/` 외의 경로에는 쓰지 않는다
- 5명 스폰은 **반드시 단일 메시지에 병렬**로. 순차 호출 금지 (컨텍스트 격리 깨짐)
- Summarizer는 5명 결과 수신 완료 후 호출 (의존성)
- 같은 날 재실행 시 기존 파일 덮어쓰지 말고 `-N` 접미사 부여
