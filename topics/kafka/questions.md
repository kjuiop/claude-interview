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

**면접 세션 피드백 (2026-04-28 2회차)**:
- 잘한 점: enable.idempotence+PID/Sequence, isolation.level=read_committed, transactional.id, Outbox/Inbox 패턴까지 전 레이어를 계층별로 완벽히 설명. 10/10.
- 보완: transactional.id 설명에서 "전체 롤백"보다 "이전 Producer의 미완료 트랜잭션 Abort + 새 Producer Epoch 활성화"가 더 정확한 표현.
- 점수: 10/10

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

**auto.offset.reset 동작 조건 (2026-05-03 3회차 — 꼬리 질문 연속 실패)**:
- `auto.offset.reset`은 **커밋된 오프셋이 없을 때만** 동작 (이미 커밋된 오프셋 있으면 무시)
- `earliest`: 파티션 가장 처음부터 읽기
- `latest`: 새로 들어오는 메시지부터 읽기
- **동작하는 조건**: 신규 Consumer 그룹 최초 실행, 또는 retention 기간 초과로 오프셋 소멸 시

**면접 세션 피드백 (2026-05-03 3회차)**:
- 자동 커밋 위험성(처리 중 죽으면 유실), commitSync 블로킹 설명 맞음
- 취약: `auto.offset.reset` 동작 조건 전혀 모름(꼬리 2회 연속 "잘 모르겠습니다") — 암기 필요
- 암기 포인트: "커밋된 오프셋이 **없을 때만** 동작. 있으면 무시"

**면접 세션 피드백 (2026-05-03 5회차 — 재도전)**:
- auto.commit 위험성, auto.offset.reset 조건, earliest/latest 모두 정확 ✅ → 3회차 대비 개선
- 취약: commitSync/commitAsync 선택 기준 오답 — **처리 시간이 아닌 도메인 정확성 요구사항**
- 암기 포인트: "결제/주문=commitSync(유실 불허+처리량 희생), 이벤트로그=commitAsync(처리량 우선+소량 중복 허용)"
- 점수: 7/10 (3회차 5/10 → 개선)

**면접 세션 피드백 (2026-05-06 4회차 — 재실패 2/10)**:
- 동작 조건 오답: "브로커 재시작" → "Consumer 재시작" 으로 수정했으나 여전히 오답
- 핵심 정리: `auto.offset.reset`은 **__consumer_offsets에 유효한 커밋 오프셋이 없을 때만** 동작
  - ① 신규 Consumer Group (커밋 기록 없음)
  - ② Retention 만료로 커밋된 오프셋 위치가 삭제 (offset out of range)
  - 브로커/Consumer 재시작은 트리거 아님 — 재시작해도 커밋된 오프셋이 남아 있어 이어서 읽음
- **반드시 암기**: "커밋된 오프셋이 없거나 유효하지 않을 때만 동작. 있으면 완전히 무시."

**⚠️ commitAsync 재시도 주의사항 (면접 세션 피드백 2026-04-14)**:
`commitAsync` 실패 시 이전 offset으로 재시도하면 안 된다. 비동기 특성상 이미 더 높은 offset이 성공적으로 commit됐을 수 있기 때문이다. 재시도는 반드시 **현재 시점의 최신 offset 기준**으로만 해야 중복 commit을 방지할 수 있다.

```java
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        // ❌ 이전 offsets로 재시도 금지 — 더 높은 offset이 이미 commit됐을 수 있음
        // ✅ 로깅 후 다음 poll 시 자연스럽게 최신 offset으로 처리되도록 방치
        log.error("Commit failed for offsets {}", offsets, exception);
    }
});
```

**꼬리 질문 예시**:
- 자동 커밋 환경에서 메시지가 유실될 수 있는 시나리오를 설명해주세요.
- 파티션별로 다른 offset에서 커밋하려면 어떻게 하나요?
- commitAsync 실패 시 이전 offset으로 재시도하면 안 되는 이유는?

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

**⚠️ max.poll.interval.ms 실무 주의사항 (면접 세션 피드백 2026-04-14)**:
파티션 수 변경이나 consumer 재시작 없이도 rebalancing이 발생하는 가장 흔한 원인이다. Consumer가 **살아있고 heartbeat도 정상**인데, `poll()` 호출 간격이 `max.poll.interval.ms`(기본 5분)를 초과하면 broker가 해당 consumer를 죽었다고 판단해 rebalancing을 트리거한다. 처리 로직이 무거운 consumer에서 자주 발생하며, 처리 중이던 메시지를 다른 consumer가 재할당받아 **중복 처리**가 발생한다.

