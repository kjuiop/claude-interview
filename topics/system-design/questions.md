---
tags: [system-design, architecture, real-time, interview-questions]
related: [redis, kafka, kubernetes, golang, distributed-systems]
---

# System Design — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/system-design/concepts]]

---

## CQRS (Command Query Responsibility Segregation)

### Q. CQRS 패턴을 설명하고, 두 모델 간 동기화와 Eventual Consistency 허용 시 stale 대응 방법을 설명해주세요.

**모범 답변 방향**:

CQRS는 Command와 Query의 책임을 분리하는 패턴으로, 분리 수준에 따라 두 가지로 나뉩니다. 코드 레벨 분리는 같은 DB를 쓰면서 Command(쓰기)와 Query(읽기) 처리 경로만 분리하는 것이고, 저장소 레벨 분리는 쓰기 DB(PostgreSQL)와 읽기 DB(Redis, Elasticsearch)를 물리적으로 나누는 방식입니다. 저장소 수준으로 분리하면 읽기 저장소만 수평 확장해서 트래픽을 수용할 수 있고, 쓰기는 트랜잭션과 정합성에 집중하고 읽기는 조회 성능에 최적화할 수 있습니다. 두 저장소의 동기화 방식으로는 Event-Driven CQRS와 CDC 두 가지가 있습니다. Event-Driven CQRS는 쓰기 모델이 Kafka 이벤트를 발행하면 읽기 모델이 구독해서 업데이트하는 방식이고, CDC(Change Data Capture)는 Debezium 같은 도구가 binlog를 캡처해서 읽기 저장소에 자동으로 반영하는 방식입니다. 두 저장소 간에는 불가피하게 전파 지연이 생기는데, stale 데이터 대응 방법은 상황에 따라 네 가지가 있습니다. 결제나 잔액처럼 정합성이 중요한 경우에는 읽기 저장소를 거치지 않고 Command DB에서 직접 읽습니다. 지연이 허용되는 경우에는 사용자에게 "반영까지 N초 소요"를 안내하는 것으로 충분합니다. 즉각적인 UX가 필요한 경우에는 클라이언트가 쓰기 결과를 즉시 UI에 반영하고 이후 서버 응답으로 보정하는 Optimistic Update 방식을 씁니다. 마지막으로 쓰기 응답에 버전이나 타임스탬프를 포함시키고 클라이언트가 해당 버전 이상을 요청하는 read-your-writes 패턴도 활용할 수 있습니다.

**이력서 연결:**
> "카테노이드 채팅 서버에서 메시지 저장은 PostgreSQL에 쓰고, 채팅방 목록/최근 메시지 미리보기는 Redis에 캐시하는 구조로 분리했다면 5,000 동시접속에서 가장 빈번한 조회 트래픽을 Command DB와 분리해 부하를 줄일 수 있었을 것입니다."

**꼬리 질문 대비:**
- "CQRS와 Event Sourcing의 차이는?" → Event Sourcing은 상태 대신 이벤트 자체를 저장. CQRS는 읽기/쓰기 분리만. 같이 쓰는 경우 많지만 별개 패턴.
- "읽기 모델이 stale할 때 언제 직접 읽기를 허용하고 언제 안 하나요?" → 비즈니스 중요도 기준. 과금·재고·잔액은 직접 읽기, 피드·조회수는 eventual 허용.

**면접 세션 피드백 (2026-04-12 3회차)**:
- 저장소 분리 이점, binlog CDC 동기화 언급 좋음
- stale 대응 패턴을 "Command DB 직접 조회" 하나만 언급 → 4가지 패턴 암기 필요
- 동기화 방식에 Event-Driven CQRS, CDC 패턴 이름 미언급
- 이력서 채팅 서버 연결 누락 패턴 반복 → 면접 중 의식적으로 연결 시도 필요

---

## 대용량 실시간 채팅 시스템 설계

### Q. 동시 시청자 10만 명이 참여하는 라이브 스트리밍 채팅 시스템을 처음부터 설계한다면 어떤 아키텍처를 선택하시겠나요?

**핵심 답변 포인트:**

1. **기술 선택 + 이유**
   - Go: goroutine 2KB, 대규모 WebSocket 커넥션 병렬 처리
   - Redis pub/sub: 서버 다중화 시 채팅방별 브로드캐스트, push 기반 낮은 지연
   - MongoDB: 채팅 메시지 메타데이터 유연성, 날짜/채팅방 인덱스로 조회
   - K8s: active connection 또는 CPU 기반 HPA 수평 확장

