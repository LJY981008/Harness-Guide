# 스킬(Skills) 작성 가이드

## 스킬이란?

`.claude/skills/*.md` 파일들. CLAUDE.md에서 BLOCKING 규칙으로 참조되어 **필요할 때만** 컨텍스트에 로드된다. CLAUDE.md를 짧게 유지하면서 도메인 디테일을 깊게 문서화할 수 있는 핵심 장치.

## 언제 스킬을 만드는가

- CLAUDE.md가 200줄을 넘으려 할 때 → 일부를 스킬로 분리
- 같은 설명을 두 번 이상 한 적 있을 때 → 스킬로 박제
- 도메인 특유 규칙(외부 시스템, 비즈니스 플로우 등)이 있을 때
- 신규 작업자가 헷갈려할 만한 부분

## 스킬 분류 (검증된 카테고리)

실전 멀티모듈 Spring Boot 프로젝트에서 적용된 15~20종 스킬의 카테고리:

| 카테고리 | 예시 파일 | 용도 |
|---|---|---|
| **코딩 규칙** | `coding-rules/SKILL.md` | import, naming, API 작성 규칙 |
| **아키텍처 패턴** | `patterns/SKILL.md` | 모듈 경계, TX 규칙, 의존성 방향 |
| **외부 시스템** | `external-systems/SKILL.md` | 외부 API/DB 접근법, 서버 IP |
| **데이터 플로우** | `<domain>-flow/SKILL.md`, `flow-diagrams/SKILL.md` | 핵심 비즈니스 플로우 |
| **상태 머신** | `status-lifecycle/SKILL.md` | enum 전이도, 복구 로직 |
| **장애 대응** | `consumer-dlq/SKILL.md`, `integrity-check/SKILL.md` | DLQ, 복구 절차 |
| **체크리스트** | `checklists/SKILL.md` | 신규 기능 추가 시 누락 방지 |
| **빠른 참조** | `class-index/SKILL.md`, `quick-reference/SKILL.md` | 클래스 위치, 자주 쓰는 컴포넌트 |
| **명세서** | `<message>-spec/SKILL.md`, `jsonb-schema/SKILL.md` | 메시지 포맷, 스키마 |
| **시나리오** | `scenarios/SKILL.md` | 자주 발생하는 케이스별 대응 |

새 프로젝트는 처음부터 15+개 다 만들 필요 없다. **3개부터 시작:**
1. `CODING_RULES.md`
2. `EXTERNAL_SYSTEMS.md`
3. `PATTERNS.md`

## 스킬 작성 원칙

### 1. 한 파일 = 한 주제
파일이 여러 주제를 다루면 BLOCKING 표에서 매핑하기 어렵다. 분리하라.

### 2. 코드 예시는 실측 기반
가짜 예시(`foo`, `bar`) 대신 실제 프로젝트 코드 인용. 변경 시 같이 업데이트하기 쉽다.

### 3. 안티패턴 명시
"X를 하지 마라" 만 적지 말고, **왜** 안 되는지 사례로 적어둔다. Claude가 엣지 케이스에서 판단할 수 있게.

```markdown
## ❌ 하지 말아야 할 것

### SSH로 운영 DB 접근
**금지 이유**: 운영 서버는 SSH 비활성. 무리하게 시도하면 보안 알림 발생.
**대신**: Docker 원격 API로 접근 (`DOCKER_HOST=tcp://...`)
```

### 4. 명령어는 복붙 가능 형태
`# user/password를 채우세요` 같은 placeholder는 OK. 하지만 명령어 구조 자체는 그대로 실행 가능해야 함.

### 5. 길이 제한 없음
스킬은 필요할 때만 로드되므로 길어도 OK. 다만 한 파일이 1000줄을 넘으면 분리 검토.

## CLAUDE.md와의 연결

스킬을 만들면 반드시 CLAUDE.md의 BLOCKING 표에 추가:

```markdown
| **외부 DB 작업** | [EXTERNAL_SYSTEMS.md](.claude/skills/EXTERNAL_SYSTEMS.md) | SSH 금지, Docker API |
```

표에 없으면 Claude가 그 스킬의 존재를 모른다.

## 유지보수

- **스킬은 자주 부패한다**: 코드/구조가 변하면 스킬도 같이 갱신해야 함
- **거짓 정보는 차라리 삭제**: 오래된 정보는 추측 근거가 되어 더 위험
- **PR마다 영향받는 스킬 갱신**: 새 패턴 도입 시 PATTERNS.md 같이 업데이트

## 템플릿

- [templates/skills/CODING_RULES.md.template](templates/skills/CODING_RULES.md.template)
- [templates/skills/EXTERNAL_SYSTEMS.md.template](templates/skills/EXTERNAL_SYSTEMS.md.template)
- [templates/skills/PATTERNS.md.template](templates/skills/PATTERNS.md.template)
