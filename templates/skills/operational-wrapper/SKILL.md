---
name: <command-name>
description: <한 줄 설명 + 트리거 문구 예시. 예: "DLQ 현황 조회 및 원인 분류. DLQ에 쌓인 메시지 수, 원인 분포, redrive 가이드. '~ 확인해줘' 등 트리거.">
argument-hint: [optional-arg?]
disable-model-invocation: true
context: fork
agent: <subagent-name>
---

# <command-name> 운영 매크로

$ARGUMENTS (인자 의미 설명) 에 대해 <작업 설명>.

## 수행 절차

1. **1단계**
   - 구체적 수행 내용

2. **2단계**
   - 수행 내용

3. **3단계**
   - 수행 내용

4. **출력 정리**
   - 형식 (표 / 리스트 / 코드 블록)

## 필수 참조

- [../<related-skill>/SKILL.md](../<related-skill>/SKILL.md) — 관련 도메인 skill

## 제약

- 코드 수정 금지 (진단만)
- 추측 금지 (실측 기반)
- 못 찾으면 범위 확장 알림
