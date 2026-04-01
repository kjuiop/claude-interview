---
tags: [kafka, messaging, event-streaming, interview-questions]
related: [rabbitmq, distributed-systems]
---

# Kafka — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/kafka/concepts]] | 비교: [[topics/rabbitmq/questions]]

---

## 메시지 전달 보장

### Q. Kafka에서 메시지 손실과 중복 처리를 어떻게 방지하나요? at-least-once, at-most-once, exactly-once의 차이를 설명해주세요.

**세 가지 전달 보장 수준 — offset commit 시점 기준**

| 보장 수준 | offset commit 시점 | 특성 |
|---|---|---|
| at-most-once | 처리 **전** commit | 크래시 시 메시지 소실 가능, 중복 없음 |
| at-least-once | 처리 **후** commit | 크래시 시 재처리 → 중복 발생 가능 |
| exactly-once | 처리 + commit 원자적 | 손실·중복 없음, 구현 복잡 |

**프로듀서 측 메시지 손실 방지**
- 재시도 정책: retryable 예외(네트워크 오류)와 non-retryable 예외(파싱 오류) 분리
- 지수 백오프(exponential backoff): 재시도 간격을 2의 지수로 늘려 부하 분산
- **Idempotent Producer**: `enable.idempotence=true` + `acks=all` → 브로커에서 시퀀스 번호로 중복 제거
- **Kafka Transactions**: `transactional.id` 설정 → `beginTransaction() → send → commitTransaction()` 으로 여러 파티션/토픽에 원자적 쓰기

**프로듀서 측 정합성 보장 — Outbox 패턴**
- Kafka 직접 발행 대신 DB에 메시지 발행 데이터를 저장 (같은 트랜잭션)
- 트랜잭션 성공 시에만 메시지 발행 데이터 존재 → 정합성 보장
- DB → Kafka 전송: polling 방식 또는 CDC(Change Data Capture) 방식

**컨슈머 측 중복 처리 방지**
- **멱등키**: 도메인 비즈니스키 기반 unique 제약 → 중복 이벤트로 인한 데이터 중복 방어 (최후 방어선)
- **Inbox 패턴**: 수신 메시지를 DB에 저장 + 이벤트키 unique 설정으로 중복 처리 원천 차단
- **수동 ack**: 로직 성공 후에만 commit → at-least-once 보장

**모범 답변 구조:**
세 개 정의(offset commit 시점 기준) → at-least-once가 기본값인 이유 → Idempotent Producer + Kafka Transactions(Kafka 자체 exactly-once) → Outbox 패턴(프로듀서 정합성) → 멱등키 + Inbox 패턴(컨슈머 중복 방지) → 실무에선 at-least-once + 멱등성이 현실적 선택

**꼬리 질문 대비:**
- "Idempotent Producer의 한계는?" → PID는 메모리 기반, 재시작 시 새 PID → `transactional.id` 함께 사용해야 재시작 후에도 중복 방지
- "`transactional.id`만 설정하면 Exactly-Once?" → 아니요. `enable.idempotence=true` + 컨슈머 `isolation.level=read_committed`도 필수
- "`sendOffsetsToTransaction`이 왜 필요한가?" → 메시지와 offset을 같은 트랜잭션으로 묶어야 완전한 Exactly-Once. 없으면 메시지와 offset 불일치
- "`abortTransaction()` 후 메시지는?" → 토픽에 기록되지만 `read_committed` 컨슈머는 볼 수 없음 (marker로 표시됨)
- "Zombie Fencing이란?" → 같은 `transactional.id`로 새 Producer 시작 시 이전 zombie Producer의 write를 자동 차단. 분산 환경에서 중복 발행 방지.

---

## Idempotent Producer

### Q. Kafka Idempotent Producer는 무엇이고, 어떤 한계가 있나요?

**핵심 답변 구조:**
- 정의: 시퀀스 번호로 브로커에서 중복 메시지 제거
- 설정: `enable.idempotence=true` + `acks=all`
- 범위: 단일 프로듀서 세션 내 중복만 제거 (컨슈머 중복은 별개)
- 한계: PID는 메모리 기반 → **재시작 시 새 PID 발급 → 재시작 전후 구간 중복 가능**
- 해결: `transactional.id` 고정값 설정 → 재시작 후 동일 PID 복구 + Zombie Fencing

