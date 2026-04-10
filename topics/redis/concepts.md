---
tags: [redis, cache, distributed-lock, backend]
related: [distributed-systems, golang, kubernetes]
---

# Redis 핵심 개념

→ [[home]] | [[topics/redis/questions]] | [[topics/distributed-systems/concepts]]

---

## 자료구조 (Data Structures)

| 자료구조 | 내부 인코딩 | 사용 예 | 대표 명령 |
|---|---|---|---|
| **String** | Raw / int / embstr | 캐싱, 카운터, 세션 토큰 | GET, SET, INCR, SETNX |
| **List** | ziplist(소규모) / linkedlist | 작업 큐, 최근 N개 로그 | LPUSH, RPOP, LRANGE |
| **Hash** | ziplist(128필드↓) / hashtable | 사용자 프로필, 구조체 캐싱 | HSET, HGET, HMGET |
| **Set** | intset(정수) / hashtable | 태그, 좋아요, 집합 연산 | SADD, SISMEMBER, SINTER |
| **Sorted Set** | ziplist(128개↓) / skiplist+hashtable | 랭킹, 리더보드, 시간 순 정렬 | ZADD, ZRANGE, ZRANGEBYSCORE |
| **HyperLogLog** | — | 중복 제거 카운팅 (UV 집계), 오차 0.81% | PFADD, PFCOUNT |
| **Bitmap** | String 기반 | 일일 출석체크, 기능 플래그 | SETBIT, GETBIT, BITCOUNT |

---

## 싱글 스레드 원리와 원자성

**Redis는 싱글 스레드로 명령을 처리한다.** (정확히는 I/O 처리는 멀티 스레드, 명령 실행만 싱글 스레드 — Redis 6.0+)

### 왜 싱글 스레드인가?
- **원자성 자동 보장**: 모든 명령이 순차 실행 → Race Condition 없음
- **Lock 비용 없음**: 멀티스레드의 Mutex/동기화 오버헤드 없음
- **구현 단순성**: 데이터 구조를 동기화 걱정 없이 단순하게 구현 가능
- 메모리 연산이 주이므로 CPU 병목 드묾 → 싱글 스레드로도 충분한 처리량

### 원자성이 보장되는 범위
- **단일 명령**: `INCR`, `SETNX`, `GETSET` 등 → 항상 원자적
- **Lua 스크립트**: 스크립트 전체가 하나의 원자적 명령으로 실행
- **MULTI/EXEC**: 큐에 쌓인 명령 일괄 실행 (격리 O, 롤백 X)
- ⚠️ **Pipeline**: 원자성 보장 없음 — 네트워크 최적화 용도

### 주의: 블로킹 명령
- `KEYS *`, `SMEMBERS`, `HGETALL` 등 O(N) 명령은 싱글 스레드 블록 → 운영 환경 금지
- 대안: `SCAN`, `SSCAN`, `HSCAN` (커서 기반 점진적 조회)

---

## 메모리 성능 — 왜 Redis를 쓰는가?

### 인메모리 구조
- 모든 데이터를 **메모리(RAM)에 저장** → 디스크 I/O 없음
- 메모리 접근 속도: ~100ns vs 디스크 SSD: ~100μs → **약 1000배 빠름**
- 단순 Get/Set 기준 **초당 10만~100만 ops** 처리 가능

### 영속성 옵션
| 방식 | 동작 | 장점 | 단점 |
|---|---|---|---|
| **RDB (Snapshot)** | 주기적 스냅샷 파일(.rdb) 저장 | 파일 작음, 복구 빠름 | 마지막 스냅샷 이후 데이터 유실 가능 |
| **AOF (Append Only File)** | 모든 쓰기 명령을 로그 파일에 append | 데이터 유실 최소화 | 파일 커짐, 재시작 시 복구 느림 |
| **RDB + AOF 혼합** | 주기 스냅샷 + 그 이후 AOF 로깅 | 빠른 복구 + 최소 유실 | 복잡도 증가 |
| **No Persistence** | 영속성 없음 | 최고 성능 | 재시작 시 데이터 전부 유실 |

### 메모리 관리 — Eviction Policy
메모리 한계(`maxmemory`) 도달 시 키 제거 정책:

