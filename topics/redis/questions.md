---
tags: [redis, 면접질문, distributed-lock, cache]
related: [redis/concepts]
---

# Redis 면접 질문

→ [[home]] | [[topics/redis/concepts]]

---

## Redis가 빠른 이유는 무엇인가요? 싱글 스레드인데 어떻게 고성능을 낼 수 있나요?

**난이도**: 기초

**핵심 키워드**: In-Memory, 싱글 스레드, Context Switch, I/O Multiplexing, 원자성

**모범 답변 방향**:
- **In-Memory**: 디스크 I/O 없이 RAM에서 직접 읽기/쓰기 → 마이크로초 단위 응답
- **싱글 스레드**: Context Switching 비용 없음, Lock/Deadlock 없음, 명령어 자체가 원자적
- **I/O Multiplexing**: epoll(Linux) 기반으로 단일 스레드가 수천 개 클라이언트 커넥션을 비동기 처리
- **단순한 자료구조**: 복잡한 조인 없이 O(1)~O(log N) 연산
- **결론**: 싱글 스레드 + In-Memory + 비동기 I/O 조합 → 단순 연산에서 초당 수십만 건 처리 가능

**싱글 스레드의 한계**:
- CPU 코어 1개만 사용 → CPU 병목 발생 시 스케일업이 아닌 Redis Cluster로 수평 확장
- `KEYS *`, `SMEMBERS`(대형 Set) 등 O(N) 명령어 → 다른 요청 전체 블로킹 → 운영 환경 금지
- 대안: `SCAN`, `SSCAN`(cursor 기반 분할 조회)

**꼬리 질문 예시**:
- Redis 6.0+에서 I/O Thread가 추가됐는데 여전히 싱글 스레드라고 할 수 있나요? → 명령어 처리(Command Execution)는 여전히 싱글 스레드. I/O Thread는 네트워크 읽기/쓰기만 담당 → 핵심 처리는 싱글 스레드 보장
- KEYS * 를 운영 환경에서 쓰면 안 되는 이유는?

> 출처: https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/

---

## Redis의 영속성(Persistence) 옵션을 비교해주세요. 각각 언제 사용하나요?

**난이도**: 중급

**핵심 키워드**: RDB, AOF, No Persistence, BGSAVE, fsync, 데이터 유실

**모범 답변 방향**:

| 방식 | 원리 | 장점 | 단점 |
|---|---|---|---|
| **No Persistence** | 메모리만 사용 | 최고 성능 | 재시작 시 전체 유실 |
| **RDB (Snapshot)** | 주기적으로 fork() + BGSAVE → 바이너리 스냅샷 파일 | 빠른 재시작, 백업 편리 | 주기 사이 데이터 유실 가능 |
| **AOF (Append Only File)** | 모든 쓰기 명령을 로그로 기록 | 유실 최소화 (최대 1초) | 파일 크기 증가, 재시작 느림 |
| **RDB + AOF** | 둘 다 활성화 | 재시작 시 AOF로 복구 (더 완전) | 성능 오버헤드 |

**AOF fsync 정책**:
- `always`: 매 명령마다 fsync → 가장 안전, 성능 최저
- `everysec`: 1초마다 fsync → **권장** (최대 1초 유실)
- `no`: OS에 위임 → 성능 최고, 유실 범위 불확실

**선택 기준**:
- 캐시 전용(유실 허용): No Persistence 또는 RDB만
- 세션 저장소(유실 최소화): AOF everysec
- 금융/결제 데이터: RDB + AOF 조합

**꼬리 질문 예시**:
- RDB 저장 중 서버가 크래시되면 어떻게 되나요? → fork()된 자식 프로세스가 임시 파일에 쓰다가 죽으면 이전 스냅샷이 유지됨 → 마지막 성공 스냅샷으로 복구
- AOF 파일이 너무 커지면 어떻게 처리하나요? → AOF Rewrite(BGREWRITEAOF): 현재 메모리 상태를 최소 명령어 셋으로 재기록