**꼬리 질문 대비:**
- "재시작 시 왜 중복이 생기나?" → PID 메모리 기반, 재시작 = 새 PID, 브로커는 이전 세션 기억 못함
- "transactional.id를 설정하면 뭐가 달라지나?" → 브로커가 고정 ID로 PID를 영속적으로 관리, Zombie Fencing 추가

---

## Kafka Transactions

### Q. Kafka Transactions의 동작 흐름과 `sendOffsetsToTransaction()`이 필요한 이유를 설명해주세요.

**핵심 답변 구조:**
- 정의: 여러 파티션/토픽에 걸친 원자적 쓰기 보장
- 흐름: `initTransactions() → beginTransaction() → send() → sendOffsetsToTransaction() → commitTransaction()`
- 컨슈머 설정: `isolation.level=read_committed` + `enable.auto.commit=false`

**`sendOffsetsToTransaction()` 이 필요한 이유:**
- Consume → Process → Produce 패턴에서 offset commit을 트랜잭션에 포함
- 없으면: produce 완료 → crash → offset 미commit → 재처리 → 중복 produce
- 있으면: produce + offset commit이 원자적 완료 → Exactly-Once 보장

**꼬리 질문 대비:**
- "`abortTransaction()` 후 메시지는?" → 토픽에 기록되나 `read_committed` 컨슈머는 marker로 인해 무시
- "transactional.id 없이 exactly-once 가능?" → 불가. 재시작 시 PID 재발급으로 멱등성 깨짐

**`sendOffsetsToTransaction` 없을 때 두 실패 케이스 정리** (2026-04-01 세션 — 자주 반대로 기억함):

| 케이스 | 결과 | 설명 |
|---|---|---|
| Produce 성공 + Offset commit **실패** | **중복 처리** (at-least-once) | offset 미올라감 → 같은 메시지 재소비 → 중복 produce |
| Offset commit 성공 + Produce **실패** | **메시지 유실** (at-most-once) | offset 올라감 → 메시지 재소비 안 함 → output topic에 없음 |

- `sendOffsetsToTransaction`은 두 작업을 하나의 트랜잭션으로 묶어 둘 다 커밋되거나 둘 다 롤백되게 함 → exactly-once 완성

---

## 메시지 순서 보장

### Q. Kafka에서 메시지 순서 보장 조건과 파티션 수 변경 시 발생하는 문제를 설명해주세요.

**핵심 답변 구조:**
- 조건: 동일 키 → 동일 파티션 → 파티션 내 순서 보장 (파티션 간 순서는 보장 안 됨)
- 파티션 수 변경 문제: 키 해시 재계산 → 기존 키가 다른 파티션으로 → 전체 순서 깨짐
- **리밸런싱 중 순서 깨짐**: 기존 파티션 미처리 메시지 + 새 파티션 신규 메시지 → 다른 컨슈머가 처리 → 전체 순서 불가
- 해결: 미처리 메시지 소진 후 파티션 수 변경 / 처음부터 충분한 파티션 수 설정

**설계 전략:**
- 완전한 순서 보장: 파티션 1개 + 컨슈머 1개 (처리량 희생)
- 처리량 필요 시: 파티션 수 = 최대 예상 컨슈머 수 × 2~3배로 초기 설정, 이후 컨슈머로 처리량 조절
- `max.in.flight.requests.per.connection=1`: 재시도 시 순서 역전 방지

**꼬리 질문 대비:**
- "파티션 수 변경 도중 미처리 메시지 순서는?" → 기존 P0 미처리 + 신규 P2 메시지 → 다른 컨슈머 처리 → 전체 순서 깨짐
- "순서 보장하면서 처리량 높이려면?" → 키 설계로 파티션 분산 + 파티션 수 초기에 충분히 설정

---

## 작성 예정

주요 주제:
- Kafka의 높은 처리량 비결 (순차 I/O, Zero-copy, 배치)
- 파티션 수 결정 기준
- Consumer lag 모니터링 및 대응
