---
tags: [redis, cache, distributed-lock, backend]
related: [distributed-systems, golang, kubernetes]
---

# Redis 핵심 개념

→ [[home]] | [[topics/redis/questions]] | [[topics/distributed-systems/concepts]]

---

## 분산락 (Distributed Lock)

여러 서버/프로세스가 동시에 같은 자원에 접근하지 못하도록 Redis를 이용해 잠금을 거는 패턴.

**사용 시나리오**: 재고 차감, 중복 결제 방지, 스케줄러 중복 실행 방지, 선착순 처리.

### SETNX 방식

```
SET lock:resource_id unique_value NX PX 5000
```

- `NX`: 키가 없을 때만 세팅 (원자적)
- `PX 5000`: 만료 시간 5초 (milliseconds). **필수** — 없으면 서버 crash 시 락 영구 잠김
- `unique_value`: 자신이 건 락인지 확인하기 위한 고유값 (UUID 등)

**락 해제 — Lua 스크립트 필수**:
```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```
- 확인(GET)과 삭제(DEL)를 원자적으로 처리. Lua 없이 하면 확인 후 삭제 사이에 다른 프로세스가 락을 가져갈 수 있음.

**단점**: Redis 단일 노드 장애 시 락 유실. 페일오버 직후 두 클라이언트가 동시에 락을 획득하는 상황 발생 가능.

### Redlock 알고리즘

Redis 노드 **N개(홀수, 보통 5개)** 에 동시 획득 시도 → **과반수(N/2+1개)** 이상 성공 시 락 획득.

```
노드1 SET lock NX PX 5000 → 성공
노드2 SET lock NX PX 5000 → 성공
노드3 SET lock NX PX 5000 → 성공  ← 과반수 달성 → 락 획득
노드4 SET lock NX PX 5000 → 실패
노드5 SET lock NX PX 5000 → 실패
```

- 단일 노드 장애에도 안전
- 실제 유효 시간 = TTL - 획득에 걸린 시간
- 실무에서는 `go-redis/redislock` 라이브러리 사용

**선택 기준**:
- Redis Cluster로 고가용성 이미 보장 → SETNX로 충분
- 강한 일관성이 필요한 크리티컬 자원 → Redlock

> 출처: Redis 공식 문서 - https://redis.io/docs/manual/patterns/distributed-locks/

---

## MULTI / EXEC — Redis 트랜잭션

여러 명령을 하나의 원자적 단위로 묶어 실행하는 Redis의 트랜잭션 메커니즘.

### 동작 흐름

```
MULTI          → 트랜잭션 시작, OK 응답
SET foo bar    → QUEUED (즉시 실행 안 됨)
INCR counter   → QUEUED
EXEC           → 큐에 쌓인 명령 일괄 실행, 배열로 응답 반환
```

- `MULTI` ~ `EXEC` 사이 명령은 즉시 실행되지 않고 **클라이언트 큐에 쌓임**
- `EXEC` 호출 시 큐의 모든 명령을 **단일 격리 단위로 순차 실행**
- 실행 중 다른 클라이언트 명령이 끼어들지 않음 (격리 보장)
- `DISCARD`: 트랜잭션 취소, 큐 비움

### ⚠️ 롤백 없음 (RDBMS와 핵심 차이)

```
MULTI
SET foo bar    → 성공
INCR foo       → 실패 (foo가 string이라 INCR 불가)
EXEC           → 실패한 명령만 건너뜀, 나머지는 정상 실행
```

**명령 실패해도 롤백되지 않음.** 부분 성공 상태가 될 수 있음.

### WATCH — 낙관적 잠금 (Optimistic Lock)

```
WATCH mykey         → mykey 감시 시작
GET mykey           → 현재값 읽기
MULTI
SET mykey newval    → QUEUED
EXEC                → WATCH 이후 mykey가 변경됐으면 nil 반환 (트랜잭션 취소)
                      변경 없으면 정상 실행
```

- `EXEC` 전에 WATCH 대상 키가 변경되면 트랜잭션 자동 취소
- 실패 시 재시도(retry) 로직을 애플리케이션에서 구현해야 함
- **낙관적 잠금**: 충돌이 드물 때 효율적. 충돌 빈번하면 비효율.

---

## Pipeline vs TxPipeline (go-redis)

[[golang]] 의 `go-redis` 라이브러리에서 제공하는 두 가지 배치 실행 방식.

### Pipeline — 네트워크 최적화

```go
pipe := rdb.Pipeline()
incr := pipe.Incr(ctx, "counter")
pipe.Expire(ctx, "counter", time.Hour)
_, err := pipe.Exec(ctx)
// → 2개 명령을 한 번의 네트워크 왕복으로 전송
```

- 여러 명령을 한 번의 네트워크 왕복(round trip)으로 전송
- **원자성 보장 없음** — 명령 사이에 다른 클라이언트가 끼어들 수 있음
- 목적: 네트워크 레이턴시 절감

### TxPipeline — MULTI/EXEC 래핑

