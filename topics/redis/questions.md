---
tags: [redis, 면접질문, distributed-lock, cache]
related: [redis/concepts, distributed-systems, system-design, kafka]
---

# Redis 면접 질문

→ [[home]] | [[topics/redis/concepts]]

---

## Redis가 빠른 이유는 무엇인가요? 싱글 스레드인데 어떻게 고성능을 낼 수 있나요?

**난이도**: 기초

**핵심 키워드**: In-Memory, 싱글 스레드, Context Switch, I/O Multiplexing, 원자성

**모범 답변 (3분 이상 말하기 형태)**:

> Redis가 빠른 근본 이유는 세 가지가 조합된 결과입니다. 첫째, 모든 데이터를 RAM에서 직접 읽고 쓰는 In-Memory 구조이기 때문에 디스크 I/O가 전혀 없고 응답 시간이 마이크로초 단위입니다. 둘째, 싱글 스레드 모델을 채택하고 있어 스레드 간 Context Switching 비용이 없고, Lock이나 Deadlock 문제가 구조적으로 발생하지 않으며, 모든 명령어가 그 자체로 원자적으로 처리됩니다. 셋째, I/O Multiplexing, 리눅스 환경에서는 epoll 기반의 비동기 이벤트 루프를 사용해 단일 스레드가 수천 개의 클라이언트 커넥션을 동시에 처리할 수 있습니다. 여기에 String, List, Hash 등 단순한 자료구조 덕분에 대부분의 연산이 O(1)에서 O(log N) 수준으로 마무리됩니다. 이 세 가지 조합으로 단순 연산 기준 초당 수십만 건 처리가 가능합니다.
>
> 다만 싱글 스레드이기 때문에 CPU 코어 하나만 사용한다는 한계가 있습니다. CPU 병목이 발생하면 스케일업이 아닌 Redis Cluster로 수평 확장해야 합니다. 또한 `KEYS *`나 대형 Set의 `SMEMBERS` 같은 O(N) 명령어는 그 시간 동안 다른 모든 요청을 블로킹하기 때문에 운영 환경에서는 절대 사용해서는 안 됩니다. 대신 `SCAN`, `SSCAN` 같은 커서 기반 분할 조회 명령어로 대체해야 합니다. Redis 6.0 이후에는 I/O Thread가 추가되어 네트워크 읽기/쓰기는 멀티 스레드로 처리되지만, 명령어 실행 자체는 여전히 싱글 스레드로 처리되기 때문에 핵심 원자성은 유지됩니다.

**꼬리 질문 예시**:
- Redis 6.0+에서 I/O Thread가 추가됐는데 여전히 싱글 스레드라고 할 수 있나요? → 명령어 처리(Command Execution)는 여전히 싱글 스레드. I/O Thread는 네트워크 읽기/쓰기만 담당 → 핵심 처리는 싱글 스레드 보장
- KEYS * 를 운영 환경에서 쓰면 안 되는 이유는?

> 출처: https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/

---

## Redis의 영속성(Persistence) 옵션을 비교해주세요. 각각 언제 사용하나요?

**난이도**: 중급

**핵심 키워드**: RDB, AOF, No Persistence, BGSAVE, fsync, 데이터 유실

**모범 답변 (3분 이상 말하기 형태)**:

> Redis의 영속성 옵션은 No Persistence, RDB, AOF, 그리고 RDB와 AOF를 동시에 사용하는 방식 네 가지로 나뉩니다. 선택 기준은 데이터 유실 허용 범위와 재시작 복구 속도 사이의 트레이드오프에서 결정됩니다.
>
> No Persistence는 데이터를 메모리에만 저장하기 때문에 성능이 가장 높지만, 재시작 시 전체 데이터가 유실됩니다. 캐시 전용으로 유실이 허용되는 경우에 적합합니다. RDB는 주기적으로 fork()를 수행하고 BGSAVE 명령으로 바이너리 스냅샷 파일을 저장하는 방식입니다. 재시작 시 스냅샷을 한꺼번에 로드하기 때문에 복구 속도가 빠르고 백업이 편리합니다. 단, 마지막 스냅샷 이후 변경된 데이터는 유실될 수 있습니다. 주기 사이에 장애가 나면 그 사이 쓰기가 모두 손실됩니다.
>
> AOF는 모든 쓰기 명령을 로그 파일에 순차적으로 기록하는 방식입니다. fsync 정책으로 동작 방식을 조정할 수 있는데, `always`로 설정하면 매 명령마다 fsync하여 가장 안전하지만 성능이 가장 낮습니다. `everysec`는 1초마다 fsync하는 방식으로 최대 1초치만 유실되고 성능 오버헤드도 적어 실무에서 권장합니다. `no`는 OS에 위임하기 때문에 성능은 가장 높지만 유실 범위를 예측할 수 없습니다. AOF는 시간이 지날수록 파일 크기가 커지는 단점이 있는데, `BGREWRITEAOF` 명령으로 현재 메모리 상태를 최소 명령어 셋으로 재기록하는 AOF Rewrite로 파일 크기를 줄일 수 있습니다.
>
> 가장 완전한 복구를 원한다면 RDB와 AOF를 동시에 활성화합니다. 재시작 시 더 완전한 복구가 가능한 AOF를 우선으로 사용하고, RDB는 주기적 백업용으로 활용합니다. 사용 목적별 권장 설정을 정리하면, 캐시 전용이라면 No Persistence 또는 RDB, 세션 저장소처럼 유실을 최소화해야 한다면 AOF everysec, 결제나 금융 데이터처럼 절대 유실이 허용되지 않는다면 RDB와 AOF를 함께 활성화하는 방식입니다.

**꼬리 질문 예시**:
- RDB 저장 중 서버가 크래시되면 어떻게 되나요? → fork()된 자식 프로세스가 임시 파일에 쓰다가 죽으면 이전 스냅샷이 유지됨 → 마지막 성공 스냅샷으로 복구
- AOF 파일이 너무 커지면 어떻게 처리하나요? → AOF Rewrite(BGREWRITEAOF): 현재 메모리 상태를 최소 명령어 셋으로 재기록

> 출처: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/

---

## Redis의 메모리 관리와 Eviction 정책을 설명해주세요.

**난이도**: 중급

**핵심 키워드**: maxmemory, Eviction Policy, LRU, LFU, allkeys, volatile, TTL

**모범 답변 (3분 이상 말하기 형태)**:

> Redis는 `maxmemory` 한계에 도달하면 설정된 Eviction 정책에 따라 키를 자동으로 삭제합니다. 먼저 TTL이 만료된 키의 처리 방식을 이해해야 합니다. Redis는 만료된 키를 두 가지 방식으로 제거하는데, 해당 키에 접근하는 시점에 지연 삭제하는 Lazy 방식과 주기적으로 랜덤 샘플링을 해서 능동적으로 제거하는 Active 방식을 병행합니다.
>
> Eviction 정책은 크게 전체 키를 대상으로 하는 `allkeys` 계열과 TTL이 설정된 키만 대상으로 하는 `volatile` 계열, 그리고 `noeviction`으로 나뉩니다. `noeviction`은 메모리 초과 시 쓰기 요청에 오류를 반환하는 방식으로, 캐시가 아닌 원본 데이터를 저장하는 용도에 적합합니다. `allkeys-lru`는 전체 키 중 가장 오래 전에 접근된 키를 제거하고, `allkeys-lfu`는 접근 빈도가 가장 낮은 키를 제거합니다. `volatile-lru`는 TTL이 설정된 키만 대상으로 LRU를 적용하고, `volatile-ttl`은 TTL이 가장 짧게 남은 키를 우선 제거합니다.
>
> 선택 기준은 사용 목적에 따라 달라집니다. 캐시 전용 Redis라면 `allkeys-lru` 또는 `allkeys-lfu`가 적합합니다. 같은 캐시라도 접근 빈도 기반으로 핫 데이터를 보호해야 한다면 LFU가 더 나은 선택입니다. LRU는 최근 한 번 접근한 키를 오래 유지할 수 있지만, LFU는 오랜 기간 자주 접근된 키를 더 강하게 보호하기 때문입니다. TTL이 설정된 데이터만 관리하고 나머지는 보존해야 한다면 `volatile-lru`가 안전하고, 절대 삭제되어서는 안 되는 데이터가 섞인 경우에는 `noeviction`으로 설정하고 모니터링을 강화하는 방식을 선택합니다.
>
> 한 가지 중요한 점은 Redis의 LRU가 정확한 LRU가 아니라 근사 LRU라는 것입니다. 모든 키에 대한 정확한 접근 시간을 추적하면 메모리 오버헤드가 크기 때문에, Redis는 랜덤으로 샘플링한 키들 중 가장 오래된 키를 제거하는 방식을 씁니다. `maxmemory-samples` 값을 높이면 샘플 수가 늘어 정확도가 향상되지만 그만큼 CPU 비용도 증가합니다.

