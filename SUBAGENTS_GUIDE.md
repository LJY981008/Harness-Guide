# 서브에이전트(Sub-agents) 작성 가이드

## 서브에이전트란?

`.claude/agents/*.md`. 메인 Claude 대화에서 `Agent` 툴로 호출되는 **별 컨텍스트의 작업자**. 메인 컨텍스트를 더럽히지 않고 복잡한 분석을 위임할 때 쓴다.

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

## 안티패턴

- ❌ **너무 일반적인 에이전트**: "코드 리뷰 에이전트" → 매칭 모호. 차라리 `tx-integrity-analyzer`처럼 특화.
- ❌ **에이전트 안에서 또 에이전트 호출**: 가능하긴 하나 컨텍스트 폭발. 한 단계만.
- ❌ **메인이 할 일을 굳이 에이전트로**: 단순 grep, 단일 파일 수정 등은 메인에서.
- ❌ **출력 포맷 미정의**: 에이전트 결과가 자유서술이면 메인이 다시 정리해야 함. 표/체크리스트 강제.

## 실행 예시

메인 Claude가:
```
Agent(
  description="Outbox 취약점 분석",
  subagent_type="general-purpose",  # 또는 커스텀 등록 시 그 이름
  prompt="..."
)
```
→ 서브에이전트가 자기 컨텍스트에서 작업 → 결과 텍스트만 메인에 반환.

## 병렬 실행

여러 에이전트를 동시에 돌릴 수 있다 (메인이 한 메시지에 여러 Agent 툴 호출). 독립적인 분석들을 병렬화하면 시간 절약.

## 템플릿

[templates/agents/vulnerability-analyzer.md.template](templates/agents/vulnerability-analyzer.md.template)