> 출처: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/

---

## Redis의 메모리 관리와 Eviction 정책을 설명해주세요.

**난이도**: 중급

**핵심 키워드**: maxmemory, Eviction Policy, LRU, LFU, allkeys, volatile, TTL

**모범 답변 방향**:
- `maxmemory` 도달 시 Redis는 설정된 Eviction 정책에 따라 키 삭제
- TTL 만료된 키: Lazy(접근 시 삭제) + Active(주기적 샘플링으로 삭제) 병행

**주요 Eviction 정책**:

| 정책 | 대상 | 기준 |
|---|---|---|
| `noeviction` | - | 메모리 초과 시 쓰기 오류 반환 |
| `allkeys-lru` | 전체 키 | 가장 오래 전에 접근된 키 제거 |
| `allkeys-lfu` | 전체 키 | 가장 적게 접근된 키 제거 |
| `volatile-lru` | TTL 있는 키만 | 가장 오래 전에 접근된 키 제거 |
| `volatile-ttl` | TTL 있는 키만 | TTL 짧은 순서로 제거 |
| `allkeys-random` | 전체 키 | 무작위 제거 |

**실무 선택 기준**:
- 캐시 전용: `allkeys-lru` 또는 `allkeys-lfu` (접근 빈도 기반이면 lfu 우세)
- 세션 등 TTL 있는 데이터만: `volatile-lru`
- 절대 삭제 안 되어야 하는 데이터 포함: `noeviction` + 모니터링

**꼬리 질문 예시**:
- LRU와 LFU의 차이는? → LRU: 최근성(접근 시점), LFU: 빈도(접근 횟수). LFU가 핫 데이터 보호에 더 적합
- Redis의 LRU는 정확한 LRU가 아닌데 왜 그런가요? → 성능을 위해 근사 LRU 사용: 랜덤 샘플링(기본 5개)에서 가장 오래된 키 제거 → `maxmemory-samples` 늘리면 정확도 향상

---

## Redis Cache Hit/Miss 비율을 어떻게 관리하고, Cache Miss 시 발생하는 문제를 어떻게 대응하나요?

**난이도**: 중급

**핵심 키워드**: Cache Hit Ratio, Cache Miss, Cache Aside, Cache Penetration, Cache Avalanche, Warm-up

**모범 답변 방향**:

**Cache Hit Ratio 모니터링**:
- `INFO stats` 명령어: `keyspace_hits`, `keyspace_misses`로 계산
- Hit Ratio = `keyspace_hits / (keyspace_hits + keyspace_misses)`
- 일반적으로 80% 이상 유지가 목표

**Cache Miss 유형별 대응**:

| 문제 | 설명 | 대응 |
|---|---|---|
| **Cache Penetration** | 존재하지 않는 키 반복 조회 → 항상 DB까지 도달 | Null 캐싱 (짧은 TTL로 "없음" 캐싱), Bloom Filter |
| **Cache Avalanche** | 다수 캐시 동시 만료 → DB 부하 폭증 | TTL Jitter, 사전 워밍(Cache Warm-up) |
| **Cache Stampede** | 동시 Miss → 동시 DB 조회 | Mutex Lock, Probabilistic Early Expiration |

**Cache Warm-up 전략**:
- 배포/재시작 후 주요 데이터 선제 로딩 → Cold Start 방지
- 트래픽 낮은 시간대에 사전 로딩

**꼬리 질문 예시**:
- Null 캐싱의 문제점은? → 실제로 데이터가 나중에 생기면 TTL 만료까지 stale "없음" 상태 서빙 → 짧은 TTL + 데이터 생성 시 캐시 무효화 병행 필요
- Bloom Filter를 캐시와 함께 쓰는 이유는? → 존재하지 않는 키 조회를 Redis/DB 접근 전에 필터링 → Cache Penetration 원천 차단

---

## Java 환경에서 Lettuce와 Redisson 중 어떤 클라이언트를 선택해야 하나요?