2. **Redis pub/sub 트레이드오프 명시**
   - 유실 허용 근거: 라이브 채팅은 실시간성 우선
   - 보완: 재연결 시 MongoDB에서 최근 N개 fetch
   - 유실 불허 시: Redis Streams(`XADD`/`XREAD`) 사용

3. **K8s WebSocket Graceful Shutdown**
   - `terminationGracePeriodSeconds` + `preStop hook`
   - readinessProbe 실패 → endpoints 제거 → 기존 커넥션 대기 → 자연 종료

**꼬리 질문 대비:**
- "Redis pub/sub 메시지 유실을 어떻게 처리하나요?" → 재연결 시 MongoDB fallback, 또는 Redis Streams
- "Pod 교체 시 WebSocket은 어떻게 처리하나요?" → terminationGracePeriodSeconds + preStop hook
- "10만 명을 단일 채팅방 vs 여러 채팅방으로 나눌 때 차이는?" → 단일 채팅방: 모든 서버가 동일 topic 구독, 브로드캐스트 부하 집중
- "WebSocket을 여러 서버로 수평 확장할 때 LB 설정은?" → **sticky session 필수** (IP hash 또는 쿠키 기반). round-robin이면 연결 끊김.

**면접 세션 피드백 (2026-04-20 2회차)**:
- Redis vs Kafka 트레이드오프(속도 vs 내구성), last_sequence 복구 패턴, MongoDB 선택 이유(스키마 유연성) 모두 정확.
- 보완: sticky session 미언급(WebSocket 장기 연결 특성상 round-robin LB 불가). Go 설명에서 "이벤트 루프 N개" 오표현(Go는 goroutine M:N 스케줄링, 이벤트 루프 모델 아님). 카테노이드 경험을 "안 됐다"로만 끝냄 → "수평 확장이 필요했다면 이렇게 했을 것" 방향으로 전환 훈련 필요.

**모범 답변:**

동시 시청자 10만 명 규모의 라이브 스트리밍 채팅을 설계한다면, 언어 선택부터 시작합니다. Go는 goroutine 하나가 2KB 수준의 메모리만 사용하기 때문에 수만 개의 WebSocket 커넥션을 단일 서버에서 병렬로 유지할 수 있습니다. 샵라이브에서 라이브 스트리밍 채팅을 다루면서 대규모 동시 커넥션 처리가 언어 선택에 직결된다는 걸 경험했습니다.

서버 간 브로드캐스트는 Redis pub/sub으로 처리합니다. 채팅방 하나를 Redis topic 하나에 대응시키면, 어느 서버에 연결된 사용자든 같은 채팅방 메시지를 받을 수 있습니다. Redis pub/sub은 push 기반이라 지연이 낮고 구현이 단순합니다. 다만 메시지 유실 가능성이 있는데, 라이브 채팅은 실시간성이 우선이므로 이는 허용 가능한 트레이드오프입니다. 유실을 보완하기 위해 재연결 시 MongoDB에서 최근 N개 메시지를 fetch하는 fallback을 둡니다. 순서 보장이나 유실 불허가 요구되는 경우라면 Redis Streams(`XADD`/`XREAD`)나 Kafka로 전환합니다.

메시지 영속 저장에는 MongoDB를 사용합니다. 채팅 메시지는 유형마다 포함 필드가 다를 수 있어 유연 스키마가 유리하고, `{ roomId, createdAt }` 복합 인덱스로 채팅방별 시간순 조회도 효율적으로 처리됩니다. 카테노이드에서 5,000 동시접속 채팅 서버에 MongoDB를 적용한 경험이 있어, 이 패턴의 동작을 실제로 확인했습니다.

인프라는 Kubernetes 위에서 운영하며, active connection 수 또는 CPU 기준으로 HPA를 설정해 트래픽 급증에 자동 대응합니다. Pod 교체 시 WebSocket 커넥션이 갑자기 끊기지 않도록 `terminationGracePeriodSeconds`와 `preStop hook`을 설정해 graceful shutdown을 구현합니다. readinessProbe가 실패하면 새 트래픽을 받지 않고, 기존 커넥션은 grace period 동안 자연스럽게 종료되도록 합니다.

---

## 10만 동시접속 채팅 서버를 설계한다면?

**난이도**: 심화

**핵심 키워드**: WebSocket 수평 확장, Redis pub/sub, Hot Partition, Consistent Hashing, Sequence Number, 재연결, Graceful Shutdown

**모범 답변:**