**꼬리 질문 예시**:
- LRU와 LFU의 차이는? → LRU: 최근성(접근 시점), LFU: 빈도(접근 횟수). LFU가 핫 데이터 보호에 더 적합
- Redis의 LRU는 정확한 LRU가 아닌데 왜 그런가요? → 성능을 위해 근사 LRU 사용: 랜덤 샘플링(기본 5개)에서 가장 오래된 키 제거 → `maxmemory-samples` 늘리면 정확도 향상

---

## Cache-Aside 패턴과 write-around 캐시 무효화 순서

**난이도**: 기초~중급

**핵심 키워드**: Cache-Aside, write-through, write-around, TTL 안전망, invalidation, race condition, DB 업데이트 순서

**Cache-Aside 동작:**
- 캐시 miss → DB에서 읽기 → 애플리케이션이 캐시에 쓰기
- 정합성 문제 발생 상황: ①Redis 쓰기 실패, ②DB 읽은 후 DB 값 변경
- **TTL의 역할 2가지**:
  1. 데이터 신선도 조정 (일반적 인식)
  2. **invalidation 완전 실패 시 최후 안전망** — 재시도도 실패하면 TTL 없이는 stale 데이터 무기한 유지

**캐싱 쓰기 전략 비교:**

| 전략 | 동작 | 장점 | 단점 |
|---|---|---|---|
| **write-through** | 쓰기 시 캐시+DB 동기 업데이트 | 항상 캐시 최신 보장 | write amplification (읽지 않을 데이터도 캐시에 쌓임) |
| **write-around** | DB 먼저 업데이트 → 캐시 invalidate | 불필요한 캐시 저장 방지 | invalidation 실패 시 stale 잔류 |
| **write-behind** | 캐시에 먼저 쓰고 비동기로 DB 동기화 | 쓰기 성능 최고 | DB 반영 전 장애 시 유실 위험 |

**invalidate 순서별 race condition:**

```
❌ 캐시 먼저 삭제 → DB 업데이트:
  T1: 캐시 삭제
  T2: 캐시 miss → DB에서 구버전 읽기 → 구버전을 캐시에 쓰기
  T1: DB 업데이트 완료
  결과: 캐시에 구버전 stale 데이터 잔류

✅ DB 먼저 업데이트 → 캐시 삭제 (권장):
  T1: DB 업데이트
  T2: 캐시 hit → 잠깐 구버전 읽음 (허용 가능한 짧은 window)
  T1: 캐시 삭제 완료
  결과: 이후 요청은 DB에서 최신 데이터 읽어 캐시 갱신
```

**왜 DB 먼저가 안전한가:**
- DB가 원본(source of truth) → 원본 먼저 갱신
- 캐시 삭제 실패 → 재시도 로직으로 대응
- 재시도도 실패 → TTL 만료 시 자동 갱신 (이중 안전망)

**모범 마무리 문장:**
> "DB 업데이트 → 캐시 삭제 순서를 지키고, 삭제 실패에 대비해 재시도 로직과 TTL을 이중 안전망으로 둡니다. TTL이 없으면 invalidation이 영구 실패할 경우 stale 데이터가 무기한 남습니다."

**꼬리 질문 예시:**
- "캐시 먼저 삭제하는 방식을 선택하는 경우는 언제인가요?" → 구버전 stale 보다 일시적 cache miss가 더 허용될 때. 단, 두 번 삭제(삭제 → DB 업데이트 → 재삭제) 패턴으로 race 최소화 가능
- "분산락으로 해결하면 안 되나요?" → 가능하지만 매 읽기에 락을 걸면 비용이 큼. DB first + TTL 조합이 일반적으로 충분

**면접 세션 피드백 (2026-04-10 1회차)**:
- 잘한 점: Cache-Aside 개념, write-through/write-around 구분, 두 race condition 시나리오 정확. DB 먼저 이유와 재시도 대응 제시.
- 보완: TTL을 "신선도 조정"으로만 설명 → "invalidation 실패 시 최후 안전망" 역할 추가 필요

**면접 세션 피드백 (2026-04-16 3회차)**:
- 잘한 점: 세 가지 패턴(Cache-Aside/Write-Through/Write-Behind) 동작 및 사용 시나리오 정확. Write-Behind 리스크(데이터 유실) 즉시 파악. Kafka append-only + 멱등 처리 + 배치 발행 완화 방안 실무적 제시.
- 보완:
  - Cache-Aside의 Cache Stampede 미언급 → TTL Jitter / 분산락 완화 방법 세트로 암기
  - Write-Behind 완화: Redis AOF `appendfsync everysec` 옵션 추가 (Kafka 없는 환경 대안)
  - Kafka 멱등 설정 구체화: `enable.idempotence=true` + `acks=all` 명시

---

## Redis Cache Hit/Miss 비율을 어떻게 관리하고, Cache Miss 시 발생하는 문제를 어떻게 대응하나요?

**난이도**: 중급

**핵심 키워드**: Cache Hit Ratio, Cache Miss, Cache Aside, Cache Penetration, Cache Avalanche, Warm-up

**모범 답변 (3분 이상 말하기 형태)**:

> Cache Hit Ratio는 `INFO stats` 명령어로 조회할 수 있는 `keyspace_hits`와 `keyspace_misses` 값으로 계산합니다. hits를 hits와 misses의 합으로 나눈 비율이고, 일반적으로 80% 이상을 유지하는 것이 목표입니다. Hit Ratio가 낮다면 TTL이 너무 짧거나 캐시 용량이 부족하거나, 혹은 캐시에 적합하지 않은 데이터를 캐싱하고 있는 경우입니다.
>
> Cache Miss가 발생할 때 대표적으로 세 가지 문제가 생깁니다. 첫 번째는 Cache Penetration입니다. 존재하지 않는 키를 반복적으로 조회하면 항상 DB까지 도달하게 되어 부하가 누적됩니다. 예를 들어 악의적인 요청이나 잘못된 ID로 반복 조회하는 경우가 해당합니다. 이 경우 "없음" 상태 자체를 짧은 TTL로 캐싱하는 Null 캐싱이나, 존재하지 않는 키를 Redis와 DB 접근 전에 필터링하는 Bloom Filter로 대응합니다. 단, Null 캐싱은 나중에 데이터가 실제로 생겨도 TTL 만료까지 stale "없음" 상태를 서빙할 수 있어 데이터 생성 시 캐시 무효화도 병행해야 합니다.
>
> 두 번째는 Cache Avalanche입니다. 다수의 캐시가 동시에 만료되어 DB에 부하가 한꺼번에 몰리는 문제입니다. TTL에 랜덤 오프셋을 추가하는 TTL Jitter로 만료 시점을 분산시키거나, 배포 전 주요 데이터를 선제 로딩하는 Cache Warm-up 전략으로 완화합니다.
>
> 세 번째는 Cache Stampede입니다. 동일한 인기 키에 대해 동시 Miss가 발생해 수십 개의 요청이 동시에 DB를 조회하는 상황입니다. Mutex Lock으로 첫 번째 요청만 DB를 조회하게 직렬화하고 나머지는 락 해제 후 캐시에서 읽는 방식이나, TTL 만료 전에 확률적으로 선제 갱신을 트리거하는 Probabilistic Early Expiration 방식으로 해결합니다.

