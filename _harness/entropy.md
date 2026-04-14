---
type: harness-entropy
updated: 2026-04-14
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
| distributed-systems | `topics/distributed-systems/concepts.md` | 🔴 높음 | CAP 정리 오늘 세션 전혀 모름 → 개념 추가 완료, 복습 필요 |
| kotlin | `topics/kotlin/concepts.md` | 🔴 높음 | suspend fun Continuation, Dispatcher 타입 오개념. 스텁만 있음 |
| redis | `topics/redis/concepts.md` | 🟡 중간 | Redis 장애 fallback(Sentinel/Cluster, degraded mode) 면접 세션에서 누락 확인 — 정리 필요 |
| rabbitmq | `topics/rabbitmq/concepts.md` | 🟡 중간 | 공고 언급 |
| rabbitmq | `topics/rabbitmq/questions.md` | 🟡 중간 | 공고 언급 |
| java | `topics/java/concepts.md` | 🟡 중간 | 공고 요구사항 |
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
| 2026-04-01 | 4회차 | CAP 정리 전혀 모름 | distributed-systems/concepts.md CAP 개념 작성 + 복습 |
| 2026-04-01 | 3회차 | Kotlin Coroutine "싱글스레드 기반" 오개념 | kotlin/concepts.md Dispatcher 타입 설명 추가 필요 |
| 2026-04-01 | 1회차 | TLS Handshake 인증서 전송 방향 역전 | 2회차에서 교정 완료 |
| 2026-04-01 | 1,2회차 | Kafka 두 실패 케이스(중복 vs 유실) 정반대 | 2회차에서 교정 완료 |
| 2026-04-14 | 2회차 | Redis 장애 fallback 미언급 | redis/concepts.md Sentinel/Cluster, degraded mode 정리 필요 |
| 2026-04-14 | 2회차 | ZooKeeper Watch 1회성 재등록 미언급 | zookeeper/questions.md에 재등록 패턴 보강 — 이미 기록됨 |

---

## 완료된 항목

| 날짜 | 항목 | 처리 내용 |
|---|---|---|
| 2026-03-27 | java-kotlin 폴더 분리 | java/, kotlin/ 으로 분리 완료 |
| 2026-03-27 | kafka-rabbitmq 폴더 분리 | kafka/, rabbitmq/ 으로 분리 완료 |
| 2026-03-27 | golang concepts.md 분리 | 주제별 파일 10개로 분리 완료 |
