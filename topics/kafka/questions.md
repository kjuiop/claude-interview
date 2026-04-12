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

**모범 답변:**
Kafka의 전달 보장 수준은 offset commit 시점에 따라 세 가지로 나뉩니다. at-most-once는 처리 전에 commit하는 방식으로 메시지 소실 가능성이 있지만 중복은 없고, at-least-once는 처리 후에 commit하기 때문에 크래시 발생 시 재처리로 인한 중복이 생길 수 있습니다. exactly-once는 처리와 commit을 원자적으로 묶는 방식으로 손실과 중복 모두 없지만 구현이 복잡합니다. 실무에서는 at-least-once를 기본으로 선택하고, 컨슈머 측에서 멱등성을 보장해 중복을 방어하는 방식이 현실적입니다. 프로듀서에서는 `enable.idempotence=true`와 `acks=all`을 설정한 Idempotent Producer로 단일 세션 내 중복을 제거하고, DB 정합성이 중요한 경우에는 Outbox 패턴을 사용합니다. Outbox 패턴은 Kafka에 직접 발행하는 대신 DB 트랜잭션 안에 메시지 발행 데이터를 함께 저장하고, 이후 CDC나 polling으로 Kafka에 전달하는 방식입니다. 컨슈머 측에서는 도메인 비즈니스키 기반 멱등키로 중복 이벤트를 최후 방어하거나, Inbox 패턴으로 수신 메시지를 DB에 저장하면서 이벤트키 unique 제약으로 중복 처리를 원천 차단합니다.

**꼬리 질문 대비:**
- "Idempotent Producer의 한계는?" → PID는 메모리 기반, 재시작 시 새 PID → `transactional.id` 함께 사용해야 재시작 후에도 중복 방지
- "`transactional.id`만 설정하면 Exactly-Once?" → 아니요. `enable.idempotence=true` + 컨슈머 `isolation.level=read_committed`도 필수
- "`sendOffsetsToTransaction`이 왜 필요한가?" → 메시지와 offset을 같은 트랜잭션으로 묶어야 완전한 Exactly-Once. 없으면 메시지와 offset 불일치
- "`abortTransaction()` 후 메시지는?" → 토픽에 기록되지만 `read_committed` 컨슈머는 볼 수 없음 (marker로 표시됨)
- "Zombie Fencing이란?" → 같은 `transactional.id`로 새 Producer 시작 시 이전 zombie Producer의 write를 자동 차단. 분산 환경에서 중복 발행 방지.

---

## acks 설정 및 내구성 보장

### Q. Kafka Producer의 acks 설정 0/1/all 차이와 min.insync.replicas와의 관계를 설명해주세요.

| acks 값 | 동작 | 트레이드오프 |
|---|---|---|
| `0` | 브로커 응답 기다리지 않음 | 처리량 최대, 유실 가능성 최대 |
| `1` | **리더 브로커**만 ack | 팔로워 복제 전 리더 장애 시 유실 가능 |
| `all(-1)` | ISR(In-Sync Replica) 전체 ack | 가장 안전, 처리량 낮음 |

**min.insync.replicas와의 관계:**
- `acks=all`은 "ISR 전체"가 아니라 **min.insync.replicas 이상**의 ISR이 ack하면 됨
- 예: replication factor=3, `min.insync.replicas=2` → 3개 중 최소 **2개** ack 필요
- ISR이 min.insync.replicas 미만으로 떨어지면 `NotEnoughReplicasException` 발생 (쓰기 거부)

**PID 재시작 주체 — 면접 자주 틀리는 포인트:**
- PID는 **Producer(클라이언트) 재시작** 시 새로 발급됨
- 브로커가 재시작해도 PID/Sequence Number는 브로커에 저장 → 문제없음
- `transactional.id` 설정 시 Producer 재시작 후에도 동일 PID 복구

**면접 세션 피드백 (2026-04-12 3회차)**:
- acks 3단계 방향, min.insync.replicas "최소" 의미 정확히 파악
- PID 재시작 주체 오류: "브로커 재시작"이라고 했으나 → **Producer 재시작**이 맞음
- acks=1 트레이드오프: "1개라도 ack"보다 "리더만 ack → 팔로워 복제 전 장애 시 유실" 명시 필요

---

## Idempotent Producer