**꼬리 질문 예시**:
- Null 캐싱의 문제점은? → 실제로 데이터가 나중에 생기면 TTL 만료까지 stale "없음" 상태 서빙 → 짧은 TTL + 데이터 생성 시 캐시 무효화 병행 필요
- Bloom Filter를 캐시와 함께 쓰는 이유는? → 존재하지 않는 키 조회를 Redis/DB 접근 전에 필터링 → Cache Penetration 원천 차단

---

## Java 환경에서 Lettuce와 Redisson 중 어떤 클라이언트를 선택해야 하나요?

**난이도**: 중급

**핵심 키워드**: Lettuce, Redisson, 커넥션 풀, 분산락, Reactive, Pub/Sub

**모범 답변 (3분 이상 말하기 형태)**:

> Java 환경에서 Redis 클라이언트를 선택할 때는 사용 목적에 따라 Lettuce와 Redisson을 구분해야 합니다. 두 클라이언트 모두 Netty 비동기 I/O 기반이라는 공통점이 있지만, 커넥션 모델과 제공하는 추상화 수준이 다릅니다.
>
> Lettuce는 Spring Boot의 spring-data-redis에서 기본으로 사용하는 클라이언트입니다. Netty Event Loop 기반으로 명령어를 직렬화해 단일 커넥션을 스레드 안전하게 공유하는 방식으로 동작합니다. 커넥션 하나로 다수의 스레드가 공유하더라도 내부적으로 명령어가 순차 처리되기 때문에 안전합니다. Reactor나 RxJava 같은 Reactive 라이브러리와 연동이 잘 되어 WebFlux 환경이나 단순한 캐싱, 조회 용도에 경량으로 쓰기 좋습니다.
>
> 반면 Redisson은 분산락이 필요한 경우에 적합합니다. RLock, RFairLock 같은 고수준 락 추상화가 내장되어 있고, Watchdog 기능이 특히 중요합니다. 락을 획득하면 별도 스케줄러가 동작해 TTL의 1/3 주기마다 TTL 연장 요청을 자동으로 보냅니다. 락을 보유한 클라이언트가 살아있는 한 TTL이 계속 갱신되고, 프로세스가 죽으면 갱신이 멈춰 TTL이 자연스럽게 만료되어 락이 해제됩니다. 이 Watchdog 덕분에 처리 시간이 불확실한 작업에서도 락 타임아웃을 어떻게 잡아야 할지 고민을 크게 줄여줍니다. RateLimiter나 분산 컬렉션 같은 고수준 자료구조가 필요한 경우에도 Redisson이 더 적합합니다.
>
> 정리하면, 단순 캐싱이나 Reactive 스택이라면 Lettuce, 분산락이나 고수준 분산 자료구조가 필요하다면 Redisson을 선택합니다. 두 가지를 동시에 써야 하는 경우에는 Lettuce를 spring-data-redis 기본 클라이언트로 유지하고, 분산락 용도만 Redisson을 추가로 의존성에 포함하는 방식도 가능합니다.

**꼬리 질문 예시**:
- Lettuce가 단일 커넥션을 공유해도 스레드 안전한 이유는? → Netty Event Loop 기반으로 명령어를 직렬화하여 처리 → 명령어 자체는 순차 처리
- Redisson의 Watchdog이 자동으로 TTL을 연장하는 원리는? → 락 획득 시 별도 스케줄러 실행 → TTL의 1/3 주기마다 연장 요청 → 클라이언트 프로세스 종료 시 자동 중단

> 출처: https://redis.io/docs/latest/develop/clients/

---

## Redis Cluster의 구조와 정족수(Quorum) 기반 장애 허용을 설명해주세요.

**난이도**: 심화

**핵심 키워드**: 16384 슬롯, 해시 슬롯, 정족수, Failover, cluster-require-full-coverage, Sentinel vs Cluster

**모범 답변 (3분 이상 말하기 형태)**:

> Redis Sentinel과 Redis Cluster는 목적이 다릅니다. Sentinel은 단일 데이터셋의 고가용성을 위한 구성이고, Cluster는 수평 확장과 고가용성을 동시에 제공합니다. 데이터를 여러 노드에 분산해야 한다면 Cluster를 선택합니다.
>
> Redis Cluster는 전체 키를 16384개의 해시 슬롯으로 분산 관리합니다. 각 키는 `CRC16(key) % 16384` 연산으로 슬롯이 결정되고, 해당 슬롯을 담당하는 노드로 라우팅됩니다. 3개 마스터 구성이라면 각 노드가 약 5461개씩 슬롯을 나눠 갖습니다.
>
> Failover는 Quorum, 즉 정족수 방식으로 동작합니다. 각 Master 노드는 최소 하나의 Slave를 가지며, Master가 장애가 나면 해당 Slave들 중 과반수 이상의 Master 투표를 받아야 새 Master로 승격될 수 있습니다. 3 Master 클러스터라면 2개 투표가 필요하고 Master 1개 장애는 허용합니다. 네트워크 파티션으로 과반수와의 연결이 끊어지면 Failover가 보류됩니다.
>
> `cluster-require-full-coverage` 설정도 중요합니다. 기본값인 `yes`로 설정하면 커버되지 않는 슬롯이 생겼을 때 전체 클러스터의 쓰기를 거부해 데이터 안정성을 우선시합니다. `no`로 설정하면 정상 슬롯에 한해 계속 서비스해 가용성을 우선시합니다. 어떤 값이 맞는지는 서비스의 데이터 정합성 요구사항에 따라 다릅니다. 금융이나 결제처럼 정합성이 중요하다면 `yes`, 일부 슬롯 장애에도 나머지 기능이 계속 동작해야 한다면 `no`가 적합합니다.

**꼬리 질문 예시**:
- Redis Cluster에서 MGET을 여러 키에 사용할 때 주의점은? → 다른 슬롯의 키를 한 번에 조회하면 CROSSSLOT 에러 → Hash Tag `{user}:profile`, `{user}:session` 으로 같은 슬롯에 배치
- Redis Cluster 환경에서 Redlock(분산락)이 안전하지 않은 이유는? → Cluster는 슬롯 단위로 독립적 → Redlock이 가정하는 "N개 독립 노드 과반수 획득"이 Cluster 내에서는 보장되지 않음 → 독립 Redis 인스턴스 N개 필요

> 출처: https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/

---

## Redis Sorted Set(ZSet)의 시간 복잡도와 Set/List/ZSet 선택 기준은 무엇인가요?

**난이도**: 기초

**핵심 키워드**: ZADD O(log N), ZRANGE O(log N+M), score, 중복 불가, 정렬, 랭킹, 최근 본 상품

**모범 답변 (3분 이상 말하기 형태)**:

> Redis Sorted Set은 score를 기준으로 자동 정렬되는 자료구조입니다. ZADD는 skiplist에서 정렬 위치를 찾아 삽입하기 때문에 O(log N), ZRANGE와 ZRANGEBYSCORE는 O(log N + M)입니다. M은 반환되는 원소 수입니다. 실시간 랭킹처럼 순위 기반 범위 조회가 필요하고, 동일 유저가 중복 집계되면 안 되는 경우에 ZSet을 씁니다. Set은 순서가 없어 범위 조회가 불가능하고, List는 중복을 허용하기 때문에 중복 없이 정렬 상태가 필요한 경우에는 ZSet이 유일한 선택입니다. 수백만 건으로 커질 때는 시간 단위로 키를 분리해서 `ranking:daily:2026-04-13` 형태로 관리하고, `ZREMRANGEBYSCORE`로 만료 데이터를 주기적으로 정리합니다. 도메인별로 키를 분리하면 전체 랭킹을 합산할 때도 각 ZSet에서 top K를 꺼내 merge sort하는 방식으로 처리할 수 있습니다.