**난이도**: 중급

**핵심 키워드**: Lettuce, Redisson, 커넥션 풀, 분산락, Reactive, Pub/Sub

**모범 답변 방향**:

| 항목 | Lettuce | Redisson |
|---|---|---|
| 기반 | Netty 비동기 I/O | Netty 비동기 I/O |
| 커넥션 모델 | 기본 단일 커넥션 공유 (스레드 안전) | 커넥션 풀 |
| Reactive 지원 | ✅ (Reactor/RxJava) | 제한적 |
| 분산락 | 직접 구현 필요 | ✅ RLock, RFairLock 내장 |
| 고수준 추상화 | 낮음 (명령어 레벨) | 높음 (RMap, RQueue 등) |
| Spring Boot | spring-data-redis 기본 클라이언트 | 별도 의존성 추가 |

**선택 기준**:
- **Lettuce 선택**: Spring Boot 기본 사용, Reactive WebFlux, 단순 캐싱/조회 → 경량성 우선
- **Redisson 선택**: 분산락 필요 (Watchdog 내장), 고수준 자료구조 (분산 컬렉션), RateLimiter 필요

**꼬리 질문 예시**:
- Lettuce가 단일 커넥션을 공유해도 스레드 안전한 이유는? → Netty Event Loop 기반으로 명령어를 직렬화하여 처리 → 명령어 자체는 순차 처리
- Redisson의 Watchdog이 자동으로 TTL을 연장하는 원리는? → 락 획득 시 별도 스케줄러 실행 → TTL의 1/3 주기마다 연장 요청 → 클라이언트 프로세스 종료 시 자동 중단

> 출처: https://redis.io/docs/latest/develop/clients/

---

## Redis Cluster의 구조와 정족수(Quorum) 기반 장애 허용을 설명해주세요.

**난이도**: 심화

**핵심 키워드**: 16384 슬롯, 해시 슬롯, 정족수, Failover, cluster-require-full-coverage, Sentinel vs Cluster

**모범 답변 방향**:

**슬롯 기반 데이터 분산**:
- 전체 키를 **16384개 해시 슬롯**으로 분산
- 키 → `CRC16(key) % 16384` → 해당 슬롯을 담당하는 노드로 라우팅
- 각 Master 노드가 슬롯 범위를 나눠 담당 (예: 3노드 → 각 약 5461슬롯)

**Failover와 정족수**:
- 각 Master는 최소 1개 Slave(Replica)를 가짐
- Master 장애 시: Slave들 중 **과반수(N/2+1) 이상의 Master 투표**를 받아야 새 Master 승격
- 3 Master 클러스터: 2개 투표 필요 → Master 1개 장애 허용
- 투표 불성립(네트워크 파티션으로 과반수 연결 불가) → Failover 보류

**`cluster-require-full-coverage` 설정**:
- `yes`(기본): 슬롯 미커버 상태 → 전체 클러스터 쓰기 거부 → 데이터 안정성 우선
- `no`: 정상 슬롯에 한해 계속 서비스 → 가용성 우선

**Cluster vs Sentinel 비교**:
| | Redis Sentinel | Redis Cluster |
|---|---|---|
| 목적 | 고가용성(HA) | 수평 확장 + HA |
| 데이터 분산 | 단일 데이터셋 | 슬롯 기반 분산 |
| Failover | Sentinel이 모니터링 | 노드 간 자체 감지 |
| 적합한 상황 | 단일 대용량 데이터, 간단한 HA | 데이터 수평 분산 필요 |

**꼬리 질문 예시**:
- Redis Cluster에서 MGET을 여러 키에 사용할 때 주의점은? → 다른 슬롯의 키를 한 번에 조회하면 CROSSSLOT 에러 → Hash Tag `{user}:profile`, `{user}:session` 으로 같은 슬롯에 배치
- Redis Cluster 환경에서 Redlock(분산락)이 안전하지 않은 이유는? → Cluster는 슬롯 단위로 독립적 → Redlock이 가정하는 "N개 독립 노드 과반수 획득"이 Cluster 내에서는 보장되지 않음 → 독립 Redis 인스턴스 N개 필요