### Q. Kafka Idempotent Producer는 무엇이고, 어떤 한계가 있나요?

**모범 답변:**
Idempotent Producer는 `enable.idempotence=true`와 `acks=all` 설정을 통해 브로커에서 시퀀스 번호로 중복 메시지를 제거하는 기능입니다. 프로듀서가 메시지를 전송할 때마다 단조 증가하는 시퀀스 번호를 함께 보내고, 브로커는 이미 기록된 번호의 메시지가 재전송되면 무시합니다. 이 방식으로 네트워크 오류나 재시도로 인한 단일 프로듀서 세션 내 중복을 방지할 수 있습니다. 다만 중요한 한계가 있는데, PID(Producer ID)가 메모리 기반이라는 점입니다. 프로듀서가 재시작하면 새로운 PID가 발급되고 브로커는 이전 세션을 기억하지 못하기 때문에, 재시작 전후 구간에서 중복이 발생할 수 있습니다. 이 한계를 해결하려면 `transactional.id`에 고정값을 설정해야 합니다. `transactional.id`를 설정하면 브로커가 해당 ID로 PID를 영속적으로 관리하기 때문에 재시작 후에도 동일 PID를 복구하고, Zombie Fencing으로 이전 인스턴스의 중복 발행도 자동 차단됩니다.

**꼬리 질문 대비:**
- "재시작 시 왜 중복이 생기나?" → PID 메모리 기반, 재시작 = 새 PID, 브로커는 이전 세션 기억 못함
- "transactional.id를 설정하면 뭐가 달라지나?" → 브로커가 고정 ID로 PID를 영속적으로 관리, Zombie Fencing 추가

---

## Kafka Transactions

### Q. Kafka Transactions의 동작 흐름과 `sendOffsetsToTransaction()`이 필요한 이유를 설명해주세요.

**모범 답변:**
Kafka Transactions는 여러 파티션과 토픽에 걸친 쓰기를 원자적으로 보장하는 기능입니다. 동작 흐름은 `initTransactions()`로 트랜잭션 코디네이터에 등록한 뒤, `beginTransaction()`으로 트랜잭션을 시작하고, `send()`로 메시지를 전송하고, `sendOffsetsToTransaction()`으로 offset commit을 트랜잭션에 포함시킨 뒤, 마지막으로 `commitTransaction()`으로 전체를 원자적으로 완료합니다. 컨슈머 측에서는 `isolation.level=read_committed`와 `enable.auto.commit=false`를 함께 설정해야 커밋된 메시지만 읽을 수 있습니다. `sendOffsetsToTransaction()`이 필요한 이유는 Consume-Process-Produce 패턴에서 exactly-once를 완성하기 위해서입니다. 이 메서드 없이 produce만 트랜잭션에 포함할 경우, produce가 성공한 뒤 크래시가 나면 offset이 커밋되지 않아 같은 메시지를 다시 소비하고 중복 produce가 발생합니다. 반대로 offset commit이 먼저 되고 produce가 실패하면 해당 메시지는 output 토픽에 존재하지 않는 유실 상태가 됩니다. `sendOffsetsToTransaction()`은 produce와 offset commit을 하나의 트랜잭션으로 묶어 둘 다 커밋되거나 둘 다 롤백되도록 보장해 exactly-once를 완성합니다.

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

**모범 답변:**
Kafka에서 순서 보장은 파티션 단위로만 이뤄집니다. 동일한 키를 가진 메시지는 동일한 파티션으로 라우팅되고, 파티션 내부에서는 append-only 로그 구조로 메시지 순서가 보장됩니다. 파티션 간 순서는 보장되지 않습니다. 파티션 수를 변경하면 키 해시 계산 결과가 달라져 기존 키가 다른 파티션으로 배정될 수 있고, 리밸런싱 도중에는 기존 파티션의 미처리 메시지와 새 파티션의 신규 메시지를 서로 다른 컨슈머가 처리하게 돼 전체 순서가 깨집니다. 때문에 파티션 수 변경은 미처리 메시지를 모두 소진한 뒤 진행해야 하고, 이상적으로는 처음부터 충분한 파티션 수를 설정해 이후에 변경하지 않는 것이 좋습니다. 완전한 순서 보장이 필요하다면 파티션 1개와 컨슈머 1개 조합을 사용해야 하지만 처리량을 희생하게 됩니다. 처리량이 중요한 경우에는 파티션 수를 최대 예상 컨슈머 수의 2~3배로 초기에 설정하고 컨슈머 수로 처리량을 조절하는 방식이 현실적입니다. 재시도 시 순서 역전을 방지하려면 `max.in.flight.requests.per.connection=1`도 함께 설정해야 합니다.