**꼬리 질문 예시**:
- ZADD 시간복잡도가 O(1)이 아닌 O(log N)인 이유는? (skiplist 삽입 시 정렬 위치 탐색 필요)
- 랭킹 데이터를 해시 기반 라운드로빈으로 샤딩하면 안 되는 이유는? (전체 상위 K 집계 시 모든 샤드 merge 필요)
- ZSet의 score가 동일할 때 정렬 기준은? (사전순 lexicographic)

**면접 세션 피드백 (2026-04-13 1회차)**:
- Set/List/ZSet 비교 표현 좋음: "중복 허용하지 않으면서 정렬이 필요할 때"
- ZADD를 O(1)로 잘못 답변 → O(log N) 교정 필요
- 샤딩 방법: 라운드로빈 ❌ → 시간/도메인 기반 키 분리 + ZREMRANGEBYSCORE ✅

---

## Redis 기본 자료구조 5가지를 각각 언제 사용하나요?

**난이도**: 기초

**핵심 키워드**: String, List, Hash, Set, Sorted Set, ziplist, TTL, LPUSH/RPOP

**모범 답변 (3분 이상 말하기 형태)**:

> Redis의 기본 자료구조 다섯 가지는 String, List, Hash, Set, Sorted Set입니다. String은 최대 512MB까지 저장 가능한 가장 범용적인 타입으로, 캐싱, INCR 명령어를 이용한 카운터, 세션 토큰 저장에 주로 씁니다. List는 양방향 연결 리스트로, LPUSH와 RPOP 조합으로 큐를, LPUSH와 LPOP 조합으로 스택을 구현할 수 있어 작업 큐나 최근 N개 로그 저장에 적합합니다. Hash는 하나의 키 아래 여러 필드를 저장할 수 있어 사용자 프로필처럼 `user:1` 키에 name, age, email을 묶어서 관리할 때 유용합니다. 128개 이하의 필드라면 ziplist 인코딩으로 압축 저장되어 메모리 효율이 높습니다. Set은 중복 없는 집합 자료구조로 태그 관리, 좋아요 목록, 교집합이나 합집합 연산이 필요한 경우에 씁니다. Sorted Set은 score 기준으로 정렬된 집합이어서 랭킹 시스템이나 리더보드, 시간 순 이벤트 로그 처리에 가장 적합합니다. Hash와 String 다중 키의 선택 기준도 중요한데, 관련 데이터를 묶어서 관리하고 메모리 효율이 중요하다면 Hash를 선택하고, 각 개별 필드에 서로 다른 TTL을 설정해야 한다면 String 다중 키를 써야 합니다. Hash는 키 전체에만 TTL을 적용할 수 있고 개별 필드 TTL은 지원하지 않기 때문입니다.

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

**모범 답변 (3분 이상 말하기 형태)**:

> 분산락은 재고 차감이나 중복 결제 방지처럼 여러 서버에서 동시에 하나의 자원에 접근할 때 순서를 보장해야 하는 상황에서 사용합니다. Redis로 분산락을 구현할 때는 `SET key value NX PX {ttl}` 명령어로 원자적으로 락을 획득합니다. NX 옵션은 키가 존재하지 않을 때만 설정하고, PX는 밀리초 단위 TTL을 지정해 클라이언트가 죽어도 락이 자동으로 해제되게 합니다. 락 해제 시 단순히 DEL을 쓰면 안 됩니다. GET으로 소유자를 확인하고 DEL을 실행하는 사이에 다른 요청이 끼어들 수 있기 때문에, GET 확인과 DEL 삭제를 하나의 원자적 연산으로 처리하는 Lua 스크립트를 사용해야 합니다. 단일 노드 SETNX의 약점은 해당 Redis 인스턴스가 장애가 나면 락 정보가 유실된다는 점입니다. 이를 해결하기 위한 방법이 Redlock 알고리즘인데, N개의 독립적인 Redis 인스턴스에 과반수 이상 락 획득에 성공했을 때만 락을 확보한 것으로 간주합니다. 단일 노드 장애가 생겨도 나머지 과반수가 유지되는 한 락의 안전성이 보장됩니다. 고가용성이 필요한 경우 Redlock, 단순한 환경에서는 SETNX 방식을 선택합니다.

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

**모범 답변 (3분 이상 말하기 형태)**:

> Redis의 MULTI/EXEC 트랜잭션은 MULTI로 시작해서 이후 명령들이 즉시 실행되지 않고 QUEUED 상태로 대기열에 쌓이다가, EXEC를 실행하는 순간 대기열의 명령들이 격리된 상태에서 일괄 실행되는 방식입니다. RDBMS 트랜잭션과 가장 중요한 차이는 롤백이 없다는 점입니다. RDBMS는 트랜잭션 중 하나의 명령이 실패하면 이전 상태로 전체 롤백이 가능하지만, Redis의 MULTI/EXEC는 중간 명령이 실패하더라도 나머지 명령들은 그대로 계속 실행됩니다. 따라서 일관성 보장은 Redis가 해주지 않고 애플리케이션이 직접 책임져야 합니다. 트랜잭션을 취소하고 싶다면 EXEC 전에 DISCARD를 호출하면 대기열이 비워집니다. 낙관적 잠금이 필요한 경우에는 WATCH 명령어와 함께 사용하는데, WATCH로 지정한 키가 EXEC 전에 외부에서 변경되면 EXEC가 nil을 반환해 트랜잭션 실패를 알립니다.

**꼬리 질문 예시**:
- MULTI/EXEC에서 명령 하나가 실패하면 어떻게 되나요?
- WATCH를 함께 쓰는 이유는 무엇인가요?

> 출처: https://redis.io/docs/latest/develop/using-commands/transactions/

---

## go-redis에서 Pipeline과 TxPipeline의 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: Pipeline, TxPipeline, MULTI/EXEC 래핑, 원자성, 네트워크 왕복

**모범 답변 (3분 이상 말하기 형태)**:

> Pipeline과 TxPipeline은 둘 다 여러 명령을 한 번의 네트워크 왕복으로 전송해 효율을 높이는 방식이지만, 원자성 보장 여부에서 결정적으로 다릅니다. Pipeline은 여러 명령을 한꺼번에 보내서 네트워크 RTT를 줄이는 순수한 네트워크 최적화입니다. 명령들 사이에 원자성은 없기 때문에 중간에 다른 클라이언트의 명령이 끼어들 수 있습니다. TxPipeline은 내부적으로 MULTI와 EXEC로 명령들을 감싸서 전송하기 때문에 대기열에 쌓인 명령들이 격리된 상태로 일괄 실행되어 원자성이 보장됩니다. 실무에서 TxPipeline 단독 사용보다는 `rdb.Watch()`와 함께 `tx.TxPipelined()`를 사용하는 낙관적 잠금 패턴으로 주로 씁니다. WATCH로 지정한 키가 변경되면 EXEC가 실패하고 `redis.TxFailedErr`를 반환하기 때문에 애플리케이션에서 재시도 처리를 구현해야 합니다. 충돌이 드문 환경에서는 매번 락을 거는 비관적 잠금보다 이 낙관적 잠금 방식이 처리량 면에서 유리합니다.

**꼬리 질문 예시**:
- TxPipeline을 단독으로 쓰는 것과 Watch와 함께 쓰는 것의 차이는?
- 낙관적 잠금이 비관적 잠금(분산락)보다 적합한 상황은 언제인가요?

