---
type: harness-entropy
updated: 2026-04-28
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
| 2026-04-15 | 1~3회차(채널톡) | 경험→질문 연결 훈련 부족 — 5문 연속 "잘 모르겠습니다" | 대표 경험 4가지(MultiCDN/S3파이프라인/ZooKeeper/CMAF) STAR 암기 노트 system-design/concepts.md에 추가 필요 |
| 2026-04-15 | 1~3회차(채널톡) | 비즈니스 임팩트 마무리 습관 없음 | "그래서 고객에게 어떤 가치?" 패턴을 모든 답변 마무리에 붙이는 연습 필요 |
| 2026-04-20 | 3회차 | Istio VirtualService/DestinationRule Canary 패턴 완전 미지(0/10) | kubernetes/questions.md YAML 패턴 암기. subset+weight+headers 조건 코드 수준 |
| 2026-04-20 | 3회차 | LLM Hallucination Temperature·CoT·Re-ranking 미언급 | ai-stt/questions.md Hallucination 섹션 추가 완료. 이력서(STT→RAG) 연결 훈련 필요 |
| 2026-04-20 | 4회차 | goroutine leak close(ch) 패턴 미언급 — "특정값 보내 nil 변경"으로 오답 | golang/goroutine.md close(ch) vs nil channel 차이 패턴 추가 완료. 암기 필요 |
| 2026-04-28 | 3회차 | WebSocket/STOMP — @SendTo를 1:1로 오해. @SendToUser, SimpMessagingTemplate 역할 미설명(5/10) | java/questions.md WebSocket 섹션 모범 답변 추가 완료. 어노테이션 역할 3종 구분 반복 필요 |
| 2026-04-28 | 3회차 | K8s volumeClaimTemplates — Pod별 독립 PVC 자동생성 동작 꼬리 "모르겠습니다"(5/10) | kubernetes/questions.md StatefulSet 섹션 보완. data-kafka-0/data-kafka-1 패턴 암기 필요 |
| 2026-04-28 | 4회차 | FastAPI Depends() — 요청 스코프 격리 메커니즘 꼬리 "모르겠습니다"(5/10) | "매 요청마다 새 인스턴스 생성 → 요청 스코프" 표현 암기 필요 |
| 2026-04-28 | 1회차 | PostgreSQL XID Wraparound — autovacuum 실패의 치명적 위험 미언급(6/10) | 32비트 XID 한계, freeze 동작, 쓰기 전면 차단 → 모범 답변에 포함됨 |

---

## 완료된 항목

| 날짜 | 항목 | 처리 내용 |
|---|---|---|
| 2026-03-27 | java-kotlin 폴더 분리 | java/, kotlin/ 으로 분리 완료 |
| 2026-03-27 | kafka-rabbitmq 폴더 분리 | kafka/, rabbitmq/ 으로 분리 완료 |
| 2026-03-27 | golang concepts.md 분리 | 주제별 파일 10개로 분리 완료 |