10만 동시접속을 설계할 때 가장 먼저 하는 것은 용량 계산입니다. Go 기반 WebSocket 서버는 goroutine 하나가 2KB로 경량이기 때문에 서버 1대가 5~8만 커넥션을 안정적으로 처리할 수 있습니다. 10만이면 서버 2대로 수용 가능하지만 장애 대비 여유를 포함해 최소 3대로 시작하고, HPA로 자동 스케일을 구성합니다. CPU보다 goroutine 스케줄러 포화가 먼저 병목이 되는 경우가 많으므로 active connection 수를 함께 메트릭으로 설정합니다.

아키텍처는 세 레이어로 구성합니다. 첫 번째는 WebSocket 서버 레이어입니다. Go의 goroutine 모델 덕분에 대규모 커넥션을 적은 스레드로 병렬 처리할 수 있고, 카테노이드 채팅 서버에서 5,000 동시접속을 Go로 운영한 경험이 이 선택의 근거입니다. 두 번째는 서버 간 브로드캐스트 레이어로 Redis pub/sub을 사용합니다. 채팅방 하나를 Redis topic으로 매핑해서, 어느 서버에 붙은 사용자든 같은 채팅방 메시지를 받습니다. Redis pub/sub은 메시지를 구독자에게 push하기 때문에 지연이 낮습니다. 유실 가능성이 있지만, 라이브 채팅처럼 실시간성이 우선인 경우에는 허용 가능한 트레이드오프이며, 재연결 시 MongoDB에서 최근 메시지를 fetch하는 fallback으로 보완합니다. 세 번째는 영속 저장 레이어로 MongoDB를 사용합니다. `{ roomId, createdAt }` 복합 인덱스로 채팅방별 시간순 조회를 효율적으로 처리하고, 메시지 유형마다 다른 필드 구조도 유연 스키마로 수용합니다.

트레이드오프 측면에서, 히스토리 보장이나 메시지 손실 불허가 요구된다면 Redis pub/sub 대신 Kafka로 전환합니다. 이 경우 room_id를 파티션 키로 써서 같은 채팅방 메시지가 같은 파티션에 들어가도록 하면 순서를 보장할 수 있습니다.

심화 상황으로 Hot Partition 문제가 생기면 Consistent Hashing으로 채팅방을 여러 서버에 분산하고, 인기 채팅방은 자동 분할합니다. 재연결 처리는 클라이언트가 lastSeq를 로컬에 저장하고, 재연결 시 서버에 전달하면 서버가 MongoDB에서 해당 시퀀스 이후 메시지를 fetch해서 보충합니다. 재연결 자체는 지수 백오프를 적용해 서버에 재연결 폭풍이 쏠리는 것을 방지합니다.

**꼬리 질문 예시:**
- "인기 채팅방에 트래픽이 몰리면 어떻게 하나요?" → Consistent Hashing + 채팅방 분할
- "메시지 순서가 뒤집히는 문제는?" → Kafka room_id 파티셔닝 또는 Sequence Number + 클라이언트 재정렬
- "재연결 시 놓친 메시지는?" → lastSeq 저장 → 재연결 시 서버에 전달 → MongoDB에서 diff fetch
- "서버 대수를 얼마나 준비해야 하나요?" → 5만/서버 기준 × 2배 + 장애 여유 = 최소 3대, HPA로 자동 조절
- "Redis pub/sub vs Kafka 언제 어떻게 선택?" → 유실 허용 라이브 채팅 → Redis, 히스토리/손실 불허 → Kafka, 혼합 가능

**실제 사례:**
- LINE LIVE: Redis pub/sub + MySQL(배치), 채팅방 자동 분할로 라이브 급증 대응
- Slack: Kafka로 메시지 순서 보장 + 히스토리 제공

> 출처: https://engineering.linecorp.com/ko/blog/the-architecture-behind-chatting-on-line-live

---

## API Rate Limiting — Token Bucket vs Sliding Window Log

**난이도**: 기초~중급

**핵심 키워드**: Token Bucket, Sliding Window Log, Fixed Window, Leaky Bucket, Redis Sorted Set, Lua 스크립트, burst, 메모리 트레이드오프

**모범 답변 방향**:

Rate Limiting 알고리즘을 선택할 때는 burst 허용 여부와 메모리 트레이드오프를 기준으로 판단합니다. Fixed Window는 구현이 가장 단순하지만 경계 burst라는 치명적인 약점이 있습니다. 1분에 100번 허용이라면 59초에 100번, 다음 윈도우 첫 1초에 100번이 통과되면 2초 안에 200번이 처리되는 구간이 생겨 실질적으로 정책이 두 배까지 위반됩니다. Token Bucket은 버킷에 초당 N개씩 토큰을 채우고 요청마다 1개씩 소비하는 방식입니다. 버킷이 꽉 찬 상태에서는 평소에 적게 쓴 만큼 몰아서 요청을 처리할 수 있는 burst 허용이 특징입니다. API 특성상 일시적인 트래픽 급증을 허용해야 하는 서비스에 적합하며, Redis에서는 `{last_refill_time, token_count}` 두 필드를 Hash에 저장하고 Lua 스크립트로 refill과 차감을 원자적으로 처리합니다. Sliding Window Log는 Sorted Set에 요청마다 타임스탬프를 score로 저장하고, `ZREMRANGEBYSCORE`로 윈도우 밖 항목을 제거한 뒤 `ZCARD`로 현재 요청 수를 세는 방식입니다. Fixed Window의 경계 burst 문제가 없고 가장 정밀하지만, 모든 요청의 타임스탬프를 저장해야 하므로 트래픽이 많을수록 메모리 비용이 선형으로 증가합니다. 실무에서는 API Gateway에서 1차로 대략적으로 거르고, 결제나 인증처럼 중요한 엔드포인트는 Redis로 정밀하게 제어하는 조합을 씁니다. 분산 환경에서는 각 서버의 로컬 카운터를 합산할 수 없으므로 Redis를 글로벌 레이어로 두고 모든 서버가 같은 카운터를 참조해야 일관성이 보장됩니다.

**꼬리 질문 예시:**
- "분산 서버 환경에서 rate limit을 어떻게 공유하나요?" → Redis 중앙 집중식 카운터, 또는 각 서버 로컬 카운터 + 주기적 동기화(느슨한 제한 허용)
- "Redis rate limit에서 race condition은?" → Lua 스크립트로 원자적 처리(EVAL), 또는 INCR + EXPIRE 조합

**면접 세션 피드백 (2026-04-02 2회차)**:
- 현황: 처음 출제, 전혀 모름 → 신규 암기 최우선
- 암기 우선순위: Token Bucket(burst 허용) vs Sliding Window Log(정확하지만 메모리 비용) → Redis Sorted Set 구현 패턴

---

## 외부 API 장애 대응 — Circuit Breaker 패턴

### Q. OTA 파트너사 API가 갑자기 응답하지 않거나 5초 이상 지연된다면 어떻게 처리하시겠어요?

**난이도**: 중급

**핵심 키워드**: Connection Timeout, Read Timeout, Retry, Exponential Backoff, Circuit Breaker, Resilience4j, Fallback, DLQ, Graceful Degradation

**4단계 답변 구조 (암기):**

1. **Timeout 설정** — 무한 block 방지
   - Connection Timeout: 1초 (연결 수립 최대 대기)
   - Read Timeout: 3초 (응답 수신 최대 대기)
   - 설정 없으면 스레드 풀 전체가 외부 장애에 잠김

2. **Retry + Exponential Backoff** — 일시적 장애 대응
   - 최대 3회 재시도, 대기 시간: 1s → 2s → 4s
   - 네트워크 순간 불안정 처리
   - 재시도 대상: Timeout, 5xx 오류. 4xx는 재시도 불필요

3. **Circuit Breaker (Resilience4j)** — 반복 장애 시 자동 차단
   - Closed(정상) → Open(차단) → Half-Open(탐색) 3가지 상태
   - 실패율이 임계값(예: 50%) 초과 시 Open으로 전환 → 요청 즉시 거부
   - Half-Open에서 시험 요청 성공 시 Closed 복귀
   - 파트너사 장애가 내부 서비스 전체로 전파되는 것을 방지

4. **Fallback** — 사용자에게 Graceful Degradation
   - Circuit Open 상태에서 실행되는 대체 로직
   - 예: "파트너사 일시 서비스 불가" 안내, 캐시된 재고 데이터 반환, 대기열 등록

**Resilience4j 설정 예시 (Spring Boot):**
```java
@CircuitBreaker(name = "otaApi", fallbackMethod = "otaFallback")
@Retry(name = "otaApi")
public OtaResponse callOtaApi(String partnerId) { ... }

public OtaResponse otaFallback(String partnerId, Exception e) {
    return OtaResponse.unavailable("파트너사 일시 불가");
}
```

**추가 운영 포인트:**
- 실패 이력 저장: REQUIRES_NEW 별도 트랜잭션으로 API·예외유형·빈도 추적
- 알람: 실패 2회 이상 → 개발자 알림 (CloudWatch, Slack 등)
- checked exception rollbackFor: `IOException`/`SocketTimeoutException`은 기본 롤백 안 됨 — 명시 필요

