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

## Lua 스크립트 — 상세 동작 원리

Redis 2.6.0부터 지원. 서버 내장 Lua 5.1 인터프리터가 스크립트를 **단일 원자적 명령**으로 실행.

### 기본 구조

```lua
-- EVAL script numkeys key [key ...] arg [arg ...]
EVAL "
  local val = redis.call('GET', KEYS[1])
  if val == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  end
  return 0
" 1 mykey myvalue
```

- `KEYS[N]`: Redis 키 인자 (1-indexed). Cluster에서 라우팅 결정에 사용
- `ARGV[N]`: 키 이외의 값 인자
- `redis.call()`: 명령 실행 — 오류 발생 시 클라이언트로 에러 전파
- `redis.pcall()`: 명령 실행 — 오류 발생 시 스크립트 내에서 핸들링 가능 (try-catch 유사)

### redis.call() vs redis.pcall()

```lua
-- redis.call(): 에러 발생 시 즉시 클라이언트에 에러 반환 (스크립트 중단)
local result = redis.call('GET', KEYS[1])

-- redis.pcall(): 에러를 테이블로 반환 → 스크립트 내에서 처리 가능
local ok, err = pcall(function()
  return redis.call('INCR', KEYS[1])  -- KEYS[1]이 string이면 에러
end)
if not ok then
  return "error handled"
end
```

**선택 기준**: 에러 발생 시 즉시 중단해도 되면 `redis.call()`, 에러를 스크립트 안에서 처리해야 하면 `redis.pcall()`.

### EVAL vs EVALSHA — 스크립트 캐싱

```bash
# 매번 전체 스크립트 전송 (비효율)
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# SCRIPT LOAD: 스크립트를 서버에 캐싱 → SHA1 해시 반환
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "e0e1f9fabfa9d353e7f0942b4af7d46c7a914c69"

# EVALSHA: SHA1로 캐싱된 스크립트 실행 (스크립트 재전송 불필요)
EVALSHA e0e1f9fabfa9d353e7f0942b4af7d46c7a914c69 1 mykey
```

**EVALSHA 주의사항**:
- 스크립트 캐시는 **휘발성** — 서버 재시작, `SCRIPT FLUSH`, 페일오버 시 캐시 소멸
- EVALSHA 실행 중 캐시가 없으면 `NOSCRIPT` 에러 → 애플리케이션에서 EVAL로 폴백 처리 필요
- 파이프라인에서 EVALSHA 사용 시 NOSCRIPT 에러 처리 어려움 → 파이프라인에서는 EVAL 권장

```go
// go-redis에서 EVALSHA + NOSCRIPT 폴백 패턴
sha := "e0e1f9..."
result, err := rdb.EvalSHA(ctx, sha, []string{"mykey"}).Result()
if err != nil && strings.Contains(err.Error(), "NOSCRIPT") {
    // 스크립트 재로드 후 재실행
    result, err = rdb.Eval(ctx, script, []string{"mykey"}).Result()
}
```

---

## Lua 스크립트 실무 패턴

### 패턴 1: 분산락 해제 (대표 사례)

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

→ MULTI/EXEC로는 GET 후 값 보고 분기할 수 없어 **반드시 Lua** 사용

### 패턴 2: 재고 차감 (조건부 갱신)

```lua
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock == nil or stock <= 0 then
    return -1  -- 재고 없음
end
return redis.call('DECRBY', KEYS[1], ARGV[1])
```

→ GET 후 조건 확인 + DECRBY를 원자적으로 처리. MULTI/EXEC + WATCH로도 가능하지만 충돌 시 재시도 필요 — Lua가 더 단순

### 패턴 3: Cache Stampede 방지 (PER 패턴)

Cache Stampede: 캐시 만료 순간 다수의 요청이 동시에 DB 조회 → DB 과부하

```lua
-- 남은 TTL을 확인해 특정 확률로 미리 갱신 트리거
local ttl = redis.call('TTL', KEYS[1])
local threshold = tonumber(ARGV[1])  -- 갱신 시작 임계 TTL
if ttl < threshold then
    -- 락을 걸어 단 하나의 요청만 DB 조회
    if redis.call('SET', KEYS[2], '1', 'NX', 'PX', 5000) then
        return 1  -- 이 요청이 갱신 담당
    end
end
return 0  -- 기존 캐시 사용
```

> 출처:
> - https://redis.io/docs/latest/develop/programmability/eval-intro/
> - https://engineering.linecorp.com/ko/blog/atomic-cache-stampede-redis-lua-script/
> - https://oneuptime.com/blog/post/2026-01-21-redis-lua-scripts-atomic-operations/view

---

## Redis Cluster 환경에서의 MULTI/EXEC vs Lua

Redis Cluster에서는 키가 **해시 슬롯(hash slot)** 에 따라 다른 노드에 분산된다.

### MULTI/EXEC의 한계 (Cluster)

```bash
MULTI
GET key1   # 노드1에 있는 키
GET key2   # 노드2에 있는 키 ← 다른 노드!
EXEC       # ❌ Cross-slot 에러 발생
```

MULTI/EXEC는 **단일 노드에서만 동작** — 서로 다른 슬롯의 키를 한 트랜잭션으로 묶을 수 없음.

### Lua 스크립트의 Cluster 동작

```bash
EVAL "..." 2 key1 key2  # KEYS[1]=key1, KEYS[2]=key2
```

- KEYS 인자로 키를 명시하면 Redis Cluster가 **첫 번째 키의 슬롯 노드로 스크립트를 라우팅**
- 단, 스크립트 내 모든 키는 **같은 해시 슬롯에 있어야** 함
- Hash Tag `{user:123}:lock` 같은 패턴으로 같은 슬롯 보장 가능

```bash
# Hash Tag로 같은 슬롯 보장
SET {user:123}:balance 100
SET {user:123}:lock ""
# → 두 키 모두 "user:123" 기준 같은 슬롯
```

> 출처: https://dgle.dev/redis-multi-lua/

---

## MULTI/EXEC vs Lua 스크립트 비교

| 항목 | MULTI/EXEC | Lua 스크립트 |
|---|---|---|
| 원자성 | ✅ 보장 | ✅ 보장 |
| 조건부 로직 | ❌ 불가 | ✅ if/else 가능 |
| 롤백 | ❌ 없음 | ❌ 없음 |
| 네트워크 효율 | 왕복 2회(MULTI+EXEC) | 왕복 1회 |
| 복잡한 처리 | ❌ 제한적 | ✅ 복잡한 로직 가능 |
| Redis Cluster | ❌ 단일 노드만 (Cross-slot 불가) | ✅ KEYS 인자로 라우팅 |
| 에러 처리 | 명령별 개별 에러 | redis.call/pcall 선택 |
| 디버깅 | 쉬움 | 어려움 (서버 사이드 실행) |
| 사용 시나리오 | 단순 다중 명령 묶음 | 조건부 갱신, 분산락 해제 |

**선택 기준**:
- 단순히 여러 명령을 원자적으로 묶고 싶다 → MULTI/EXEC
- 조건부 처리가 필요하다 (GET 후 값 보고 분기) → Lua 스크립트
- 분산락 해제처럼 확인+실행 원자성 → Lua 스크립트
- Redis Cluster 환경에서 여러 키 원자 처리 → Lua 스크립트 (Hash Tag 활용)
- 스크립트 로직이 복잡하고 반복 호출이 많다 → SCRIPT LOAD + EVALSHA 캐싱

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