| 정책 | 동작 |
|---|---|
| `noeviction` | 메모리 초과 시 에러 반환 (기본값) |
| `allkeys-lru` | 전체 키 중 LRU(가장 오래 미사용) 제거 |
| `volatile-lru` | TTL 있는 키 중 LRU 제거 |
| `allkeys-lfu` | 전체 키 중 LFU(가장 적게 사용) 제거 |
| `volatile-ttl` | TTL 가장 짧은 키 먼저 제거 |

- 캐시 용도: `allkeys-lru` 권장
- 영속 데이터 혼재: `volatile-lru` (TTL 없는 키는 보존)

---

## 캐시 전략 (Cache Strategy)

### Cache Hit / Miss
- **Cache Hit**: 요청 데이터가 캐시에 존재 → Redis에서 즉시 반환
- **Cache Miss**: 캐시에 없음 → DB 조회 후 캐시에 저장
- **Hit Ratio** = Hit / (Hit + Miss) → 높을수록 DB 부하 감소

### 캐시 읽기 전략
| 전략 | 동작 | 특징 |
|---|---|---|
| **Cache-Aside (Lazy Loading)** | Miss 시 앱이 DB 조회 후 캐시 저장 | 가장 일반적. 실제 읽힌 데이터만 캐싱 |
| **Read-Through** | 캐시가 Miss 시 자동으로 DB 조회 | 앱 코드 단순, 캐시 레이어가 DB 접근 담당 |

### 캐시 쓰기 전략
| 전략 | 동작 | 일관성 |
|---|---|---|
| **Write-Through** | DB와 캐시를 동시에 쓰기 | 높음 (항상 최신) |
| **Write-Back (Behind)** | 캐시에만 먼저 쓰고 나중에 DB에 반영 | 낮음 (지연 반영), 성능 높음 |
| **Write-Around** | DB에만 쓰고 캐시는 Miss 시 로드 | 쓰기 직후 읽기는 항상 Miss |

### TTL 설계 원칙
- 너무 짧으면 → Cache Miss 증가, DB 부하
- 너무 길면 → 오래된 데이터(Stale) 서빙, 메모리 증가
- **TTL Jitter**: `ttl = base_ttl + random(0, base_ttl * 0.1)` → 대량 동시 만료 방지

---

## Java 클라이언트 — Lettuce vs Redisson

### Lettuce
- **Spring Data Redis 기본 클라이언트**
- Netty 기반 **비동기 Non-Blocking** I/O
- 단일 커넥션으로 멀티플렉싱 → 커넥션 수 적음
- Reactive(WebFlux) 지원
- 기본 기능 충실, 분산락 등 고수준 기능 직접 구현 필요

### Redisson
- 고수준 분산 객체/서비스 추상화 라이브러리
- **RLock**: Watchdog + Lua Script 기반 분산락 (TTL 자동 연장)
- **RMap, RList, RQueue**: Redis 자료구조를 Java 컬렉션처럼 사용
- 분산 세마포어, Rate Limiter, Pub/Sub 등 고수준 API
- 내부적으로 Netty + Lettuce 사용, 오버헤드 있음

| 항목 | Lettuce | Redisson |
|---|---|---|
| 기반 | Netty 비동기 | Netty + 고수준 추상화 |
| 분산락 | 직접 구현 (Lua Script) | `RLock` + Watchdog 내장 |
| 복잡도 | 낮음 | 높음 |
| 성능 | 더 가벼움 | 기능 많아 약간 무거움 |
| 선택 기준 | 단순 캐싱, WebFlux | 분산락, 분산 컬렉션 필요 시 |

---

## Redis Cluster

### 구조
- 데이터를 **16,384개의 해시 슬롯**으로 분할, 각 노드가 슬롯의 일부 담당
- 최소 **3개 마스터 노드** 권장 (각 마스터에 슬롯 ~5461개씩)
- 각 마스터에 최소 1개 이상의 레플리카(Replica) 권장

### 정족수(Quorum) — 클러스터 내결함성
- 클러스터는 **과반수(N/2+1) 마스터 노드가 살아있어야** 정상 동작
- 마스터 3개 기준: 1개 다운 → 클러스터 유지 / 2개 다운 → 클러스터 중단
- 마스터 장애 시: 해당 마스터의 레플리카 중 하나가 **자동 Failover**로 마스터 승격

