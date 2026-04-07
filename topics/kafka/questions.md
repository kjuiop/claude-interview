---
tags: [kafka, messaging, event-streaming, interview-questions]
related: [rabbitmq, distributed-systems, aws]
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

## Kafka의 기본 구성 요소(Producer/Consumer/Broker/Topic/Partition)를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: Pull 방식, Partition, Offset, Consumer Group, Broker, Leader/Follower

**모범 답변 방향**:
- **Broker**: 메시지를 저장·전달하는 서버. 여러 Broker가 모여 Kafka Cluster 구성
- **Topic/Partition**: 메시지 분류 단위(Topic)를 물리적으로 나눈 단위(Partition). 파티션별로 순서가 있는 로그 파일. 여러 Broker에 분산 → 수평 확장
- **Producer**: 특정 Topic의 파티션 Leader에게 직접 메시지 전송. Partition Key 있으면 동일 키 → 동일 파티션 보장
- **Consumer**: **Pull 방식**으로 Broker에서 메시지를 가져옴. 읽은 위치(Offset)를 `__consumer_offsets` 토픽에 커밋
- **Consumer Group**: 같은 그룹 내 Consumer들은 파티션을 나눠 담당 → 수평 확장. 다른 그룹은 독립적으로 동일 토픽 소비 가능

**꼬리 질문 예시**:
- Kafka가 Push 방식이 아닌 Pull 방식을 선택한 이유는 무엇인가요?
- Consumer 수가 파티션 수보다 많으면 어떻게 되나요?

---

## Kafka Producer의 Batch Processing 원리와 처리량 최적화를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: linger.ms, batch.size, buffer.memory, compression.type, Sticky Partitioner

**모범 답변 방향**:
- 메시지를 즉시 전송하지 않고 **배치로 묶어서 전송** → 네트워크 왕복(RTT) 횟수 감소
- `linger.ms`: 배치 대기 시간. 기본 0(즉시 전송). 높일수록 처리량↑ / 지연↑
- `batch.size`: 배치 최대 크기(기본 16KB). 도달 즉시 전송
- `compression.type`: gzip/snappy/lz4 — 배치 단위 압축 → 네트워크 대역폭 감소
- **Sticky Partitioner**: 배치가 채워질 때까지 동일 파티션에 메시지 모음 → 배치 효율 향상

**처리량 vs 지연 트레이드오프**:
```
linger.ms=0  → 지연 최소, 처리량 낮음 (실시간 처리 우선)
linger.ms=10 → 지연 10ms 허용, 배치 크기↑ → 처리량 향상 (배치 처리 우선)
```

**꼬리 질문 예시**:
- `linger.ms`를 높이면 무조건 좋은 건가요? 어떤 상황에서는 낮춰야 하나요?
- `buffer.memory`가 가득 차면 Producer는 어떻게 동작하나요?

---

## Consumer Offset과 Commit 방식을 설명하고, 각 방식의 트레이드오프를 비교해주세요.

**난이도**: 중급

**핵심 키워드**: __consumer_offsets, 자동 커밋, 수동 커밋, commitSync, commitAsync, at-least-once

**모범 답변 방향**:
- **Offset**: 파티션 내 메시지의 고유 순번. Consumer가 어디까지 읽었는지 나타냄
- Offset은 `__consumer_offsets` 내부 토픽에 저장 → Consumer 재시작 시 이어서 소비

| 방식 | 동작 | 장점 | 단점 |
|---|---|---|---|
| **자동 커밋** (`enable.auto.commit=true`) | `auto.commit.interval.ms` 주기로 자동 커밋 | 간단 | 처리 중 크래시 시 재처리 안 됨 (메시지 유실 가능) |
| **수동 동기 커밋** (`commitSync()`) | 처리 후 명시적 커밋, 성공 대기 | 정확한 at-least-once 보장 | 커밋 응답 대기 → 처리량 낮음 |
| **수동 비동기 커밋** (`commitAsync()`) | 처리 후 비동기 커밋, 콜백 | 처리량 높음 | 실패 시 재시도 로직 필요 |

- **실무 권장**: `enable.auto.commit=false` + `commitSync()` (정확성 우선) 또는 배치 처리 후 `commitAsync()` + 종료 시 `commitSync()` 조합

**꼬리 질문 예시**:
- 자동 커밋 환경에서 메시지가 유실될 수 있는 시나리오를 설명해주세요.
- 파티션별로 다른 offset에서 커밋하려면 어떻게 하나요?

---

## Kafka Cluster의 Replica, ISR, min.insync.replicas 개념과 acks 설정의 관계를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Leader/Follower, ISR, min.insync.replicas, acks=all, 데이터 내구성