**꼬리 질문 대비:**
- "파티션 수 변경 도중 미처리 메시지 순서는?" → 기존 P0 미처리 + 신규 P2 메시지 → 다른 컨슈머 처리 → 전체 순서 깨짐
- "순서 보장하면서 처리량 높이려면?" → 키 설계로 파티션 분산 + 파티션 수 초기에 충분히 설정

---

## Kafka의 기본 구성 요소(Producer/Consumer/Broker/Topic/Partition)를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: Pull 방식, Partition, Offset, Consumer Group, Broker, Leader/Follower

**모범 답변:**
Kafka의 핵심 구성 요소는 크게 Broker, Topic과 Partition, Producer, Consumer, Consumer Group으로 나뉩니다. Broker는 메시지를 저장하고 전달하는 서버이고, 여러 Broker가 모여 Kafka Cluster를 구성합니다. Topic은 메시지를 분류하는 논리적 단위이고, 이를 물리적으로 나눈 단위가 Partition입니다. Partition은 순서가 있는 로그 파일 구조로 여러 Broker에 분산돼 수평 확장을 지원합니다. Producer는 특정 Topic의 파티션 Leader에게 직접 메시지를 전송하며, Partition Key가 있으면 동일 키가 항상 동일 파티션으로 라우팅됩니다. Consumer는 Push가 아닌 Pull 방식으로 Broker에서 메시지를 가져오는데, 이 방식 덕분에 Consumer가 자신의 처리 속도에 맞춰 메시지를 소비할 수 있습니다. 읽은 위치는 Offset이라는 고유 순번으로 관리되며, `__consumer_offsets` 내부 토픽에 커밋해 재시작 시에도 이어서 소비할 수 있습니다. Consumer Group은 같은 그룹 내 Consumer들이 파티션을 나눠 담당해 중복 없이 병렬 처리하고, 다른 그룹은 독립적으로 동일 토픽을 소비할 수 있어 분석 서비스와 실시간 처리 서비스가 같은 이벤트를 각자 소비하는 패턴이 가능합니다.

**꼬리 질문 예시**:
- Kafka가 Push 방식이 아닌 Pull 방식을 선택한 이유는 무엇인가요?
- Consumer 수가 파티션 수보다 많으면 어떻게 되나요?

---

## Kafka Producer의 Batch Processing 원리와 처리량 최적화를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: linger.ms, batch.size, buffer.memory, compression.type, Sticky Partitioner

**모범 답변:**
Kafka Producer는 메시지를 즉시 전송하지 않고 배치로 묶어서 전송합니다. 이 방식의 핵심 이점은 네트워크 왕복(RTT) 횟수를 줄이는 것입니다. 개별 메시지를 매번 전송하면 메시지 수만큼 왕복이 발생하지만, 배치로 묶으면 훨씬 적은 횟수로 같은 양의 메시지를 전달할 수 있습니다. `linger.ms`는 배치를 전송하기 전 대기하는 시간으로, 기본값은 0이라 즉시 전송됩니다. 이 값을 높이면 배치에 더 많은 메시지가 모여 처리량이 올라가지만, 그만큼 지연이 늘어납니다. `batch.size`는 배치의 최대 크기로 기본값은 16KB이며, 이 크기에 도달하면 즉시 전송합니다. `compression.type`을 gzip, snappy, lz4 등으로 설정하면 배치 단위로 압축해 네트워크 대역폭을 절감합니다. Sticky Partitioner는 배치가 채워질 때까지 동일 파티션에 메시지를 모아 배치 효율을 높이는 전략입니다. 결국 처리량과 지연은 트레이드오프 관계로, `linger.ms=0`이면 지연이 최소화되지만 처리량이 낮고, `linger.ms=10` 정도로 올리면 10ms의 지연을 허용하는 대신 배치 크기가 늘어나 처리량이 향상됩니다. 실시간 처리가 중요한 서비스는 낮게, 배치성 대용량 처리라면 높게 설정하는 것이 적합합니다.

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