> 출처: https://redis.uptrace.dev/guide/go-redis-pipelines.html

---

## MULTI/EXEC와 Lua 스크립트의 차이점과 선택 기준은?

**난이도**: 심화

**핵심 키워드**: 조건부 로직, 원자성, 네트워크 효율, 분산락 해제, Redis Cluster

**모범 답변 (3분 이상 말하기 형태)**:

> MULTI/EXEC와 Lua 스크립트는 둘 다 원자성을 보장하고 롤백이 없다는 공통점이 있지만, 사용 목적과 기능 측면에서 중요한 차이가 있습니다. MULTI/EXEC는 단순히 여러 명령을 묶어서 격리된 상태로 실행하는 데 적합합니다. 하지만 조건부 처리가 불가능하고, Redis Cluster 환경에서 다른 슬롯의 키를 한 트랜잭션으로 묶으면 Cross-slot 에러가 발생합니다. Lua 스크립트는 서버 사이드에서 실행되기 때문에 네트워크 왕복이 한 번만 발생하고, if/else 같은 조건부 로직을 포함할 수 있습니다. Redis Cluster에서도 KEYS 인자를 통해 슬롯 라우팅이 가능합니다. 선택 기준은 명확합니다. 단순히 명령들을 묶어서 실행하는 것이 목적이라면 MULTI/EXEC를 쓰고, 현재 값을 읽어서 조건에 따라 분기하거나 다른 명령을 실행해야 한다면 Lua를 써야 합니다. 분산락 해제가 Lua를 반드시 써야 하는 대표적인 사례입니다. 내 소유의 락인지 GET으로 확인하고 맞으면 DEL하는 흐름이 원자적으로 처리되어야 하는데, 이 조건부 분기는 MULTI/EXEC로는 표현할 수 없습니다. 재고 차감처럼 현재 값을 확인하고 조건에 맞을 때만 갱신하는 패턴도 마찬가지입니다.

**꼬리 질문 예시**:
- 분산락 해제를 MULTI/EXEC로 구현할 수 없는 이유는? → WATCH + MULTI/EXEC로 가능하지만 GET 결과를 EXEC 전에 읽어야 해서 흐름이 복잡. 조건 확인 후 분기가 불가능
- Lua 스크립트의 단점은 무엇인가요? → 디버깅 어려움(서버 사이드 실행), 스크립트 길어지면 가독성 저하, EVALSHA 캐시 휘발성
- Redis Cluster에서 MULTI/EXEC를 쓰면 안 되는 이유는? → 다른 슬롯의 키를 한 트랜잭션으로 묶으면 Cross-slot 에러

**면접 세션 피드백 (2026-04-27 1회차)**:
- 잘한 점: MULTI/EXEC 큐잉 → 순차 실행, 단일 노드 원자성, Lua Script 싱글 스레드 블로킹 주의점 정확
- 보완:
  - **Cluster Cross-slot 제약**: "다른 노드의 동시 접근"으로 잘못 설명 → 핵심은 **MULTI/EXEC에 포함된 키들이 서로 다른 해시 슬롯에 있으면 CROSSSLOT 오류 발생**. 해결: `{tag}` 해시태그로 같은 슬롯 강제 배치
  - **no-rollback**: 꼬리 질문에서 정확히 답변 ✅
  - **Lua 조건 분기**: `if-else` 조건부 쓰기 가능 — "조회 결과에 따라 분기" 불가한 MULTI/EXEC와의 핵심 차이 미언급
  - **실무 연결**: `ViewerRedisRepository.addViewer()`에서 직접 MULTI/EXEC 사용 경험 언급 기회 놓침
- 점수: 6/10 (핵심 키워드 3/5, 구조 2/3, 꼬리 1/2, 보너스 +0)

> 출처: https://dgle.dev/redis-multi-lua/

---

## Redis Lua 스크립트에서 EVAL과 EVALSHA의 차이는? 언제 EVALSHA를 사용해야 하나요?

**난이도**: 심화

**핵심 키워드**: EVAL, EVALSHA, SCRIPT LOAD, SHA1, 스크립트 캐싱, NOSCRIPT 에러

**모범 답변 (3분 이상 말하기 형태)**:

> EVAL과 EVALSHA의 차이는 스크립트 전송 방식에 있습니다. EVAL은 호출할 때마다 전체 스크립트 본문을 서버로 전송하기 때문에 스크립트가 길어질수록 네트워크 오버헤드가 커집니다. 반면 EVALSHA는 SCRIPT LOAD 명령어로 스크립트를 서버에 미리 캐싱하고 반환받은 SHA1 해시만 전송하기 때문에 스크립트를 반복 호출하는 환경에서 네트워크 효율이 훨씬 좋습니다. 다만 중요한 주의점이 있는데, 스크립트 캐시는 휘발성입니다. 서버 재시작, SCRIPT FLUSH 호출, 페일오버 발생 시 캐시가 초기화됩니다. 이때 EVALSHA로 호출하면 NOSCRIPT 에러가 반환됩니다. 따라서 EVALSHA를 사용하는 코드에는 NOSCRIPT 에러 발생 시 EVAL로 폴백해 스크립트를 재로드하는 방어 코드가 반드시 필요합니다. 또한 파이프라인 내에서 EVALSHA를 사용하면 NOSCRIPT 에러가 중간에 발생했을 때 처리가 어렵기 때문에 파이프라인에서는 EVAL을 권장합니다.

**꼬리 질문 예시**:
- EVALSHA 사용 중 서버가 재시작되면 어떻게 되나요? → NOSCRIPT 에러 → EVAL로 재로드 필요
- 파이프라인에서 EVALSHA를 쓰면 안 되는 이유는? → 파이프라인 중 NOSCRIPT 에러가 발생해도 처리가 어려움 → EVAL 권장

> 출처: https://redis.io/docs/latest/develop/programmability/eval-intro/

---

## redis.call()과 redis.pcall()의 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: redis.call, redis.pcall, 에러 처리, 스크립트 중단

**모범 답변 (3분 이상 말하기 형태)**:

> `redis.call()`과 `redis.pcall()`은 Lua 스크립트 내에서 Redis 명령어를 실행하는 함수인데, 에러 처리 방식이 다릅니다. `redis.call()`은 명령 실행 중 에러가 발생하면 즉시 클라이언트에 에러를 전파하고 스크립트 실행을 중단합니다. 반면 `redis.pcall()`은 에러를 외부로 전파하지 않고 에러 내용을 테이블 형태로 반환해서, 스크립트 내부에서 에러를 캐치하고 상황에 맞는 별도 로직을 실행할 수 있습니다. 선택 기준은 에러 발생 시 처리 방식에 달려 있습니다. 에러가 생기면 그대로 실패 처리해도 되는 경우에는 `redis.call()`을 쓰고, 에러 종류에 따라 다른 분기 처리가 필요하거나 부분 실패를 허용하면서 나머지 로직을 계속 실행해야 하는 경우에는 `redis.pcall()`을 선택합니다.

**꼬리 질문 예시**:
- Lua 스크립트에서 롤백이 안 된다면 부분 실행 중 에러 처리는 어떻게 하나요?

> 출처: https://redis.io/docs/latest/develop/programmability/eval-intro/

---

## 인기 상품 캐시 만료 시 Cache Stampede 대응 설계

**난이도**: 심화

**핵심 키워드**: Probabilistic Early Expiration, Mutex Lock, Stale-While-Revalidate, TTL Jitter, Lock TTL + Watchdog

**모범 답변 (3분 이상 말하기 형태)**:

> Cache Stampede는 인기 키의 캐시가 만료되는 순간 다수의 요청이 동시에 DB를 조회하면서 부하가 폭증하는 문제입니다. 대응 전략은 상황에 따라 선택합니다. 가장 단순한 방법은 TTL Jitter로, TTL에 랜덤 오프셋을 더해 여러 키의 만료 시점을 분산시키는 방식입니다. 구현 비용이 낮고 Cache Avalanche 예방에도 함께 효과적입니다.
>
> 더 강력한 일관성이 필요하다면 Mutex Lock 방식을 씁니다. 첫 번째 요청만 DB를 조회하고 나머지는 락을 기다리게 직렬화하는데, 락 소유자가 장애로 죽을 경우를 대비해 Lock에 TTL을 반드시 설정해야 합니다. 처리 시간이 TTL을 초과할 수 있는 경우에는 Redisson의 Watchdog처럼 별도 스레드가 주기적으로 Lock TTL을 갱신하는 패턴을 사용합니다.
>
> Probabilistic Early Expiration은 TTL이 완전히 만료되기 전에 확률적으로 선제 갱신을 트리거하는 방식으로, 만료 시점에 몰리는 동시 요청 자체를 원천 차단합니다. 반면 일시적 stale 데이터를 허용할 수 있는 환경이라면 Stale-While-Revalidate가 가장 응답 지연이 적습니다. 만료된 데이터를 즉시 반환하면서 백그라운드에서 갱신하기 때문에 사용자에게 레이턴시 증가가 보이지 않습니다. 각 전략의 일관성/복잡도/메모리 트레이드오프를 기준으로 서비스 특성에 맞게 선택합니다.

**꼬리 질문 예시**:
- Redis 싱글 스레드 특성이 Mutex Lock 구현에서 어떤 이점을 주나요?
- Stale-While-Revalidate 방식에서 갱신 실패 시 stale 데이터가 계속 서빙될 수 있는데 어떻게 처리하나요?
- 분산 환경에서 Redis Cluster를 쓸 때 분산락이 안전하지 않은 이유와 Redlock 알고리즘을 설명해주세요.

---

## Redis Sorted Set은 내부적으로 어떻게 구현되어 있나요? ListPack과 SkipList는 언제 각각 사용되나요?

**난이도**: 중급

**핵심 키워드**: ListPack, SkipList, Hashtable, zset-max-listpack-entries, 인코딩 전환, 메모리 vs 성능

**모범 답변 (3분 이상 말하기 형태)**:

> Sorted Set은 데이터 규모에 따라 내부 인코딩을 자동으로 선택합니다. 원소 수가 128개 이하이고 각 원소 크기가 64바이트 이하인 경우에는 ListPack 인코딩을 사용합니다. ListPack은 선형 메모리 블록에 데이터를 순차적으로 저장하는 구조로 메모리 효율이 가장 높지만, 조회는 O(N) 선형 탐색이 필요합니다. 원소 수나 크기가 임계값을 초과하면 SkipList와 Hashtable의 조합으로 전환됩니다. SkipList는 score 기준 정렬된 상태를 O(log N)으로 유지하고, Hashtable은 ZSCORE 조회를 O(1)으로 처리하기 위해 함께 관리됩니다. 이 인코딩 전환은 단방향입니다. ListPack에서 SkipList로 넘어가면 원소를 삭제해서 임계값 아래로 내려가도 ListPack으로 돌아오지 않습니다. `zset-max-listpack-entries` 값을 높게 설정하면 더 오래 ListPack 인코딩을 유지해 메모리를 절약할 수 있지만, 그만큼 조회 성능이 낮아지는 트레이드오프가 있습니다.

**꼬리 질문 예시**:
- ListPack이 Ziplist를 대체한 이유는 무엇인가요? (Cascading Update 문제)
- `ZSCORE`가 O(1)인 이유는? (Hashtable이 함께 관리되기 때문)
- 기본 임계값(128개)을 512개로 올리면 어떤 트레이드오프가 생기나요?

> 출처: https://redis.io/docs/latest/develop/data-types/sorted-sets/ | https://jothipn.github.io/2023/04/07/redis-sorted-set.html

---

## Redis SkipList의 O(log N) 성능은 어떻게 달성되나요? Redis가 Red-Black Tree 대신 SkipList를 선택한 이유는?

**난이도**: 심화

**핵심 키워드**: SkipList, 다층 레이어, Span, 범위 조회, O(log N), Red-Black Tree, Express Lane

**모범 답변 (3분 이상 말하기 형태)**:

> SkipList는 다층 레이어 구조로 O(log N) 성능을 달성합니다. 상위 레이어에서 큰 폭으로 건너뛰며 목표에 빠르게 접근하고, 목표에 가까워질수록 하위 레이어로 내려와 정밀 탐색하는 방식입니다. 삽입 시 확률적으로 레이어 높이를 결정하는데, 기본 확률 p=0.25로 평균 O(log N) 수준의 노드 분포가 유지됩니다. ZRANK 같은 순위 조회가 빠른 이유는 Span이라는 메타데이터 덕분입니다. 각 레이어 포인터에 "건너뛴 노드 수"를 저장해 두어, HEAD에서 목표 노드까지 span을 누적하면 O(log N)에 순위를 계산할 수 있습니다. Red-Black Tree로 같은 기능을 구현하려면 각 서브트리의 노드 수를 별도로 관리해야 합니다. Redis가 Red-Black Tree 대신 SkipList를 선택한 이유는 세 가지입니다. 첫째, 범위 조회가 쉽습니다. SkipList의 최하위 레이어는 정렬된 연결 리스트이기 때문에 ZRANGEBYSCORE나 ZRANGEBYRANK 같은 연속 스캔이 자연스럽습니다. Red-Black Tree에서 범위 조회는 In-order Traversal이 필요해 더 복잡합니다. 둘째, 구현이 단순합니다. Red-Black Tree의 회전과 색 변경 로직이 필요 없습니다. 셋째, Lock-free 구현이 Red-Black Tree보다 상대적으로 쉬워 동시성 친화적입니다.

**꼬리 질문 예시**:
- SkipList의 최악의 경우 시간복잡도는? 어떤 상황에서 발생하나요? (O(N) — 모든 노드가 레이어 1만 가질 때. 확률적으로 극히 드묾)
- `ZRANGE key 0 -1`이 O(N)인 이유는? (전체 원소 반환이므로 출력 크기 자체가 N)
- Hashtable 없이 SkipList만으로 구현하면 `ZSCORE`의 복잡도는? (O(log N) — SkipList 탐색 필요)

> 출처: https://jothipn.github.io/2023/04/07/redis-sorted-set.html | https://sam-wei.medium.com/deep-dive-into-redis-zset-internals-8d10fa1f674c

---

## Ziplist의 Cascading Update 문제란 무엇이고, ListPack은 어떻게 이를 해결했나요?

**난이도**: 심화

**핵심 키워드**: Ziplist, ListPack, Cascading Update, prevlen, 인코딩, Redis 7.0

**모범 답변 (3분 이상 말하기 형태)**:

> Ziplist의 Cascading Update는 구조적 설계에서 비롯된 문제입니다. Ziplist의 각 엔트리는 이전 엔트리의 크기를 `prevlen` 필드에 저장하는데, 이 필드는 이전 엔트리가 253바이트 이하면 1바이트, 그 이상이면 5바이트를 사용합니다. 여기서 문제가 생깁니다. 중간에 있는 원소의 크기가 253바이트에서 254바이트로 늘어나면 그 다음 엔트리의 `prevlen`이 1바이트에서 5바이트로 커져야 합니다. 그런데 `prevlen`이 커지면 그 엔트리 전체 크기도 커지고, 그러면 그 다음 엔트리의 `prevlen`도 업데이트가 필요해지는 연쇄 반응이 일어납니다. 최악의 경우 O(N) 재기록이 발생하고, 작은 집합이라도 예측 불가한 지연이 생깁니다. ListPack은 Redis 7.0 이후 기본 인코딩으로, 이 문제를 `prevlen` 필드 자체를 제거하는 방식으로 해결했습니다. 대신 각 엔트리 끝에 자신의 길이 정보를 인코딩해 두어 역방향 스캔도 가능하게 만들었습니다. 이 설계 변경으로 Cascading Update 문제가 완전히 사라졌고, 헤더 크기도 10바이트에서 6바이트로 줄었습니다.