> 출처: https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/

---

## Redis 기본 자료구조 5가지를 각각 언제 사용하나요?

**난이도**: 기초

**핵심 키워드**: String, List, Hash, Set, Sorted Set, ziplist, TTL, LPUSH/RPOP

**모범 답변 방향**:
- **String**: 단순 Key-Value, 최대 512MB, binary 가능. 캐싱, 카운터(INCR), 세션 토큰
- **List**: LPUSH/RPOP(큐), LPUSH/LPOP(스택). 작업 큐, 최근 N개 로그 저장
- **Hash**: 하나의 키에 여러 필드 저장. 사용자 프로필(`user:1` → `{name, age, email}`). ziplist 압축으로 메모리 효율
- **Set**: 중복 없는 집합. 태그, 좋아요 목록, 교집합/합집합 연산
- **Sorted Set**: score 기준 정렬. 랭킹, 리더보드, 시간 순 이벤트 로그

**Hash vs String 다중 키 선택 기준**:
- Hash: 관련 데이터 묶음, 메모리 효율 (128필드 이하 ziplist 인코딩)
- String 다중 키: **개별 필드 TTL이 필요할 때** (Hash는 키 전체에만 TTL 적용 가능)

**꼬리 질문 예시**:
- Hash와 String 여러 개를 쓸 때의 차이 및 선택 기준은?
- Sorted Set의 시간 복잡도는? (ZADD: O(log N), ZRANGE: O(log N + M))

**면접 세션 피드백 (2026-04-01)**:
- 잘한 점: 5가지 용도 기본 정의, Sorted Set 랭킹 예시
- 보완: Hash vs String 선택 기준(메모리 효율, TTL 제한) 필수 추가

---

## Redis 분산락을 어떻게 구현하나요? SETNX와 Redlock의 차이도 설명해주세요.

**난이도**: 심화

**핵심 키워드**: SETNX, NX PX, Lua 스크립트, Redlock, 과반수, TTL

**모범 답변 방향**:
- 분산락이 필요한 시나리오 먼저 제시 (재고 차감, 중복 결제 방지 등)
- `SET key value NX PX {ttl}` 원자적 획득
- 락 해제는 Lua 스크립트로 확인+삭제 원자 처리 (단순 DEL 금지)
- SETNX 단점: 단일 노드 장애 시 락 유실
- Redlock: N개 노드 과반수 획득으로 단일 장애 대응
- 선택 기준: 고가용성 보장 여부에 따라 SETNX vs Redlock

**꼬리 질문 예시**:
- TTL이 만료되기 전에 작업이 끝나지 않으면 어떻게 되나요? (락 갱신/watchdog 패턴)
- 락을 해제할 때 단순히 DEL을 쓰면 안 되는 이유는? → GET 후 DEL 사이 race condition → Lua 스크립트로 원자적 처리 필요
- Redis Cluster 환경에서 SETNX를 쓰면 안 되는 케이스가 있나요?
- Lua 스크립트를 쓰지 않고 GET + DEL을 따로 실행하면 어떤 문제가 생기나요?

**면접 세션 피드백 (2026-03-28)**:
- 잘한 점: SET NX PX 원자성 + 단일 스레드 이유, DEL 위험 시나리오 설명
- 보완: Lua 스크립트 코드 수준으로 암기, TTL watchdog 패턴 숙지

> 출처: Redis 공식 문서 - https://redis.io/docs/manual/patterns/distributed-locks/

---

## Redis MULTI/EXEC 트랜잭션이란 무엇이고, RDBMS 트랜잭션과 어떻게 다른가요?

**난이도**: 중급

**핵심 키워드**: MULTI, EXEC, QUEUED, 롤백 없음, 격리, DISCARD