**모범 답변:**
Offset은 파티션 내 메시지의 고유 순번으로, Consumer가 어디까지 읽었는지를 나타내는 위치 정보입니다. 이 Offset은 `__consumer_offsets` 내부 토픽에 저장되어 Consumer가 재시작해도 중단된 위치에서 이어서 소비할 수 있습니다. Commit 방식은 자동 커밋과 수동 커밋으로 나뉩니다. 자동 커밋은 `enable.auto.commit=true`로 설정하면 `auto.commit.interval.ms` 주기마다 자동으로 커밋되어 사용이 간단하지만, 처리 도중 크래시가 나면 커밋된 offset 이후 메시지가 재처리되지 않아 유실될 수 있습니다. 수동 동기 커밋인 `commitSync()`는 처리 후 명시적으로 커밋을 호출하고 성공 응답을 기다리기 때문에 정확한 at-least-once를 보장하지만, 커밋 응답 대기로 처리량이 낮아집니다. 수동 비동기 커밋인 `commitAsync()`는 처리 후 비동기로 커밋하기 때문에 처리량이 높지만, 실패 시 재시도 로직을 별도로 구현해야 합니다. 실무에서는 `enable.auto.commit=false`에 `commitSync()`를 조합하는 정확성 우선 방식이나, 배치 처리 후 `commitAsync()`를 사용하고 종료 시에만 `commitSync()`로 마무리하는 방식이 권장됩니다.

| 방식 | 동작 | 장점 | 단점 |
|---|---|---|---|
| **자동 커밋** (`enable.auto.commit=true`) | `auto.commit.interval.ms` 주기로 자동 커밋 | 간단 | 처리 중 크래시 시 재처리 안 됨 (메시지 유실 가능) |
| **수동 동기 커밋** (`commitSync()`) | 처리 후 명시적 커밋, 성공 대기 | 정확한 at-least-once 보장 | 커밋 응답 대기 → 처리량 낮음 |
| **수동 비동기 커밋** (`commitAsync()`) | 처리 후 비동기 커밋, 콜백 | 처리량 높음 | 실패 시 재시도 로직 필요 |

**꼬리 질문 예시**:
- 자동 커밋 환경에서 메시지가 유실될 수 있는 시나리오를 설명해주세요.
- 파티션별로 다른 offset에서 커밋하려면 어떻게 하나요?

---

## Kafka Cluster의 Replica, ISR, min.insync.replicas 개념과 acks 설정의 관계를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Leader/Follower, ISR, min.insync.replicas, acks=all, 데이터 내구성

**모범 답변:**
Kafka의 데이터 내구성은 Replica 구조를 기반으로 합니다. 파티션별로 Leader Replica와 Follower Replica가 존재하는데, 모든 읽기와 쓰기는 Leader를 통해 이뤄지고 Follower는 Leader를 복제하며 장애 대비 대기 상태를 유지합니다. ISR은 Leader와 실시간으로 동기화된 Replica 집합을 의미합니다. `replica.lag.time.max.ms` 시간 내에 복제하지 못한 Follower는 ISR에서 제외되고, Leader 장애 시 ISR 내 Follower 중에서만 새 Leader가 선출됩니다. acks 설정은 이 내구성 수준을 결정합니다. `acks=0`은 응답을 기다리지 않아 속도가 가장 빠르지만 유실 가능성이 있고, `acks=1`은 Leader 기록만 확인하지만 Leader 장애 시 Follower 복제 전이면 유실이 발생합니다. `acks=all`은 ISR 전체 기록을 확인하기 때문에 가장 안전하지만 지연이 늘어납니다. `min.insync.replicas`는 `acks=all`과 조합해 최소 ISR 수를 지정합니다. replication factor가 3일 때 `min.insync.replicas=2`로 설정하면 ISR이 2개 이상일 때만 쓰기를 허용해 가용성과 내구성의 균형을 맞출 수 있어 실무에서 가장 많이 사용하는 조합입니다.

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