해결 방법:
- `max.poll.interval.ms` 값 늘리기 (처리 시간 상한에 맞게)
- `max.poll.records` 줄여 배치 크기 감소
- 처리 로직을 비동기로 분리해 poll 루프를 빠르게 유지

**모범 답변:**
Consumer Rebalancing은 Kafka 운영에서 처리 중단의 주요 원인 중 하나입니다. 기본 Eager Rebalancing 방식은 리밸런싱이 시작되면 모든 Consumer가 보유한 파티션을 전부 반납하고 재배정을 기다리는 Stop-The-World 방식이기 때문에 수십 초의 처리 중단이 발생할 수 있습니다. 이를 해결하는 첫 번째 방법은 `CooperativeStickyAssignor`를 사용하는 것입니다. 이 전략은 Incremental 방식으로 변경이 필요한 파티션만 단계적으로 재배정하기 때문에 나머지 Consumer들은 리밸런싱 중에도 계속 처리를 유지할 수 있고, 기존 30초 이상의 중단을 수 초 이내로 줄일 수 있습니다. 두 번째 방법은 Static Group Membership으로, `group.instance.id`에 고정값을 부여하면 Consumer가 재시작했을 때 동일한 ID로 재합류해 리밸런싱 없이 기존 파티션을 그대로 재획득합니다. Rolling Update와 조합하면 배포 중 처리 중단을 대폭 줄일 수 있습니다. 타임아웃 설정도 중요합니다. `session.timeout.ms`는 브로커가 Consumer 장애를 감지하는 시간으로 기본값은 45초이고, `max.poll.interval.ms`는 `poll()` 호출 간격의 최대치입니다. Static Membership 환경에서 `session.timeout.ms` 이내에 재접속하면 리밸런싱 없이 복구됩니다. KRaft 도입 이후 ZooKeeper를 제거하면서 컨트롤러가 KRaft 기반으로 동작하게 됐고, 파티션 메타데이터 관리 속도가 빨라져 리밸런싱 자체의 속도도 개선됐습니다.

**꼬리 질문 예시**:
- `CooperativeStickyAssignor`와 `StickyAssignor`의 차이점은 무엇인가요?
- Static Membership에서 `session.timeout.ms`를 너무 길게 잡으면 어떤 문제가 생기나요?
- KRaft 모드에서 Leader Election 방식은 ZooKeeper 기반과 어떻게 다른가요?

---

## Kafka Consumer에서 외부 API 호출 시 Rebalancing Storm 방지 패턴

**난이도**: 중급

**핵심 키워드**: max.poll.interval.ms, Rebalancing Storm, DB 중간 저장, 비동기 워커, DLQ, at-least-once, 멱등성

**면접 질문 예시**:
> Kafka Consumer에서 메시지를 처리할 때 외부 API를 동기로 호출하면 어떤 문제가 생기고, 어떻게 해결하나요?

**핵심 흐름**:

**문제**: Consumer에서 외부 API를 동기 호출 → 처리 100개 × 1초 = 100초 → max.poll.interval.ms 초과 → Rebalancing → 다른 Consumer가 같은 100개 재처리 → 똑같이 100초 걸림 → **Rebalancing Storm**

**해결**: DB를 중간에 끼워 Kafka offset 커밋과 외부 API 호출을 분리

```
Consumer: 메시지 수신 → DB에 PENDING 상태로 저장 → offset 커밋 (빠름)
별도 워커: DB에서 PENDING 읽음 → 외부 API 호출
         → 성공: DONE
         → 실패: retry_count 증가 (exponential backoff)
         → 3회 실패: FAILED + 워커가 직접 DLQ 토픽에 발행
```

**DLQ 주의사항**:
- Kafka에서는 RabbitMQ DLX처럼 브로커가 자동으로 DLQ로 라우팅하지 않음
- offset이 이미 커밋됐으므로 **워커가 직접 DLQ 토픽에 명시적으로 발행**해야 함
- 또는 DB의 별도 dead_letter_queue 테이블로 관리

**DB 상태 컬럼 설계**:
```sql
status: PENDING → PROCESSING → DONE / FAILED
retry_count: 0~3
next_retry_at: exponential backoff 시각
error_message: 마지막 실패 원인
```