### Fault Tolerance (내결함성)
```
마스터3 + 레플리카3 구성 (총 6노드):
  - 마스터 1개 장애 → 레플리카가 마스터 승격 → 클러스터 계속 동작
  - 마스터 2개 동시 장애 + 각각의 레플리카 없으면 → 클러스터 중단 (quorum 미달)
```

- `cluster-require-full-coverage yes` (기본): 일부 슬롯 손실 시 전체 클러스터 중단
- `cluster-require-full-coverage no`: 일부 슬롯 손실해도 나머지 슬롯은 서비스 지속

### Cluster vs Sentinel
| | Sentinel | Cluster |
|---|---|---|
| 목적 | 단일 마스터 고가용성(HA) | 수평 확장 + HA |
| 데이터 분산 | 없음 (전체 복제) | 슬롯 기반 분산 |
| 클라이언트 | Sentinel-aware 필요 | Cluster-aware 필요 |
| 적합 케이스 | 데이터 양 적고 HA만 필요 | 대용량 데이터, 쓰기 확장 |

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

## Sorted Set 내부 인코딩 구조 — ListPack vs SkipList

Redis Sorted Set은 원소의 개수와 크기에 따라 두 가지 내부 인코딩을 자동으로 선택한다.

### 인코딩 전환 기준

```
원소 수 ≤ 128 AND 모든 원소 크기 ≤ 64바이트  →  ListPack (구: ziplist)
위 조건 중 하나라도 초과                        →  SkipList + Hashtable
```

설정값:
- `zset-max-listpack-entries` (기본 128): 원소 수 임계값
- `zset-max-listpack-value` (기본 64): 단일 원소 크기 임계값(바이트)

전환은 단방향(ListPack → SkipList). 원소를 삭제해도 SkipList에서 ListPack으로 되돌아가지 않는다.

---

### ListPack (구: Ziplist)

**구조**: 모든 원소를 **단일 선형 메모리 블록**에 순차 저장.

```
[header(6바이트)][entry1][entry2]...[entryN][end(0xFF)]
각 entry: [encoding][length][value]
```

**특성**:
- 메모리 효율 최고 — 캐시 친화적 연속 메모리
- 접근 복잡도: **O(N)** (순차 스캔 필요)
- 원소 수가 적을 때는 O(N)이어도 실제 성능 충분

**Ziplist의 Cascading Update 문제** (ListPack이 대체한 이유):
- Ziplist의 각 엔트리는 이전 엔트리의 길이를 저장
- 중간 원소의 크기가 변하면 이전 길이 필드 크기가 바뀌고, 이것이 연쇄적으로 뒤 엔트리의 이전길이 필드까지 변경 → 최악 O(N) 재기록
- **ListPack은 이전 엔트리 길이를 저장하지 않음** → Cascading Update 제거

---

### SkipList + Hashtable

**SkipList 구조**: 정렬된 연결 리스트에 **다층 레이어(Express Lane)** 를 추가한 확률적 자료구조.

```
레이어 4:  HEAD ─────────────────────────── 80 ─── TAIL
레이어 3:  HEAD ────── 20 ─────────────────── 80 ─── TAIL
레이어 2:  HEAD ── 10 ─ 20 ──── 40 ──────── 80 ─── TAIL
레이어 1:  HEAD ─ 5 ─ 10 ─ 20 ─ 30 ─ 40 ─ 50 ─ 80 ─ TAIL  ← 전체 노드
```

**각 노드의 구성**:
```c
typedef struct zskiplistNode {
    sds ele;           // 멤버 문자열
    double score;      // 정렬 기준 점수
    struct zskiplistNode *backward;  // 역방향 포인터 (ZREVRANGE용)
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 다음 노드 포인터
        unsigned long span;             // 건너뛰는 노드 수 (순위 계산)
    } level[];  // 각 레이어의 forward + span
} zskiplistNode;
```

**Span의 역할**: 다음 노드까지 건너뛴 노드 수를 기록. `ZRANK`(순위 조회) 시 span을 누적해 O(log N)에 순위 계산 가능.

**레이어 높이 결정**: 삽입 시 확률적으로 결정 (p=0.25, 최대 32레이어). 평균 노드 수: O(1/p) = O(4).

**O(log N) 달성 원리**: 상위 레이어에서 큰 폭으로 건너뛰고, 목표에 가까워질수록 낮은 레이어로 내려와 정밀 탐색 → 이진 탐색과 유사한 성능.