**꼬리 질문 예시**:
- ListPack에서 역방향 스캔은 어떻게 가능한가요? (각 엔트리 끝에 자신의 바이트 수 인코딩 → 포인터를 해당 바이트만큼 뒤로 이동)
- Redis 7.0 이전 버전에서 Ziplist를 쓰는 코드와의 하위 호환성은 어떻게 처리되나요?

> 출처: https://redis.io/glossary/redis-ziplist/ | https://github.com/antirez/listpack/blob/master/listpack.md

---

---

## Rate Limiting

### Q. API Rate Limiting 알고리즘 중 Token Bucket과 Sliding Window Log를 비교해주세요. 분산 환경에서 Redis로 구현할 때 각각 어떤 자료구조를 사용하나요?

**난이도**: 기초

**핵심 키워드**: Token Bucket, Sliding Window Log, SortedSet, Hash/String, burst, O(1) vs O(요청 수)

**모범 답변 (3분 이상 말하기 형태)**:

> Token Bucket과 Sliding Window Log는 Rate Limiting을 구현하는 대표적인 두 알고리즘입니다. Token Bucket은 일정 속도로 토큰을 채우고 요청이 들어올 때마다 토큰을 소비하는 방식입니다. 버킷에 토큰이 남아 있는 한 순간적인 burst 트래픽을 허용하는 것이 설계 의도입니다. Redis로 구현할 때는 String이나 Hash의 두 필드, 즉 남은 토큰 수와 마지막 refill 시간을 저장하는 구조를 사용합니다. 조회와 갱신이 O(1)이고 메모리 효율이 좋습니다. Sliding Window Log는 요청마다 타임스탬프를 Sorted Set에 기록하고, 현재 시간 기준 window 범위를 벗어난 타임스탬프를 제거한 뒤 남은 count를 기준으로 허용 여부를 판단합니다. Fixed Window처럼 경계 시점에 2배 burst가 발생하는 문제가 없고 정확한 QPS 제어가 가능하지만, 요청 수만큼 Sorted Set에 데이터가 쌓여 메모리를 더 사용하고 연산도 O(요청 수) 수준입니다. 선택 기준은 burst 허용 여부와 메모리 효율을 기준으로 판단합니다. 적당한 burst를 허용하면서 메모리 효율을 중시한다면 Token Bucket을, 정확한 QPS 경계 제어가 필요하다면 Sliding Window Log를 선택합니다.

**꼬리 질문 예시**:
- "Token Bucket에서 refill 로직은 어떻게 구현하나요?" → HGETALL로 last_refill 읽고, 경과 시간 × rate로 토큰 계산 후 HSET
- "Fixed Window Counter와 Sliding Window Log의 차이는?" → Fixed는 경계 시점에 2배 burst 가능, Sliding은 정확

**면접 세션 피드백 (2026-04-03 1회차)**:
- 두 알고리즘 특징 방향은 맞음
- Token Bucket Redis 구조 "Set과 count" → String/Hash 2필드로 교정 필요
- "burst를 방지 못한다" → "burst를 허용하는 설계"로 표현 교정
- 메모리 트레이드오프 방향: Token Bucket이 효율적(O(1)), Sliding Window Log가 많이 씀(O(요청 수))

---

## Redis 캐싱 전략

### Q. Cache-Aside, Write-Through, Write-Behind의 차이를 설명해주세요. 각 전략에서 캐시-DB 정합성 문제는 언제 발생하고 어떻게 해결하나요?

**난이도**: 기초

**핵심 키워드**: Cache-Aside, Write-Through, Write-Behind, Cache Invalidation, Race Condition, AOF/RDB, Write Latency, Cache Pollution, 데이터 유실

**모범 답변 (3분 이상 말하기 형태)**:

> 세 가지 캐싱 전략은 쓰기 주체와 정합성 보장 수준에서 각각 다른 특성을 가집니다. Cache-Aside는 캐시 Miss가 발생하면 애플리케이션이 직접 DB를 조회하고 캐시에 데이터를 세팅하는 구조입니다. 정합성 관리는 애플리케이션 책임인데, DB 업데이트 시 캐시를 Update하지 않고 Invalidate, 즉 삭제하는 것이 안전합니다. Update 방식은 두 쓰기가 서로 다른 순서로 캐시를 업데이트하면 stale 데이터가 고착되는 Race Condition이 발생할 수 있기 때문입니다. 동시 Miss 문제는 분산락으로 첫 요청만 DB를 조회하게 직렬화해서 해결할 수 있습니다.
>
> Write-Through는 쓰기 시 캐시와 DB를 순차적으로 동기 업데이트하기 때문에 캐시가 항상 최신 상태를 유지합니다. 다만 두 곳에 쓰기 때문에 Write Latency가 증가하고, 한 번도 읽히지 않을 데이터도 캐시에 적재되는 Cache Pollution이 발생합니다. TTL 설정으로 stale 데이터와 Cache Pollution을 함께 완화할 수 있습니다.
>
> Write-Behind는 캐시에 먼저 쓰고 나중에 배치로 DB에 동기화하기 때문에 쓰기 성능이 가장 좋습니다. 핵심 리스크는 캐시 장애 시 아직 DB에 반영되지 않은 쓰기 데이터가 영구적으로 유실될 수 있다는 점입니다. 이를 완화하려면 Redis AOF나 RDB persistence를 활성화해서 재시작 시 미반영 데이터를 복구할 수 있게 해야 합니다. 좋아요 수나 집계처럼 일부 유실을 허용할 수 있는 데이터에는 적합하지만, 결제 금액처럼 절대 유실이 허용되지 않는 데이터에는 사용해서는 안 됩니다.

**꼬리 질문 예시**:
- Write-Behind에서 캐시 서버가 죽으면 어떤 문제가 생기나요? → DB 미반영 쓰기 데이터 유실. AOF/RDB로 완화
- Cache-Aside에서 DB 업데이트 후 캐시를 Delete vs Update 중 어느 것이 안전한가요? → Delete(Invalidation)가 안전. Update는 두 쓰기의 순서 역전 시 stale 데이터 고착 위험
- Write-Through에서 캐시 쓰기 성공, DB 쓰기 실패 시 어떻게 되나요? → 정합성 불일치 → 트랜잭션/2PC 또는 캐시 롤백 로직 필요

**면접 세션 피드백 (2026-04-07 2회차)**:
- 잘한 점: 3가지 전략 사용 목적 명확히 구분. Write-Behind 사용 사례(좋아요, 집계, 로그) 구체적. Cache-Aside 동시성 문제 → 분산락 언급 — 실무 감각 좋음
- 보완: Write-Behind 핵심 리스크("데이터 유실") 미언급. Cache-Aside "캐시에 반영" → Invalidation(삭제)이 정확한 표현. Write-Through 단점: Write Latency 증가 + Cache Pollution 추가 필요

---

## 한정 재고 동시 예약 처리 — Redis + DB 2-레이어 설계

**난이도**: 중급

**핵심 키워드**: Lua script, 원자적 UPDATE, write-around + invalidation, TTL 안전망, Circuit Breaker, oversell

**모범 답변 (3분 이상 말하기 형태)**:

