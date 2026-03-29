---
tags: [kafka, messaging, event-streaming, backend]
related: [rabbitmq, zookeeper, distributed-systems]
---

# Kafka — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/kafka/questions]] | 비교: [[topics/rabbitmq/concepts]]

---

## 전달 보장 수준 (Delivery Semantics)

### at-most-once
- offset commit을 메시지 처리 **전**에 수행
- 처리 중 크래시 시 메시지 소실 가능
- 중복은 없지만 손실 허용

### at-least-once
- offset commit을 메시지 처리 **후**에 수행
- 처리 후 크래시 시 동일 메시지 재처리 → 중복 발생 가능
- 실무 기본값 — 멱등성으로 보완

### exactly-once
- Kafka 자체 메커니즘:
  - **Idempotent Producer**: `enable.idempotence=true` + `acks=all` → 시퀀스 번호로 브로커에서 중복 제거
  - **Kafka Transactions**: `transactional.id` 설정 → 여러 파티션/토픽에 원자적 쓰기
- 애플리케이션 레벨: Outbox 패턴(프로듀서) + Inbox 패턴/멱등키(컨슈머) 조합

---

## 실무 패턴

### Outbox 패턴 (프로듀서 정합성)
- Kafka 직접 발행 대신 DB에 아웃박스 테이블 저장 (같은 트랜잭션)
- DB → Kafka: polling 방식 또는 Debezium 등 CDC 방식
- 효과: 비즈니스 로직 실패 시 메시지 발행 자체를 막음

### Inbox 패턴 (컨슈머 멱등성)
- 수신 메시지를 인박스 테이블에 저장 + 이벤트키 unique 제약
- 중복 메시지 수신 시 DB 레벨에서 차단
- 발행 시 이벤트 키 생성 → 추적 가능성 확보

---

## Idempotent Producer (멱등 프로듀서)

### 정의
Producer가 같은 메시지를 여러 번 보내도 브로커에서 자동으로 중복을 제거하는 기능.

### 동작 원리
1. **PID 할당**: Producer 인스턴스 생성 시 Broker로부터 고유 PID(Producer ID) 발급
2. **Sequence Number 태깅**: 메시지마다 파티션별 단조증가 시퀀스 번호 추가
3. **브로커 중복 제거**: 브로커가 시퀀스 번호로 중복 메시지 감지 및 거부
   - `seq == max_seq + 1` → 수락
   - `seq <= max_seq` → 거부 (중복)
   - `seq > max_seq + 1` → out-of-sequence 오류

```properties
# Producer 설정
enable.idempotence=true
# 자동으로 적용됨: acks=all, retries=MAX_INT, max.in.flight.requests.per.connection≤5
```

### 주의할 점 / 흔한 오개념
- **PID는 메모리 기반**: Producer 재시작 시 새 PID 발급 → 재시작 후 중복 가능 → `transactional.id` 함께 사용 필요
- **파티션별 독립**: 시퀀스 번호는 파티션마다 별도 관리
- **Kafka 3.0+**: `enable.idempotence=true`가 기본값

---

## Kafka Transactions (트랜잭션)

### 정의
여러 파티션/토픽에 걸친 원자적 쓰기를 보장. Producer 재시작 후에도 zombie fencing으로 중복 방지.

### 동작 원리

**transactional.id 역할:**
- 동일 `transactional.id`로 새 Producer가 시작되면, 이전 Producer(zombie)의 write를 자동 차단 (Zombie Fencing)
- Producer 재시작 후에도 같은 PID 유지 → 장기적 멱등성 보장

**트랜잭션 흐름:**
```
initTransactions()       → Broker에서 PID + epoch 발급
beginTransaction()       → 트랜잭션 시작
send()                   → PID + seq + epoch 포함 메시지 전송
sendOffsetsToTransaction() → Consumer offset을 트랜잭션에 포함 (Exactly-Once용)
commitTransaction()      → 원자적 커밋
  또는
abortTransaction()       → 전체 롤백 (메시지는 저장되나 read_committed consumer에게 안 보임)
```

```properties
# Producer 설정
enable.idempotence=true          # 필수 선행 조건
transactional.id=my-app-id       # 고유값, 재시작해도 동일해야 함

# Consumer 설정 (Exactly-Once 완성)
isolation.level=read_committed
enable.auto.commit=false
```

### 실무 사용 시나리오
- 주문 + 재고 차감을 동일 트랜잭션으로 묶어야 할 때
- Consume → Transform → Produce 파이프라인의 Exactly-Once 보장

### 주의할 점
- `sendOffsetsToTransaction()` 미호출 시 메시지와 offset 불일치 → Exactly-Once 보장 실패
- `abortTransaction()` 후에도 메시지는 토픽에 기록됨 (marker만 표시), `read_committed` consumer만 무시

---

## 작성 예정

- Kafka 아키텍처 (Broker, Topic, Partition, Replica)
- Consumer Group & Rebalancing
- Kafka vs RabbitMQ 차이
- KRaft 모드 (ZooKeeper 의존성 제거)
- Dead Letter Queue
