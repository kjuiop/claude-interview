---
tags: [system-design, architecture, real-time, interview-questions]
related: [redis, kafka, kubernetes, golang, distributed-systems]
---

# System Design — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/system-design/concepts]]

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

**모범 답변 구조:**
기술 선택 + 이유 → Redis pub/sub(빠른 브로드캐스트, 유실 허용 + MongoDB fallback) → MongoDB(스키마 유연성 + 인덱스) → K8s(HPA + graceful shutdown)

---

## 10만 동시접속 채팅 서버를 설계한다면?

**난이도**: 심화

**핵심 키워드**: WebSocket 수평 확장, Redis pub/sub, Hot Partition, Consistent Hashing, Sequence Number, 재연결, Graceful Shutdown

**모범 답변 방향 (전체 흐름):**

1. **용량 계산 먼저**: 서버 1대 ≈ 5~8만 커넥션 안정 처리. 10만이면 3대(장애 대비). 메모리보다 CPU가 병목.

2. **핵심 아키텍처**:
   - WebSocket 서버 (Go): goroutine 2KB, 대규모 커넥션 병렬 처리
   - Redis pub/sub: 서버 간 브로드캐스트 (채팅방 = topic)
   - MongoDB: 메시지 영속 저장 + 재연결 시 히스토리 fetch
   - K8s: CPU/active connection 기반 HPA + Graceful Shutdown

3. **트레이드오프 명시**:
   - Redis pub/sub → 유실 허용, 재연결 시 MongoDB fallback
   - 히스토리/순서 보장 필요하면 Kafka로 전환

4. **심화 포인트 (꼬리 질문 대비)**:
   - Hot Partition → Consistent Hashing + 채팅방 자동 분할
   - 메시지 순서 → room_id 기반 Kafka 파티셔닝 또는 Sequence Number
   - 재연결 → lastSeq 기억 → MongoDB에서 미수신 메시지 보충 + 지수 백오프

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

**Token Bucket:**
- 버킷에 토큰이 일정 속도로 채워짐 (예: 초당 10개)
- 요청 1건 = 토큰 1개 소비. 토큰 없으면 요청 거부
- 장점: 순간 burst 허용 (버킷에 쌓인 토큰만큼), 구현 단순
- 단점: 정확한 요청 수 추적 어려움

**Sliding Window Log:**
- 요청마다 타임스탬프를 저장
- 현재 시간 기준 윈도우(예: 1분) 내 요청 수 카운트 → 초과 시 거부
- 장점: 정확한 요청 수 추적, 경계 시점 burst 없음
- 단점: 모든 요청 타임스탬프 저장 → 메모리 비용 높음

**Fixed Window 비교:**
- 가장 단순하지만 경계 시점에 2배 burst 허용 (59초 + 00초에 각 100개)

**실무 구현:**
```
Token Bucket → Redis + Lua 스크립트 (원자적 처리)
Sliding Window Log → Redis Sorted Set (score = timestamp)
  ZADD key timestamp requestId
  ZREMRANGEBYSCORE key 0 (now - window)
  ZCARD key  → 현재 요청 수
API Gateway → Nginx, Kong, AWS API Gateway 내장 rate limiter 활용
```

**트레이드오프 한 문장:**
> "Token Bucket은 burst 허용과 단순 구현이 장점, Sliding Window Log는 정확하지만 메모리 비용. 실무에서는 Redis Sorted Set으로 Sliding Window를 구현하거나 API Gateway 내장 기능 활용."

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