**꼬리 질문 예시:**
- "Circuit Breaker의 3가지 상태를 설명해주세요."
- "Retry와 Circuit Breaker를 함께 쓸 때 순서는?"  → Retry 먼저 → 실패율 축적 → Circuit Open
- "외부 API 호출 실패 시 내부 트랜잭션은 어떻게 처리하나요?" → 트랜잭션 분리 + rollbackFor

**면접 세션 피드백 (2026-04-07 3회차)**:
- 잘한 점: 트랜잭션 분리, Retry + Exponential Backoff, rollbackFor, 실패 이력 저장 정확
- 공통 누락: **Timeout 설정(1단계)**, **Circuit Breaker(Resilience4j)**, **Fallback 전략** — 이 3가지를 반드시 추가

---

---

## API Rate Limiting을 구현할 때 Token Bucket과 Sliding Window Log의 차이를 설명하고, Redis로 각각 어떻게 구현하는지 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Token Bucket, Sliding Window Log, Fixed Window, burst, ZADD, ZREMRANGEBYSCORE, ZCARD, Lua script, API Gateway

**모범 답변 (3분 이상 말하기 형태)**:
> Rate Limiting 구현 방식은 크게 Fixed Window, Token Bucket, Sliding Window Log 세 가지로 나뉩니다. 먼저 Fixed Window는 가장 단순한 방식으로 Redis의 INCR + EXPIRE 조합으로 구현합니다. 1분에 100번 허용이라면 1분 단위 키를 만들고 카운트를 올립니다. 단점은 경계 burst입니다. 59초에 100번, 다음 윈도우 시작 1초에 100번이 통과되면 2초 안에 200번이 허용되는 구간이 생겨 실질적으로 정책을 두 배 위반할 수 있습니다.
>
> Token Bucket은 버킷에 토큰을 초당 N개씩 충전하고, 요청이 올 때마다 토큰 1개를 소비하는 방식입니다. 버킷이 꽉 차면 토큰이 더 이상 쌓이지 않고, 토큰이 없으면 요청을 차단합니다. burst를 허용한다는 의미는, 평소에 요청이 적어서 토큰이 쌓여 있으면 짧은 시간에 몰아서 요청을 보낼 수 있다는 뜻입니다. API 특성상 일시적인 트래픽 급증이 허용되는 서비스에 적합합니다. Redis로는 Hash에 `{last_refill_time, token_count}` 두 필드만 저장하면 되고, Lua 스크립트로 refill 계산과 토큰 차감을 원자적으로 처리합니다. 유저당 2개 필드만 관리하면 되므로 메모리 비용이 낮습니다.
>
> Sliding Window Log는 Sorted Set에 요청마다 타임스탬프를 score로 저장합니다. `ZADD rate:user:999 {now_ms} {request_id}` 로 기록하고, `ZREMRANGEBYSCORE` 로 윈도우 밖 오래된 요청을 제거한 뒤, `ZCARD` 로 현재 윈도우 안의 요청 수를 셉니다. 이 값이 한도를 초과하면 차단합니다. Fixed Window의 경계 burst 문제가 없고 정밀하지만, 요청 건수만큼 Sorted Set 엔트리가 쌓여서 트래픽이 많을수록 메모리 비용이 높아집니다.
>
> 분산 서버 환경에서는 각 서버가 로컬 카운터를 가지면 합산이 안 되므로, Redis를 글로벌 레이어로 두어 모든 서버가 같은 카운터를 참조하게 합니다. 또는 Nginx나 API Gateway에 내장된 Rate Limit 기능을 사용하면 애플리케이션 코드 없이 처리할 수 있습니다. 실무에서는 API Gateway에서 1차로 대략적으로 거르고, 결제나 인증처럼 중요한 엔드포인트는 Redis로 정밀하게 제어하는 조합을 씁니다.

**꼬리 질문 예시**:
- Token Bucket의 refill rate와 버킷 크기를 각각 어떻게 결정하나요?
- Sliding Window Counter(카운터 기반)와 Sliding Window Log의 차이는 무엇인가요?
- Lua 스크립트를 쓰는 이유가 무엇인가요? (원자성 — INCR + EXPIRE를 분리하면 race condition 발생)

> 출처: 2026-04-10 면접 세션 5회차

---

## 작성 예정

- 대용량 파일 업로드 시스템 설계 질문
- 분산 스케줄러 설계 질문