**Hashtable 역할**: `score → member` 역방향 조회(`ZSCORE`)를 O(1)에 처리. SkipList만으로는 O(log N).

---

### SkipList vs 이진 탐색 트리(BST/RB-Tree) 선택 이유

Redis가 Red-Black Tree 대신 SkipList를 선택한 이유:

| 항목 | SkipList | Red-Black Tree |
|---|---|---|
| 범위 조회(ZRANGE) | 리프 레이어 순차 스캔 → 간단 | In-order traversal → 복잡 |
| 구현 복잡도 | 낮음 | 높음 (회전, 색 변경) |
| 메모리 지역성 | 레이어 포인터로 캐시 미스 발생 | 비슷 |
| 순위 계산 | Span으로 O(log N) | 추가 메타데이터 필요 |
| 동시성 | Lock-free 구현 용이 | 회전 시 복잡 |

**핵심 이유**: 범위 조회(`ZRANGEBYSCORE`, `ZRANGEBYRANK`)가 Sorted Set의 핵심 연산이며, SkipList의 최하위 레이어가 정렬된 연결 리스트여서 연속 스캔이 자연스럽다.

---

### 메모리 vs 성능 트레이드오프

| | ListPack | SkipList |
|---|---|---|
| 메모리 | 최소 (헤더 6바이트) | 노드당 최소 37바이트 오버헤드 |
| 삽입 복잡도 | O(N) (재메모리화) | O(log N) |
| 조회 복잡도 | O(N) | O(log N) |
| 캐시 효율 | 높음 (연속 메모리) | 낮음 (포인터 추적) |
| 적합 크기 | 128개 이하 소형 | 중·대형 집합 |

**실무 설정 튜닝**:
- 메모리 최적화 우선 (소규모 데이터): `zset-max-listpack-entries 512`로 올려 더 오래 ListPack 유지
- 성능 우선 (빠른 랭킹 조회): 기본값(128) 유지 또는 낮춤

> 출처:
> - https://redis.io/docs/latest/develop/data-types/sorted-sets/
> - https://jothipn.github.io/2023/04/07/redis-sorted-set.html
> - https://sam-wei.medium.com/deep-dive-into-redis-zset-internals-8d10fa1f674c
> - https://github.com/zpoint/Redis-Internals/blob/5.0/Object/zset/zset.md

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


---

## Rate Limiting — Token Bucket vs Sliding Window Log

