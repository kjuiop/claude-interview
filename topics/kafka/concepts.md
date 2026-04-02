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

## 메시지 순서 보장

### 조건
- **파티션 내에서만** 순서 보장 — 파티션 간 순서는 보장 안 됨
- 동일 키 → 동일 파티션 라우팅 (기본 파티셔너: `hash(key) % partition_count`)
- 동일 파티션 내 메시지는 Producer가 보낸 순서대로 저장 + 소비

### 파티션 수 변경 시 문제
- 파티션 수 변경 → `hash(key) % 새_파티션_수` 재계산 → 기존 키가 다른 파티션으로
- **리밸런싱 중 순서 깨짐**:
  - 기존 파티션(P0) 미처리 메시지 존재
  - 변경 후 같은 키의 새 메시지 → 다른 파티션(P2)
  - 두 파티션을 다른 컨슈머가 처리 → 전체 순서 보장 불가
- **해결**: 파티션 수 변경 전 기존 파티션 미처리 메시지 완전 소진 후 변경

### 설계 전략
| 목표 | 전략 |
|---|---|
| 완전한 순서 보장 | 파티션 1개 + 컨슈머 1개 (처리량 희생) |
| 처리량 + 순서 | 파티션 수 초기에 충분히 설정 (최대 컨슈머 수 × 2~3배), 이후 컨슈머로 조절 |
| 재시도 시 순서 역전 방지 | `max.in.flight.requests.per.connection=1` |

---

## Kafka 아키텍처 핵심 구성 요소

### Broker
- Kafka 클러스터를 구성하는 **서버 노드**. 메시지를 받아 디스크에 저장하고 Consumer에게 전달
- 여러 Broker가 모여 Kafka Cluster를 구성
- 각 Broker는 고유 ID(broker.id)를 가지며, 일부 Partition의 Leader 역할 담당
- **Controller Broker**: 클러스터 내 특별 역할 — Leader 선출, 파티션 재배정 관리

### Topic & Partition
- **Topic**: 메시지를 분류하는 논리적 단위 (DB의 테이블 유사)
- **Partition**: Topic을 나누는 물리적 단위. 각 파티션은 독립된 순서가 있는 로그 파일
- 파티션은 여러 Broker에 분산 저장 → **수평 확장** 가능
- 파티션 수 = 최대 병렬 처리 가능한 Consumer 수 상한선

### Producer 원리
- 메시지를 **Broker의 특정 파티션 Leader**에게 직접 전송
- **Partition Key**: 키 있으면 `hash(key) % partition_count`로 파티션 결정 → 동일 키는 동일 파티션 보장
- 키 없으면 Round-Robin 또는 Sticky Partitioner(배치 단위 고정)로 분산

**Producer Batch Processing**:
- 메시지를 즉시 보내지 않고 **배치로 묶어서 전송** → 네트워크 왕복 횟수 감소
- `linger.ms`: 배치 대기 시간 (기본 0ms — 즉시 전송). 높일수록 처리량↑, 지연↑
- `batch.size`: 배치 최대 크기 (기본 16KB). 도달 시 즉시 전송
- `compression.type`: gzip/snappy/lz4 — 배치 단위로 압축 → 처리량 향상
- `buffer.memory`: 전송 대기 버퍼 총 크기

### Consumer 원리
- **Pull 방식**: Consumer가 Broker에 주기적으로 poll() 요청 → 메시지 가져옴
- 읽은 메시지를 삭제하지 않음 → **여러 Consumer Group이 독립적으로 소비 가능**
- `max.poll.records`: 한 번 poll()에서 가져올 최대 메시지 수

**Consumer Offset & Commit**:
- **Offset**: 파티션 내 메시지의 고유 순번 (0부터 시작)
- Consumer는 자신이 읽은 위치(offset)를 `__consumer_offsets` 토픽에 커밋
- **자동 커밋** (`enable.auto.commit=true`): 주기적 자동 커밋 → 처리 전 크래시 시 메시지 유실 가능
- **수동 커밋**: 처리 완료 후 명시적 `commitSync()` / `commitAsync()` 호출 → At-least-once 보장