**모범 답변 방향**:
- MULTI로 시작 → 명령들이 QUEUED로 쌓임 → EXEC로 일괄 실행 (격리 보장)
- RDBMS와 핵심 차이: **롤백 없음** — 중간 명령 실패해도 나머지 명령은 계속 실행됨
- 원자성은 보장하지만 일관성 보장은 애플리케이션 책임
- DISCARD로 트랜잭션 취소 가능

**꼬리 질문 예시**:
- MULTI/EXEC에서 명령 하나가 실패하면 어떻게 되나요?
- WATCH를 함께 쓰는 이유는 무엇인가요?

> 출처: https://redis.io/docs/latest/develop/using-commands/transactions/

---

## go-redis에서 Pipeline과 TxPipeline의 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: Pipeline, TxPipeline, MULTI/EXEC 래핑, 원자성, 네트워크 왕복

**모범 답변 방향**:
- Pipeline: 여러 명령을 한 번의 네트워크 왕복으로 전송 → 네트워크 최적화. **원자성 없음**
- TxPipeline: 내부적으로 MULTI + 명령들 + EXEC를 묶어 전송 → **원자성 보장**
- TxPipeline 단독 사용보다 `rdb.Watch() + tx.TxPipelined()`로 낙관적 잠금 구현할 때 주로 사용
- 실패 시 `redis.TxFailedErr` 반환 → 애플리케이션에서 재시도 처리

**꼬리 질문 예시**:
- TxPipeline을 단독으로 쓰는 것과 Watch와 함께 쓰는 것의 차이는?
- 낙관적 잠금이 비관적 잠금(분산락)보다 적합한 상황은 언제인가요?

> 출처: https://redis.uptrace.dev/guide/go-redis-pipelines.html

---

## MULTI/EXEC와 Lua 스크립트의 차이점과 선택 기준은?

**난이도**: 심화

**핵심 키워드**: 조건부 로직, 원자성, 네트워크 효율, 분산락 해제, Redis Cluster

**모범 답변 방향**:
- 둘 다 원자성 보장, 롤백 없음
- MULTI/EXEC: 단순 다중 명령 묶음, 조건부 처리 불가, Redis Cluster에서 Cross-slot 불가
- Lua: 서버에서 실행 → 네트워크 왕복 1회, if/else 같은 조건부 로직 가능, Cluster에서 KEYS 인자로 라우팅
- 선택 기준: 단순 묶음 → MULTI/EXEC / 조건부 처리(GET 값 보고 분기) → Lua
- 분산락 해제(GET 확인 + DEL 삭제 원자 처리)가 Lua를 써야 하는 대표 사례
- 재고 차감처럼 "현재값 확인 후 조건부 갱신"도 Lua 적합

**꼬리 질문 예시**:
- 분산락 해제를 MULTI/EXEC로 구현할 수 없는 이유는? → WATCH + MULTI/EXEC로 가능하지만 GET 결과를 EXEC 전에 읽어야 해서 흐름이 복잡. 조건 확인 후 분기가 불가능
- Lua 스크립트의 단점은 무엇인가요? → 디버깅 어려움(서버 사이드 실행), 스크립트 길어지면 가독성 저하, EVALSHA 캐시 휘발성
- Redis Cluster에서 MULTI/EXEC를 쓰면 안 되는 이유는? → 다른 슬롯의 키를 한 트랜잭션으로 묶으면 Cross-slot 에러

> 출처: https://dgle.dev/redis-multi-lua/

---

## Redis Lua 스크립트에서 EVAL과 EVALSHA의 차이는? 언제 EVALSHA를 사용해야 하나요?

**난이도**: 심화

**핵심 키워드**: EVAL, EVALSHA, SCRIPT LOAD, SHA1, 스크립트 캐싱, NOSCRIPT 에러