**모범 답변 방향**:
- **Leader Replica**: 모든 읽기/쓰기 처리. **Follower Replica**: Leader 복제 + 장애 대비 대기
- **ISR**: Leader와 동기화된 Replica 집합. `replica.lag.time.max.ms` 내 미복제 시 ISR 제외
- Leader 장애 → **ISR 내 Follower 중 새 Leader 선출** (ISR 밖 Follower는 대상 아님)

**acks 설정별 동작**:
```
acks=0  → 응답 안 기다림. 최고 속도, 유실 가능
acks=1  → Leader만 기록 확인. Leader 장애 시 유실 가능
acks=all → ISR 전체 기록 확인. 가장 안전, 지연 증가
```

**min.insync.replicas 조합 (replication.factor=3 기준)**:
```
acks=all + min.insync.replicas=1 → ISR 1개만 있어도 커밋 (가용성 우선)
acks=all + min.insync.replicas=2 → ISR 2개 필요 (실무 권장, 균형)
acks=all + min.insync.replicas=3 → 브로커 1개만 다운돼도 쓰기 불가 (내구성 최대)
```

**꼬리 질문 예시**:
- Follower가 ISR에서 제외되는 조건은 무엇인가요?
- `acks=1`로 설정했는데 Leader가 Follower에 복제하기 전에 죽으면 어떻게 되나요?
- `min.insync.replicas=2`인데 ISR이 1개만 남으면 Producer는 어떻게 되나요?

---

## Consumer Group과 파티션 할당 전략, 병렬 처리 확장 방법을 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Consumer Group, 파티션 할당, RangeAssignor, CooperativeStickyAssignor, 병렬도

**모범 답변 방향**:
- **같은 그룹**: 파티션을 나눠 담당 → 중복 없이 병렬 처리. 파티션 수가 병렬도 상한
- **다른 그룹**: 독립적으로 동일 토픽 소비 → 서로 다른 서비스(분석 vs 실시간 처리)가 같은 이벤트 소비 가능

**파티션 할당 전략**:
| 전략 | 특징 |
|---|---|
| RangeAssignor | 토픽별 파티션 범위 나눔. 토픽 많으면 불균형 |
| RoundRobinAssignor | 전체 파티션 균등 배분 |
| StickyAssignor | 리밸런싱 시 기존 할당 최대한 유지 |
| **CooperativeStickyAssignor** | Incremental 리밸런싱 → Stop-The-World 없음. 신규 배포 권장 |

**병렬 처리 확장**:
1. 파티션 수↑ + Consumer 수↑ → 파티션당 1 Consumer 기준 선형 확장
2. Consumer 내 스레드 풀로 내부 병렬화 (순서 보장 깨짐 주의)
3. `max.poll.records` 늘려 한 번에 더 많이 가져와 처리

**꼬리 질문 예시**:
- Consumer 수를 파티션 수보다 많이 늘리면 어떻게 되나요?
- 파티션 수를 늘리면 무조건 좋은가요? 파티션 증가의 단점은?

## Consumer Rebalancing으로 인한 처리 중단을 최소화하는 설계

**난이도**: 심화

**핵심 키워드**: CooperativeStickyAssignor, Static Membership, session.timeout.ms, max.poll.interval.ms, KRaft, Rolling Update

**모범 답변 방향**:
- **Eager Rebalancing 문제**: 리밸런싱 시작 시 모든 Consumer가 파티션을 반납(revoke) → Stop-The-World 발생
- **Cooperative Incremental Rebalancing**: `CooperativeStickyAssignor` 사용. 변경이 필요한 파티션만 단계적으로 재배정. 나머지 Consumer는 계속 처리 가능. 30초 중단 → 수 초 이내로 단축
- **Static Group Membership** (`group.instance.id`): Consumer에 고정 ID 부여 → 재시작 시 동일 ID로 재합류 → 리밸런싱 없이 파티션 재획득. Rolling Update와 찰떡 조합
- **타임아웃 설정 관계**: `session.timeout.ms`는 브로커가 Consumer 장애를 감지하는 시간(기본 45초). `max.poll.interval.ms`는 `poll()` 호출 간격 최대치(기본 5분). Static Membership에서 `session.timeout.ms` 동안 재접속하면 리밸런싱 없이 복구 가능
- **KRaft 환경 변화**: ZooKeeper 제거로 컨트롤러가 KRaft 기반으로 동작. 파티션 메타데이터 관리가 더 빠르고 안정적. 리밸런싱 속도 자체도 개선됨. ZooKeeper 의존성 제거로 운영 복잡도 감소

**꼬리 질문 예시**:
- `CooperativeStickyAssignor`와 `StickyAssignor`의 차이점은 무엇인가요?
- Static Membership에서 `session.timeout.ms`를 너무 길게 잡으면 어떤 문제가 생기나요?
- KRaft 모드에서 Leader Election 방식은 ZooKeeper 기반과 어떻게 다른가요?

---
