---
type: harness-entropy
updated: 2026-03-27
---

# 엔트로피 추적 (Entropy Management)

> Claude가 세션 중 발견한 불일치, 빈 파일, broken link, 취약 영역을 기록한다.
> `/퇴근` 실행 시 이 목록을 기준으로 정리한다.

---

## 즉시 처리 필요 (Critical)

| 항목 | 파일 | 문제 | 조치 |
|---|---|---|---|
| - | - | - | - |

---

## 채워야 할 지식 Gaps

> topics 파일이 있지만 내용이 비어 있는 것들

| 기술 | 파일 | 우선순위 | 이유 |
|---|---|---|---|
| kafka | `topics/kafka/concepts.md` | 🔴 높음 | 화이트큐브 공고 필수 스택 |
| kafka | `topics/kafka/questions.md` | 🔴 높음 | 면접 준비 필요 |
| rabbitmq | `topics/rabbitmq/concepts.md` | 🟡 중간 | 공고 언급 |
| rabbitmq | `topics/rabbitmq/questions.md` | 🟡 중간 | 공고 언급 |
| java | `topics/java/concepts.md` | 🟡 중간 | 공고 요구사항 |
| kotlin | `topics/kotlin/concepts.md` | 🟡 중간 | 공고 요구사항 |
| elasticsearch | `topics/elasticsearch/concepts.md` | 🟡 중간 | 공고 언급 |

---

## Broken / Orphan Links

> 링크 대상이 없거나 home.md에서 연결되지 않은 파일들

| 파일 | 문제 링크 | 상태 |
|---|---|---|
| - | - | - |

---

## Harness 신호 기록

> Claude가 답하지 못했거나 잘못 답변한 경우 — 다음 보강 대상

| 날짜 | 세션 | 문제 상황 | 필요한 보강 |
|---|---|---|---|
| - | - | - | - |

---

## 완료된 항목

| 날짜 | 항목 | 처리 내용 |
|---|---|---|
| 2026-03-27 | java-kotlin 폴더 분리 | java/, kotlin/ 으로 분리 완료 |
| 2026-03-27 | kafka-rabbitmq 폴더 분리 | kafka/, rabbitmq/ 으로 분리 완료 |
| 2026-03-27 | golang concepts.md 분리 | 주제별 파일 10개로 분리 완료 |