**선택 기준**:
- 외부 API가 반드시 처리되어야 함 (결제, 재고 차감) → DB 중간 저장 패턴
- 일부 유실 허용 (로그 집계) → 비동기 즉시 ack + 멱등성

**꼬리 질문 예시**:
- Consumer가 DB 저장 후 offset 커밋 전에 죽으면? → 재시작 시 같은 메시지 재처리 → DB에 PENDING 중복 → unique 제약으로 방어
- DLQ로 보낸 메시지는 어떻게 재처리? → 원인 수정 후 DLQ Consumer가 원본 토픽으로 재발행 or 수동 처리
- 파티션 hotspot 발생 시 응급조치는? → 순서 불필요: Consumer 내부 멀티스레딩 / 순서 필요: 파티션 증가(Rebalancing 감수) / 근본: 파티션 키 재설계(user_id 기반)

---

## Rebalancing — max.poll.interval.ms vs session.timeout.ms

### Q. Kafka Consumer에서 max.poll.interval.ms가 초과되어 Rebalancing이 반복될 때 원인과 해결 방법을 설명해주세요. session.timeout.ms와 트리거 주체가 어떻게 다른지도 구분해주세요.

**핵심 키워드:** max.poll.interval.ms, poll() 호출 간격, LeaveGroup 요청, session.timeout.ms, heartbeat thread, Group Coordinator, Eager Rebalancing, Stop-the-World, Cooperative Sticky Rebalancing

**트리거 주체 대조 (핵심):**

| 설정 | 초과 시 동작 | 트리거 주체 |
|---|---|---|
| `max.poll.interval.ms` | Consumer가 스스로 **LeaveGroup 요청** 전송 | Consumer (능동) |
| `session.timeout.ms` | 브로커(Group Coordinator)가 heartbeat 미수신 감지 | Broker (능동) |

- `max.poll.interval.ms`: poll() 호출 간격 최대값. 메시지 처리 로직이 무거워 다음 poll()이 늦어지면 초과. "커밋 타임아웃"이 아님 — 처리 지연 → 커밋 지연이 동반되어 혼동.
- `session.timeout.ms`: heartbeat thread가 보내는 heartbeat의 응답 타임아웃. Consumer가 죽었는지 감지용.

**Eager Rebalancing STW:**
- 리밸런싱 시작 → 모든 Consumer가 파티션 전부 반납 → Group Coordinator가 전체 재배분
- 완료까지 그룹 전체 메시지 처리 중단 (Stop-the-World)

**Cooperative Sticky Rebalancing 개선:**
- 1단계: 재할당 필요한 파티션만 선별 반납, 나머지는 계속 처리
- 2단계: 반납된 파티션만 재배분
- 전체 중단 없이 일부만 일시 중단 → downtime 최소화

**꼬리 질문 예시:**
- max.poll.interval.ms를 늘리는 것이 항상 옳은 해결책인가요? → 아님. 처리 로직 비동기화 or chunk 크기 조정이 근본 해결
- Cooperative Rebalancing 적용 방법은? → `partition.assignment.strategy=CooperativeStickyAssignor`

**면접 세션 피드백 (2026-05-02 2회차)**:
- 잘한 점: poll 간격 제한 개념 정확, session.timeout.ms heartbeat 구분, STW와 Cooperative 개선 방향 파악
- 보완: **LeaveGroup 요청** 키워드 필수 암기. 트리거 주체(Consumer vs Broker) 대조 표현 추가.

---

## Producer 배치 설정 (처리량 최적화)

### Q. Kafka Producer의 `linger.ms`, `batch.size`, `acks` 설정이 처리량(throughput)과 지연(latency)에 미치는 영향을 설명해주세요.

**세 파라미터 역할:**

| 파라미터 | 역할 | 기본값 | 처리량↑ 설정 |
|---|---|---|---|
| `linger.ms` | 배치 대기 시간 | 0 (즉시 전송) | 높게 (예: 500) |
| `batch.size` | 배치 최대 크기(bytes) | 16KB | 높게 (예: 1MB) |
| `acks` | 브로커 수신 확인 수준 | 1 | 0 (확인 없음) |

**linger.ms와 batch.size는 OR 조건:**
- `linger.ms` 경과 **또는** `batch.size` 도달 → 먼저 충족되는 조건에서 즉시 전송
- 트래픽이 많으면 `linger.ms` 이전에 `batch.size`에 도달해 더 자주 전송됨