→ [[topics/system-design/concepts]] | [[topics/redis/questions#Rate Limiting]]

### Token Bucket

- **원리**: 버킷에 일정 속도로 토큰이 채워지고(refill), 요청 시 토큰 1개 소비. 토큰 없으면 거절.
- **burst 허용**: 버킷에 남은 토큰 수만큼 순간 burst 허용 (설계상 의도)
- **Redis 구현**: String 또는 Hash 2필드

```
# Hash 방식 (userId + path 기준)
HSET rate:{userId}:{path} remaining 10 last_refill 1712100000
```

- **메모리**: O(1) — 유저당 2개 값(남은 토큰 수 + 마지막 refill 시간)만 저장

### Sliding Window Log

- **원리**: 요청마다 timestamp를 SortedSet에 기록. 현재 시간 기준으로 window 밖의 오래된 로그 제거 후 count
- **burst 엄격 제어**: 경계 시점에서도 정확한 count 가능
- **Redis 구현**: Sorted Set (score = timestamp)

```
ZADD rate:{userId}:{path} {timestamp} {request_id}
ZREMRANGEBYSCORE rate:{userId}:{path} 0 {window_start}
ZCARD rate:{userId}:{path}  # 현재 window 내 요청 수
```

- **메모리**: O(요청 수) — 모든 timestamp 유지, 트래픽 많을수록 메모리 증가

### 선택 기준

| 기준 | Token Bucket | Sliding Window Log |
|---|---|---|
| 메모리 | O(1) — 효율적 | O(요청 수) — 많이 사용 |
| burst 허용 | ✅ 허용 | ❌ 엄격 제어 |
| 구현 복잡도 | 낮음 | 높음 |
| 선택 상황 | 메모리 효율 우선, 적당한 burst 허용 | 정확한 QPS 제어 필요 시 |

> 세션 피드백 (2026-04-03 1회차): Token Bucket → String/Hash 2필드. "burst를 방지 못한다" 아닌 "burst를 허용하는 설계". 메모리 O(1) vs O(요청 수) 방향 주의.

---

## Hash 내부 인코딩 — listpack vs ziplist vs hashtable

Redis Hash는 데이터 크기에 따라 내부 인코딩을 자동으로 전환한다.

### 인코딩 전환 흐름

```
필드 수 ≤ 128 AND 모든 값 ≤ 64바이트
    → listpack (Redis 7.0+) / ziplist (Redis 7.0 이전)

임계값 초과
    → hashtable
```

설정 키:
- `hash-max-listpack-entries` (기본: 128) — 필드 수 임계값
- `hash-max-listpack-value` (기본: 64) — 개별 값 크기(byte) 임계값

### ziplist (Redis 7.0 이전)

연속된 메모리 블록에 항목을 순서대로 저장하는 컴팩트 리스트.

```
[zlbytes][zltail][zllen][entry1][entry2]...[entryN][zlend]
```

- 헤더 10바이트: 전체 크기, 마지막 항목 오프셋, 항목 수 포함
- 각 entry는 **이전 항목의 길이(prevlen)** 를 저장 → 역방향 탐색 가능
- **캐스케이딩 업데이트(cascading update) 문제**: 중간 항목을 삽입/삭제하면 prevlen이 바뀌고, 그 변경이 이후 모든 항목에 연쇄적으로 전파될 수 있음. 최악의 경우 O(N²) 재작성 발생.

### listpack (Redis 7.0+)

ziplist의 캐스케이딩 업데이트 문제를 해결하기 위해 재설계된 구조.

```
[total_bytes][num_elements][entry1][entry2]...[entryN][0xFF]
```

- 헤더 6바이트 (ziplist의 10바이트보다 작음)
- 각 entry가 **자신의 길이를 entry 끝에 저장** → 역방향 탐색 시 이전 항목 참조 불필요
- prevlen 없음 → 삽입/삭제 시 다른 항목에 영향 없음 → **O(1) 삽입/삭제**
- Redis 7.0에서 Hash, List, Sorted Set의 ziplist를 대체
- Redis 7.2에서 Set에도 적용

### hashtable

임계값 초과 시 자동 전환되는 표준 해시 테이블.

**내부 구조:**

```
버킷 배열 (bucket array)
  [0] → dictEntry → dictEntry → nil   (충돌 시 linked list chaining)
  [1] → nil
  [2] → dictEntry → nil
  ...
  [N] → nil
```

각 `dictEntry`는 3개의 포인터를 가진다:
```c
typedef struct dictEntry {
    void *key;    // 8바이트 (64bit 환경)
    void *val;    // 8바이트
    struct dictEntry *next;  // 8바이트 (chaining용)
} dictEntry;
// 합계: 24바이트 — 실제 데이터 없이 포인터만으로 24바이트
```

**왜 메모리를 더 많이 쓰는가:**

| 항목 | listpack | hashtable |
|---|---|---|
| 저장 방식 | 연속 메모리 (포인터 없음) | 버킷 배열 + 각 항목 포인터 3개 |
| 항목당 오버헤드 | ~0바이트 (길이 정보만) | 최소 24바이트 (key/val/next 포인터) |
| 빈 버킷 낭비 | 없음 | 있음 (load factor 1.0 이하 시 빈 슬롯 존재) |
| 키 저장 | 인라인 | SDS (Simple Dynamic String) 헤더 8바이트 + 내용 |
| 값 저장 | 인라인 | redisObject 헤더 16바이트 + 내용 |
| 5필드 예시 | ~100~200바이트 | ~500~800바이트 |

**rehashing (점진적 리해싱):**
- 버킷 수가 항목 수를 초과하면 2배로 확장 (항상 2의 제곱수)
- 리해싱 중 old/new 두 테이블을 동시에 유지 → 일시적으로 메모리 2배
- 매 명령 실행 시 일부 버킷씩 점진적으로 이전 (blocking 방지)

### 💬 면접 답변 형태로 읽기

hashtable이 listpack보다 메모리를 많이 쓰는 이유는 포인터 기반 구조 때문입니다. Redis의 hashtable은 버킷 배열과 각 버킷에 연결된 dictEntry 연결 리스트로 구성됩니다. 각 dictEntry는 key 포인터, value 포인터, 충돌 체이닝용 next 포인터 이렇게 3개의 포인터를 가지는데, 64비트 환경에서 포인터 하나가 8바이트이므로 실제 데이터가 아무것도 없어도 항목 하나에 최소 24바이트의 오버헤드가 붙습니다.

여기에 키는 SDS(Simple Dynamic String) 구조체로 감싸져 8바이트 헤더가 추가되고, 값은 redisObject로 감싸져 16바이트 헤더가 추가됩니다. 그리고 버킷 배열은 항상 2의 제곱수로 미리 할당되기 때문에 비어있는 슬롯이 생겨 추가 낭비가 발생합니다.

반면 listpack은 포인터가 전혀 없습니다. 키와 값을 연속된 메모리 블록에 그냥 순서대로 저장하고, 각 항목 앞뒤에 길이 정보를 붙이는 것이 전부입니다. 포인터 오버헤드도, 빈 버킷 낭비도, 별도 구조체 헤더도 없기 때문에 같은 5개 필드 데이터를 저장할 때 hashtable 대비 5~10배 적은 메모리를 사용할 수 있습니다.

그래서 Redis가 소규모 Hash에서 listpack을 쓰는 이유가 바로 이것입니다. 데이터가 작을 때 O(N) 선형 탐색의 비용은 미미한 반면, 메모리 절약 효과가 훨씬 크기 때문입니다. 임계값(필드 128개, 값 64바이트)을 넘어서 데이터가 많아지면 그때부터 O(1) 탐색의 이점이 메모리 비용보다 커지므로 hashtable로 전환합니다.

### 인코딩 확인 방법

```bash
OBJECT ENCODING user:1001
# listpack  → 소규모 (임계값 이하)
# hashtable → 임계값 초과
```

### 실무 팁

- 사용자 세션 5개 필드 → listpack 인코딩 적용, 메모리 효율 최대
- 필드를 의도적으로 128개 이하로 유지 → ziplist/listpack 유지
- 하나의 Hash key에 너무 많은 필드를 넣으면 hashtable 전환 → 메모리 증가

### 💬 면접 답변 형태로 읽기

Redis Hash는 데이터 크기에 따라 내부 인코딩을 자동으로 전환합니다. 필드 수가 128개 이하이고 각 값의 크기가 64바이트 이하일 때는 listpack 인코딩을 사용하고, 이 임계값을 초과하면 hashtable로 전환됩니다.

listpack은 Redis 7.0에서 이전의 ziplist를 대체한 구조입니다. 둘 다 연속된 메모리 블록에 항목을 순서대로 저장하는 컴팩트한 방식인데, ziplist는 각 항목이 **이전 항목의 길이(prevlen)** 를 저장하는 구조였습니다. 이 때문에 중간 항목을 삽입하거나 삭제할 때 prevlen 값이 바뀌고, 그 변경이 이후 모든 항목에 연쇄적으로 전파되는 **캐스케이딩 업데이트 문제**가 있었습니다. 최악의 경우 O(N²) 재작성이 발생할 수 있었습니다.

listpack은 이 문제를 해결하기 위해 각 항목이 자신의 길이를 **항목 끝에** 저장하는 방식으로 변경했습니다. 역방향 탐색 시 이전 항목을 참조하지 않아도 되므로, 삽입·삭제 시 다른 항목에 전혀 영향을 주지 않습니다. 헤더도 10바이트에서 6바이트로 줄었습니다.

메모리 효율 측면에서 listpack은 포인터 오버헤드 없이 연속 메모리에 데이터를 저장하기 때문에 hashtable보다 5~10배 메모리를 절약할 수 있습니다. 사용자 세션처럼 5~10개 필드를 가진 Hash는 listpack 인코딩이 유지되어 메모리 효율이 높습니다. `OBJECT ENCODING key` 명령으로 현재 인코딩을 확인할 수 있습니다.

> 출처: [How Redis Hashes Work Internally](https://oneuptime.com/blog/post/2026-03-31-redis-hashes-work-internally-hashtable-listpack/view) | [listpack spec (antirez)](https://github.com/antirez/listpack/blob/master/listpack.md) | [listpack migration issue](https://github.com/redis/redis/issues/8702)