```go
pipe := rdb.TxPipeline()
incr := pipe.Incr(ctx, "counter")
pipe.Expire(ctx, "counter", time.Hour)
_, err := pipe.Exec(ctx)
// 실제 전송: MULTI → INCR counter → EXPIRE counter 3600 → EXEC
```

- 내부적으로 `MULTI` + 명령들 + `EXEC`를 한 번에 전송
- **원자성 보장** — 명령 사이 다른 클라이언트 끼어들기 불가
- 단독 사용 시 Pipeline보다 약간 오버헤드 있음

### TxPipelined + WATCH — 낙관적 잠금 구현

```go
err := rdb.Watch(ctx, func(tx *redis.Tx) error {
    // 현재값 읽기
    val, err := tx.Get(ctx, "counter").Int()
    if err != nil {
        return err
    }

    // WATCH 이후 counter가 변경되면 TxFailed
    _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
        pipe.Set(ctx, "counter", val+1, 0)
        return nil
    })
    return err
}, "counter")

// 충돌 시 재시도
if err == redis.TxFailedErr {
    // retry
}
```

- `rdb.Watch()` → 키 감시 + `tx.TxPipelined()` → 조건부 트랜잭션
- `redis.TxFailedErr` 반환 시 애플리케이션에서 재시도 처리

---

## MULTI/EXEC vs Lua 스크립트 비교

| 항목 | MULTI/EXEC | Lua 스크립트 |
|---|---|---|
| 원자성 | ✅ 보장 | ✅ 보장 |
| 조건부 로직 | ❌ 불가 | ✅ if/else 가능 |
| 롤백 | ❌ 없음 | ❌ 없음 |
| 네트워크 효율 | 왕복 2회(MULTI+EXEC) | 왕복 1회 |
| 복잡한 처리 | ❌ 제한적 | ✅ 복잡한 로직 가능 |
| 사용 시나리오 | 단순 다중 명령 묶음 | 조건부 갱신, 분산락 해제 |

**선택 기준**:
- 단순히 여러 명령을 원자적으로 묶고 싶다 → MULTI/EXEC
- 조건부 처리가 필요하다 (GET 후 값 보고 분기) → Lua 스크립트
- 분산락 해제처럼 확인+실행 원자성 → Lua 스크립트

> 출처:
> - https://redis.uptrace.dev/guide/go-redis-pipelines.html
> - https://dgle.dev/redis-multi-lua/
> - https://redis.io/docs/latest/develop/using-commands/transactions/

---

## Redis Streams vs Pub/Sub

### 핵심 차이

| | Pub/Sub | Streams |
|---|---|---|
| 메시지 저장 | 없음 (전달 즉시 사라짐) | **Append-only log** (영속) |
| 전달 보장 | at-most-once (손실 가능) | at-least-once + ACK |
| 구독자 부재 시 | 메시지 **유실** | 메시지 **보존** |
| 재처리(Replay) | 불가 | 가능 (특정 ID부터 재읽기) |
| Consumer Group | 없음 | 있음 (작업 분산 + 중복 방지) |
| 사용 시나리오 | 실시간 브로드캐스트, 손실 허용 | 결제·주문 등 손실 불허 |

### Pub/Sub 사용 시나리오
- 라이브 채팅, 실시간 알림, 게임 상태 동기화
- 메시지 손실 허용, 극도로 낮은 레이턴시 필요

```bash
PUBLISH chat:room1 "hello"
SUBSCRIBE chat:room1
```

### Redis Streams 핵심 명령어

```bash
# 메시지 발행 (ID 자동 생성: timestamp-sequence)
XADD events * user_id 123 action "purchase"
# → "1709020800000-0"

# Consumer Group 생성
XGROUP CREATE events payment-group $ MKSTREAM
# $: 새 메시지부터 / 0: 처음부터

# Consumer가 읽기 (>: 아직 미처리 메시지만)
XREADGROUP GROUP payment-group worker-1 COUNT 5 STREAMS events >

# 처리 완료 후 ACK
XACK events payment-group 1709020800000-0

# Pending 목록 확인 (ACK 안 된 메시지)
XPENDING events payment-group

# 일정 시간 이상 미처리 메시지 다른 consumer로 이양
XAUTOCLAIM events payment-group worker-2 60000 0-0 COUNT 100
```

### Pending Entries List (PEL)
- `XREADGROUP`으로 읽은 메시지는 자동으로 PEL에 추가됨
- `XACK` 호출 전까지 PEL에 남아있음
- Consumer 장애 시 → `XCLAIM`/`XAUTOCLAIM`으로 다른 consumer가 인수인계
- PEL 미모니터링 시 메모리 누적 주의 → `XPENDING` 정기 확인 필요

### Dead Letter Queue (DLQ) 패턴
재시도 횟수 초과 메시지를 별도 스트림으로 이동:
```
events 스트림 → worker 처리 실패 3회 → events_dlq 스트림으로 이동 → 수동 처리
```

### Pub/Sub 메시지 유실 보완 전략 (라이브 채팅)
- 유실 허용 근거: 라이브 채팅은 실시간성 우선
- 재연결 시 MongoDB에서 최근 N개 메시지 fetch로 보완
- 유실 불허 시: Redis Streams 또는 Kafka로 전환