> 한정 재고 동시 예약 처리는 Redis와 DB를 2-레이어로 구성해서 고트래픽을 단계적으로 필터링하는 방식으로 설계합니다. 첫 번째 레이어는 Redis로, Lua 스크립트를 사용해 재고를 GET하고 0이면 즉시 실패, 아니면 DECR하는 흐름을 원자적으로 처리합니다. Redis 싱글 스레드 특성 덕분에 별도의 분산락 없이도 동시 요청이 직렬화됩니다. 단, Lua 스크립트에는 경량 연산만 담아야 합니다. 무거운 로직이 들어가면 그 시간 동안 전체 Redis 응답이 저하되기 때문입니다.
>
> 두 번째 레이어는 DB로, `UPDATE ... SET count = count - 1 WHERE id = ? AND count > 0` 형태의 조건부 UPDATE를 사용합니다. InnoDB의 row-level 배타락으로 동시 UPDATE가 직렬화되고, 영향받은 row 수가 0이면 재고가 없다고 판단해 예외 처리합니다. 이 DB 레이어가 oversell을 막는 최후 방어선입니다.
>
> Redis와 DB 사이 불일치 대응도 중요합니다. 쓰기 순서는 DB UPDATE를 먼저 하고 그 후 Redis를 invalidate해야 합니다. 반대 순서면 DB 커밋 전에 다른 스레드가 구버전을 Redis에 캐싱하는 race condition이 생깁니다. DB 커밋 성공 후 Redis invalidation이 실패하면 oversell 위험이 있는데, 이때 TTL을 안전망으로 설정해 만료 후 DB에서 재동기가 일어나도록 합니다. Redis가 완전히 장애 나면 Redis를 우회하고 DB 원자적 UPDATE 단독으로 처리하는 fallback 경로도 Circuit Breaker와 함께 준비해야 합니다. 이 패턴에서 쓰기 흐름은 Cache-Aside가 아니라 write-around + invalidation이 정확한 표현입니다. Cache-Aside는 읽기 패턴이기 때문입니다.

**꼬리 질문 예시**:
- "Lua script 없이 GET + DECR를 따로 실행하면 어떤 문제가 생기나요?" → GET 결과 확인 후 DECR 사이에 다른 요청이 끼어들어 음수 재고 발생 가능
- "DB TTL 안전망 TTL을 얼마로 설정하나요?" → 예약 처리 최대 시간보다 길게 (예: 5분), 재고 변경 빈도에 따라 조정
- "Redis 재고가 0인데 DB 재고가 1인 상황은 언제 발생하나요?" → Redis 장애 후 복구, TTL 만료 후 캐시 재적재 전 등
- "낙관적 락 vs 비관적 락 vs Redis 분산락 중 이 케이스에 Redis를 선택한 이유는?" → 비관적 락은 DB 락 경합으로 처리량 제한, 낙관적 락은 충돌 시 재시도 비용 증가, Redis는 고트래픽 요청을 DB 도달 전에 빠르게 필터링

**면접 세션 피드백 (2026-04-07 1회차)**:
- 잘한 점: 2-레이어 설계 즉각 제시. DB 원자적 UPDATE 패턴 정확. Lua script + 단일 스레드 원자성 설명 우수. Lua 과부하 주의사항까지 언급. Cache invalidate race condition 직접 짚어낸 점 인상적.
- 보완: Cache-Aside 용어 오용(읽기 패턴임). DB 커밋 후 Redis invalidation 실패 시 oversell 케이스 누락 → TTL 안전망 언급 필요. Redis 장애 fallback 미언급.

---

## Redis Hash 내부 인코딩(listpack/ziplist/hashtable)이 어떻게 동작하는지 설명해주세요. 언제 listpack에서 hashtable로 전환되고, 각각의 메모리 효율 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: listpack, ziplist, hashtable, 연속 메모리, cascading update, 인코딩 전환 임계값

**모범 답변 (1분 이상 말하기 형태)**:
> "Redis Hash는 데이터 크기에 따라 내부 인코딩을 자동으로 전환합니다. 필드 수가 128개 이하이고 각 값의 크기가 64바이트 이하일 때는 listpack 인코딩을 사용하고, 이 임계값을 초과하면 hashtable로 전환됩니다.
>
> listpack은 Redis 7.0에서 이전의 ziplist를 대체한 구조입니다. 둘 다 연속된 메모리 블록에 항목을 순서대로 저장하는 컴팩트한 방식인데, ziplist는 각 항목이 이전 항목의 길이(prevlen)를 저장하는 구조였습니다. 이 때문에 중간 항목을 삽입하거나 삭제할 때 prevlen 값이 바뀌고, 그 변경이 이후 모든 항목에 연쇄적으로 전파되는 캐스케이딩 업데이트 문제가 있었습니다. 최악의 경우 O(N²) 재작성이 발생할 수 있었습니다.
>
> listpack은 이 문제를 해결하기 위해 각 항목이 자신의 길이를 항목 끝에 저장하는 방식으로 변경했습니다. 역방향 탐색 시 이전 항목을 참조하지 않아도 되므로, 삽입·삭제 시 다른 항목에 전혀 영향을 주지 않습니다. 또한 헤더도 10바이트에서 6바이트로 줄었습니다.
>
> 메모리 효율 측면에서, listpack은 포인터 오버헤드 없이 연속 메모리에 데이터를 저장하기 때문에 hashtable보다 5~10배 메모리를 절약할 수 있습니다. 사용자 세션처럼 5~10개 필드를 가진 Hash는 listpack 인코딩이 유지되어 메모리 효율이 높습니다. `OBJECT ENCODING key` 명령으로 현재 인코딩을 확인할 수 있습니다."

**꼬리 질문 예시**:
- ziplist의 캐스케이딩 업데이트 문제가 실제로 성능에 얼마나 영향을 주나요?
- hash-max-listpack-entries 임계값을 높이면 어떤 장단점이 있나요?
- listpack이 도입된 Redis 버전은 언제인가요? ziplist와 어떤 자료구조에서 대체됐나요?

> 출처: [How Redis Hashes Work Internally (listpack)](https://oneuptime.com/blog/post/2026-03-31-redis-hashes-work-internally-hashtable-listpack/view) | [listpack spec](https://github.com/antirez/listpack/blob/master/listpack.md)

## ZRANGE vs ZRANGEBYSCORE 사용 기준

### Q. Redis Sorted Set에서 `ZRANGE`와 `ZRANGEBYSCORE`의 차이와 각각 언제 사용하나요?

**난이도**: 기초

**핵심 키워드**: ZRANGE, ZRANGEBYSCORE, 인덱스 번호(0-based), score 값 범위, 단일 스레드 원자성

**핵심 구분**:
- `ZRANGE key 0 9` → **인덱스 번호** 기준 (0번째~9번째 = 상위 10개). "상위 N개" 조회에 사용.
- `ZRANGEBYSCORE key 1000 5000` → **score 값** 기준 (score 1000~5000 사이). "특정 점수 구간" 필터에 사용.

**실무 선택 기준**:
- 상위 10개 광고주 조회 → `ZRANGE key 0 9`
- 입찰가 1,000원~5,000원 사이 광고주 → `ZRANGEBYSCORE key 1000 5000`

**원자성 보장 이유**: Redis는 단일 스레드로 명령을 처리하기 때문에 여러 서버가 동시에 ZADD를 호출해도 이벤트 루프가 하나씩 순서대로 처리합니다. 별도 락 없이도 score 업데이트 정합성이 유지됩니다.

**꼬리 질문 예시**:
- ZRANGE 0 -1은 어떤 의미인가요? → -1은 마지막 인덱스를 의미. 전체 조회.
- ZREVRANGE와 ZRANGE의 차이는? → ZREVRANGE는 내림차순(score 높은 것부터)

**면접 세션 피드백 (2026-05-02 1회차)**:
- 잘한 점: ZRANGE/ZRANGEBYSCORE 구분 방향 정확, 단일 스레드 원자성 파악
- 보완: ZRANGE 파라미터가 **인덱스 번호(0-based)**임을 명시. "순위를 검색"이 아닌 "`ZRANGE key 0 9` → 0번째~9번째 = 상위 10개"로 구체적으로 표현.

---