**모범 답변:**
Consumer Group은 Kafka에서 수평 확장과 메시지 격리를 동시에 가능하게 하는 핵심 개념입니다. 같은 그룹 내 Consumer들은 파티션을 나눠서 담당하기 때문에 중복 없이 병렬로 메시지를 처리할 수 있고, 파티션 수가 병렬 처리의 상한이 됩니다. 다른 그룹은 동일 토픽을 독립적으로 소비할 수 있어 분석 파이프라인과 실시간 처리 서비스가 같은 이벤트를 각자 처리하는 패턴이 가능합니다. 파티션 할당 전략은 상황에 따라 선택이 달라집니다. RangeAssignor는 토픽별로 파티션 범위를 나누는 방식인데 토픽이 많아지면 특정 Consumer에 편중될 수 있고, RoundRobinAssignor는 전체 파티션을 균등하게 배분합니다. StickyAssignor는 리밸런싱 시 기존 할당을 최대한 유지하고, CooperativeStickyAssignor는 변경이 필요한 파티션만 단계적으로 재배정하는 Incremental 방식으로 Stop-The-World가 없어 신규 배포 환경에서 가장 권장됩니다. 병렬 처리를 확장할 때는 파티션 수와 Consumer 수를 함께 늘려 파티션당 Consumer 1개 기준으로 선형 확장하거나, `max.poll.records`를 늘려 한 번에 더 많은 메시지를 가져와 처리하는 방법을 활용합니다. Consumer 내부에서 스레드 풀로 병렬화하는 방법도 있지만 순서 보장이 깨질 수 있어 주의가 필요합니다.

**파티션 할당 전략**:
| 전략 | 특징 |
|---|---|
| RangeAssignor | 토픽별 파티션 범위 나눔. 토픽 많으면 불균형 |
| RoundRobinAssignor | 전체 파티션 균등 배분 |
| StickyAssignor | 리밸런싱 시 기존 할당 최대한 유지 |
| **CooperativeStickyAssignor** | Incremental 리밸런싱 → Stop-The-World 없음. 신규 배포 권장 |

**꼬리 질문 예시**:
- Consumer 수를 파티션 수보다 많이 늘리면 어떻게 되나요?
- 파티션 수를 늘리면 무조건 좋은가요? 파티션 증가의 단점은?

## Consumer Rebalancing으로 인한 처리 중단을 최소화하는 설계

**난이도**: 심화

**핵심 키워드**: CooperativeStickyAssignor, Static Membership, session.timeout.ms, max.poll.interval.ms, KRaft, Rolling Update

**모범 답변:**
Consumer Rebalancing은 Kafka 운영에서 처리 중단의 주요 원인 중 하나입니다. 기본 Eager Rebalancing 방식은 리밸런싱이 시작되면 모든 Consumer가 보유한 파티션을 전부 반납하고 재배정을 기다리는 Stop-The-World 방식이기 때문에 수십 초의 처리 중단이 발생할 수 있습니다. 이를 해결하는 첫 번째 방법은 `CooperativeStickyAssignor`를 사용하는 것입니다. 이 전략은 Incremental 방식으로 변경이 필요한 파티션만 단계적으로 재배정하기 때문에 나머지 Consumer들은 리밸런싱 중에도 계속 처리를 유지할 수 있고, 기존 30초 이상의 중단을 수 초 이내로 줄일 수 있습니다. 두 번째 방법은 Static Group Membership으로, `group.instance.id`에 고정값을 부여하면 Consumer가 재시작했을 때 동일한 ID로 재합류해 리밸런싱 없이 기존 파티션을 그대로 재획득합니다. Rolling Update와 조합하면 배포 중 처리 중단을 대폭 줄일 수 있습니다. 타임아웃 설정도 중요합니다. `session.timeout.ms`는 브로커가 Consumer 장애를 감지하는 시간으로 기본값은 45초이고, `max.poll.interval.ms`는 `poll()` 호출 간격의 최대치입니다. Static Membership 환경에서 `session.timeout.ms` 이내에 재접속하면 리밸런싱 없이 복구됩니다. KRaft 도입 이후 ZooKeeper를 제거하면서 컨트롤러가 KRaft 기반으로 동작하게 됐고, 파티션 메타데이터 관리 속도가 빨라져 리밸런싱 자체의 속도도 개선됐습니다.

**꼬리 질문 예시**:
- `CooperativeStickyAssignor`와 `StickyAssignor`의 차이점은 무엇인가요?
- Static Membership에서 `session.timeout.ms`를 너무 길게 잡으면 어떤 문제가 생기나요?
- KRaft 모드에서 Leader Election 방식은 ZooKeeper 기반과 어떻게 다른가요?

---