**모범 답변 방향**:
- EVAL: 매 호출마다 전체 스크립트를 서버에 전송 → 네트워크 오버헤드
- SCRIPT LOAD: 스크립트를 서버에 캐싱 후 SHA1 해시 반환
- EVALSHA: SHA1만 전송 → 스크립트 반복 호출 시 네트워크 효율적
- 주의: 스크립트 캐시는 휘발성 (재시작, SCRIPT FLUSH, 페일오버 시 소멸)
- NOSCRIPT 에러 발생 시 EVAL로 폴백하는 방어 코드 필수

**꼬리 질문 예시**:
- EVALSHA 사용 중 서버가 재시작되면 어떻게 되나요? → NOSCRIPT 에러 → EVAL로 재로드 필요
- 파이프라인에서 EVALSHA를 쓰면 안 되는 이유는? → 파이프라인 중 NOSCRIPT 에러가 발생해도 처리가 어려움 → EVAL 권장

> 출처: https://redis.io/docs/latest/develop/programmability/eval-intro/

---

## redis.call()과 redis.pcall()의 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: redis.call, redis.pcall, 에러 처리, 스크립트 중단

**모범 답변 방향**:
- `redis.call()`: 에러 발생 시 즉시 클라이언트에 에러 전파 + 스크립트 중단
- `redis.pcall()`: 에러를 테이블로 반환 → 스크립트 내에서 catch하여 처리 가능
- 선택 기준: 에러 발생 시 그냥 실패해도 되면 call(), 에러에 따라 다른 로직 실행 필요하면 pcall()

**꼬리 질문 예시**:
- Lua 스크립트에서 롤백이 안 된다면 부분 실행 중 에러 처리는 어떻게 하나요?

> 출처: https://redis.io/docs/latest/develop/programmability/eval-intro/

---

## 인기 상품 캐시 만료 시 Cache Stampede 대응 설계

**난이도**: 심화

**핵심 키워드**: Probabilistic Early Expiration, Mutex Lock, Stale-While-Revalidate, TTL Jitter, Lock TTL + Watchdog

**모범 답변 방향**:

**3가지 이상 전략 및 트레이드오프**:

| 전략 | 원리 | 일관성 | 복잡도 | 메모리 |
|---|---|---|---|---|
| **TTL Jitter** | TTL에 랜덤 오프셋 추가 → 만료 시점 분산 | 높음 | 낮음 | 동일 |
| **Mutex Lock (분산락)** | 첫 요청만 DB 조회, 나머지는 Lock 대기 | 높음 | 중간 | 동일 |
| **Probabilistic Early Expiration** | TTL 만료 전 확률적으로 선제 갱신 | 높음 | 중간 | 동일 |
| **Stale-While-Revalidate** | 만료된 캐시라도 즉시 반환 + 백그라운드 갱신 | 낮음(일시적 stale) | 낮음 | 동일 |
| **로컬 캐시 (L1)** | 앱 서버 메모리에 1차 캐시 → Redis 부하 감소 | 낮음(노드별 차이) | 높음 | 증가 |

**Mutex Lock에서 소유자 장애 대응**:
- Lock에 TTL 설정 필수 (예: SET lock NX EX 5) → 소유자 장애 시 TTL 만료 후 자동 해제
- 처리 시간이 TTL을 초과할 수 있는 경우 → **Watchdog 패턴**: 별도 스레드가 주기적으로 Lock TTL 연장
- Redisson 라이브러리: Watchdog 내장 구현 (기본 30초, 1/3 주기로 연장)
- 주의: Watchdog도 클라이언트 프로세스 죽으면 연장 중단 → TTL 만료 후 해제됨

**꼬리 질문 예시**:
- Redis 싱글 스레드 특성이 Mutex Lock 구현에서 어떤 이점을 주나요?
- Stale-While-Revalidate 방식에서 갱신 실패 시 stale 데이터가 계속 서빙될 수 있는데 어떻게 처리하나요?
- 분산 환경에서 Redis Cluster를 쓸 때 분산락이 안전하지 않은 이유와 Redlock 알고리즘을 설명해주세요.

---
