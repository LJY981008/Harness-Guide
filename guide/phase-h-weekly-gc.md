# Phase H — 주간 GC (GitLab Schedule + Discord)

> ← **[HARNESS_SETUP_GUIDE.md](../HARNESS_SETUP_GUIDE.md)** (상위 가이드로 돌아가기)
>
> **관련 Phase**: [F (정적분석)](phase-f-static-analysis.md) · [G (자동 루프)](../HARNESS_SETUP_GUIDE.md#phase-g-자동-루프-posttooluse--stop-체인)
>
> **역할**: 영상 하네스 엔지니어링 ④번 "가비지 컬렉션" 구현. **AI가 누적된 나쁜 패턴을 모범으로 착각하는 것 방지**. Claude 세션 외부(CI schedule)에서 주기 청소 봇 운용.

## 목차
- [H-1. weekly-gc.sh 스크립트](#h-1-weekly-gcsh-스크립트)
- [H-2. GitLab CI job](#h-2-gitlab-ci-job)
- [H-3. 기존 job들에 schedule 차단 규칙 추가 (중요)](#h-3-기존-job들에-schedule-차단-규칙-추가-중요)
- [H-4. GitLab Schedule 등록 (REST API)](#h-4-gitlab-schedule-등록-rest-api)
- [H-5. Discord webhook 알림](#h-5-discord-webhook-알림)
- [H-6. 수동 트리거 검증](#h-6-수동-트리거-검증)
- [H-7. 실측 시간](#h-7-실측-시간-7개-모듈-spring-boot-프로젝트-기준)

---

## Phase H: 주간 GC (GitLab Schedule + Discord)

> **목표**: 영상 하네스 엔지니어링 ④번 "가비지 컬렉션" 구현. AI가 누적된 나쁜 패턴을 모범으로 착각하는 것 방지.

### H-1. weekly-gc.sh 스크립트

템플릿: `../templates/scripts/weekly-gc.sh`

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