### Consumer Group
- **같은 그룹 ID**를 가진 Consumer들의 집합
- 하나의 파티션은 **같은 그룹 내 하나의 Consumer만** 할당됨 → 중복 처리 방지
- 다른 그룹들은 독립적으로 같은 토픽을 소비 → 각 그룹별 독립 offset 관리
- Consumer 수 > 파티션 수: 초과 Consumer는 idle 상태 (낭비)
- Consumer 수 < 파티션 수: 일부 Consumer가 여러 파티션 담당

**Consumer 파티션 할당 전략**:
| 전략 | 방식 |
|---|---|
| RangeAssignor | 토픽별로 파티션 범위를 나눠 할당. 토픽 수 많으면 불균형 발생 |
| RoundRobinAssignor | 전체 파티션을 순서대로 균등 배분 |
| StickyAssignor | 리밸런싱 시 기존 할당 최대한 유지 + 균등 배분 |
| **CooperativeStickyAssignor** | StickyAssignor + Incremental 리밸런싱 (Stop-The-World 방지) |

---

## Kafka Cluster & 복제 구조

### Replica (복제본)
- 각 파티션은 N개의 복제본을 가짐 (`replication.factor` 설정)
- **Leader Replica**: 모든 읽기/쓰기 요청을 처리하는 실제 파티션
- **Follower Replica**: Leader를 복제하여 장애 대비 대기. 클라이언트와 직접 통신 안 함
- 파티션의 Leader와 Follower는 **서로 다른 Broker**에 분산 배치 → 장애 격리

### ISR (In-Sync Replicas)
- **Leader와 동기화 상태인 Replica 집합**
- Follower가 Leader의 최신 메시지를 따라잡고 있으면 ISR에 포함
- 지정 시간(`replica.lag.time.max.ms`, 기본 30초) 내 복제 못하면 ISR에서 제외
- Leader 장애 시 **ISR 내 Follower 중에서만** 새 Leader 선출 → 데이터 유실 방지

### min.insync.replicas
- 메시지가 **커밋되기 위해 최소로 동기화되어야 하는 ISR 수**
- `acks=all`(Producer 설정)과 함께 사용해야 의미 있음

| acks 설정 | 동작 |
|---|---|
| `acks=0` | 브로커 응답 안 기다림 → 최고 성능, 유실 가능 |
| `acks=1` | Leader 기록 확인만 → Leader 장애 시 유실 가능 |
| `acks=all`(-1) | **ISR 전체** 기록 확인 → 가장 안전, 지연 증가 |

**`min.insync.replicas` 조합 예시** (replication.factor=3):
```
min.insync.replicas=1: ISR 1개만 있어도 커밋 → 가용성↑, 내구성↓
min.insync.replicas=2: ISR 최소 2개 필요 → 균형 (권장)
min.insync.replicas=3: ISR 3개 모두 필요 → 내구성 최대, 브로커 1개만 죽어도 쓰기 불가
```
- `acks=all` + `min.insync.replicas=2`가 실무 권장 조합

### Leader 선출 과정
1. Broker 장애 감지 (Controller Broker 또는 KRaft 감지)
2. 해당 파티션의 ISR 목록 조회
3. ISR 중 첫 번째 Broker를 새 Leader로 선출
4. 클라이언트에 메타데이터 업데이트

---

## 병렬 Consumer (Parallel Consumer)

- **파티션 수 = 최대 병렬도**: 파티션 10개 → 동시에 최대 10개 Consumer가 병렬 처리
- 처리량 향상 전략:
  1. **파티션 수 증가** + Consumer 수 증가 → 순평행 처리
  2. **Consumer 내부 멀티스레드**: 하나의 Consumer가 받은 메시지를 스레드 풀로 처리 (순서 보장 어려워짐)
  3. **Consumer Group 복수 운영**: 서로 다른 역할의 서비스가 같은 토픽 독립 소비
- 주의: Consumer 수 > 파티션 수면 초과 Consumer는 유휴 상태 (비용 낭비)

---

## KRaft 모드 (Kafka 4.0+)
- ZooKeeper 완전 제거 → Kafka 자체 Raft 합의 알고리즘으로 메타데이터 관리
- Controller 역할을 전담하는 KRaft Controller가 클러스터 상태 관리
- 장점: 운영 단순화, 빠른 Controller Failover, 파티션 수 확장성 개선