**acks 레벨 비교:**
- `acks=0`: 브로커 수신 확인 없음. 가장 높은 처리량, 메시지 유실 가능
- `acks=1`: 리더 브로커만 수신 확인. 중간 균형
- `acks=-1(all)`: ISR 전체 복제 완료 후 확인. 가장 안전, 지연 증가

**도메인별 선택 기준:**
- 이벤트 집계(소량 유실 허용): `acks=0`, `linger.ms=500`, `batch.size=1MB` → 처리량 극대화
- 결제/주문(유실 불가): `acks=-1` + `min.insync.replicas=2` → 안전 우선

**실무 팁:** kafka-ui에서 메시지 평균 바이트 확인 → `1MB / 평균 바이트` = 배치당 메시지 수 추정 가능

**면접 세션 피드백 (2026-05-03 1회차)**:
- 잘한 점: 수치 명확(500ms, 1MB, acks=0), 비즈니스 맥락(장애 아닌 500ms 지연), OR 조건 정확, kafka-ui 실측 경험 연결
- 보완: acks 0/1/-1 레벨 비교 + 도메인별 선택 기준 추가하면 완성

---

## Outbox Pattern — DB + Kafka 정합성 보장

**난이도**: 기초

**핵심 키워드**: Outbox Pattern, DB 트랜잭션 원자성, Kafka 롤백 불가, CDC, Debezium, Polling, at-least-once, 멱등성

**모범 답변 방향**:

Kafka에 직접 발행하면 DB 트랜잭션은 롤백되어도 Kafka 메시지는 이미 발행된 상태가 되어 정합성이 깨진다. Outbox 패턴은 메시지를 Kafka로 직접 보내지 않고 같은 트랜잭션 안에서 Outbox 테이블에 저장한다. DB 저장이 성공하면 메시지도 저장, 실패하면 함께 롤백되어 정합성이 보장된다.

**DB → Kafka 전송 방식:**
- Polling: 주기적으로 Outbox 테이블 조회 → 구현 쉬움, 지연 발생, DB 부하
- CDC(Debezium): binlog 감지 → 즉시 전송, 실시간, Debezium 운영 복잡도

**주의**: Outbox 패턴도 at-least-once 보장. 중복 발행 가능성 있어 Consumer 측 멱등성 처리 필요.

**면접 세션 피드백 (2026-05-04 2회차)**:
- 잘한 점: 직접 발행 문제(정합성), Outbox 구조, Polling/CDC 트레이드오프, Debezium 언급 정확
- 미언급: at-least-once + Consumer 멱등성

---

## Consumer lag 진단 및 확장 전략

### Q. Kafka Consumer lag이 지속적으로 증가할 때 원인을 어떻게 진단하고 해결하나요? 파티션 수와 Consumer 수의 수평 확장 한계도 함께 설명해주세요.

**난이도:** 중급
**핵심 키워드:** Log End Offset, 원인 분류(처리 느림 vs 생산 빠름), kafka-ui, kafka-exporter, KEDA, 파티션=Consumer 최대 병렬도, idle Consumer, Rebalancing

**lag 원인 분류 (먼저 구분)**
- Consumer 처리 로직이 느린 경우: DB 병목, 외부 API 호출 → 파티션 늘려도 근본 해결 안 됨 → 처리 로직 최적화 먼저
- 생산 속도 > 처리 속도인 경우: 파티션 + Consumer 확장으로 해결

**모니터링 방법**
- kafka-ui: Consumer Group별 lag 직접 확인
- Kubernetes: kafka-exporter → Prometheus → Grafana 시각화
- KEDA: lag 메트릭 기반 Consumer Pod 자동 확장

**수평 확장 한계**
- Consumer 수 < 파티션 수 → Consumer 추가로 처리량 증가
- Consumer 수 = 파티션 수 → 최대 병렬도
- Consumer 수 > 파티션 수 → 초과 Consumer는 idle 상태, 처리량 기여 없음

**면접 세션 피드백 (2026-05-05 2회차)**:
- 9/10 — 모니터링, 파티션:Consumer 비율, idle Consumer, Rebalancing 주의 모두 커버
- 보완: lag 원인 분류(처리 느림 vs 생산 빠름)를 답변 앞부분에 먼저 제시할 것
