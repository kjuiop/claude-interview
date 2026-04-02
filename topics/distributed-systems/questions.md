---
tags: [distributed-systems, cap-theorem, consistency, interview-questions]
related: [kafka, redis, zookeeper, kubernetes, mysql, postgresql, system-design]
---

# Distributed Systems — 면접 질문

→ [[home]] | 개념 정리: [[topics/distributed-systems/concepts]]

---

## CAP 정리(CAP Theorem)란 무엇인가요? 왜 2개만 선택할 수 있나요?

**난이도**: 기초

**핵심 키워드**: Consistency, Availability, Partition Tolerance, CP, AP, ZooKeeper, Cassandra

**모범 답변 방향**:
- **C (Consistency)**: 모든 노드가 동시에 같은 데이터를 봄. 쓰기 후 읽기 시 항상 최신값 반환
- **A (Availability)**: 모든 요청에 응답 반환 (오류 없이). 단, 최신 데이터가 아닐 수 있음
- **P (Partition Tolerance)**: 네트워크 단절이 발생해도 시스템이 계속 동작
- **왜 2개만**: 네트워크 파티션은 실제로 반드시 발생 → P는 포기 불가 → CP 또는 AP만 선택
- **CP 예시**: ZooKeeper, HBase — 파티션 시 일관성 유지, 일부 응답 거부
- **AP 예시**: Cassandra, DynamoDB — 파티션 시 계속 응답, stale 데이터 가능
- **이력서 연결**: ZooKeeper는 CP 시스템. 리더 선출 중 쓰기 불가 → 일관성 우선

**꼬리 질문 예시**:
- Redis는 CAP에서 어느 쪽인가요? (AP에 가까움 — 파티션 시 stale 데이터 가능)
- Kafka는 CAP 관점에서? (CP — 리더 복제 완료 후 커밋)

**면접 세션 피드백 (2026-04-01)**:
- 현황: 전혀 모르는 상태. CP/AP 선택 구조와 이력서의 ZooKeeper 연결 표현 우선 암기 필요
- 모범 답변: "분산 시스템에서 C·A·P를 동시에 보장할 수 없습니다. 네트워크 파티션은 피할 수 없으므로 P는 필수, 결국 C와 A 중 하나를 선택합니다. ZooKeeper는 CP 선택으로 리더 선출 중 쓰기를 거부해 일관성을 보장합니다."

---

## Saga Pattern — Choreography vs Orchestration

**난이도**: 심화

**핵심 키워드**: Saga Pattern, Choreography, Orchestration, Compensating Transaction, 멱등성(Idempotency), Saga Log Table, 최종 일관성(Eventual Consistency)

**모범 답변 방향**:

**왜 필요한가 — 분산 트랜잭션 문제**:
- MSA에서 DB가 서비스별로 분리되면 하나의 비즈니스 로직이 여러 로컬 트랜잭션으로 분리됨
- 결제 서버 네트워크 오류 → 재고는 차감됐는데 결제는 안 됨 → 데이터 정합성 붕괴
- DB 샤딩, 분산 모놀리스도 동일 문제 발생

**Saga 패턴 핵심 구조**:
- n개 연산 → 각 서비스의 로컬 트랜잭션으로 순서대로 실행
- 실패 시 실패 지점부터 역순으로 보상 트랜잭션 실행 → **n개 작업이면 보상 포함 최대 2n개 준비 필요**
- 이벤트 기반 통신 + **Transactional Messaging(Outbox Pattern)** 권장 — 비즈니스 로직 커밋과 이벤트 발행을 원자적으로 처리

**Choreography (안무형)**:
- 각 서비스가 이벤트를 발행하고, 다른 서비스가 이를 구독해 자율적으로 반응
- 결합도 낮음, 단순한 Saga에 적합
- 단점: 전체 흐름 추적 어려움(조용한 실패), 순환 의존성 위험(A→B→C→A)

**Orchestration (지휘형)**:
- 중앙 Saga Orchestrator가 각 서비스를 순서대로 호출하고 실패 처리
- 디버깅 쉬움, 복잡한 비즈니스 흐름에 적합 — **대부분의 상황에서 권장**
- 핵심: **Orchestrator는 순서화만 담당, 비즈니스 로직을 갖지 않아야 함** (로직이 집중되면 단일 장애점)
- 단점: Orchestrator 결합도, 롤백 실패 시 재시도/알림 로직 별도 구현 필요

**보상 트랜잭션 설계 원칙**:
1. **멱등성(Idempotency) 필수**: 네트워크 재시도로 같은 이벤트가 중복 처리될 수 있음
   ```kotlin
   fun cancelPayment(paymentId: String) {
       if (payment.status == CANCELLED) return  // 이미 취소됨 → 무시
       payment.cancel()
   }
   ```
2. **Saga Log Table**: 별도 상태 테이블에 각 단계 기록 → 실패 시 어느 단계까지 완료됐는지 확인 후 역순으로 보상 트랜잭션 실행

**Compensating Workflow (보상 워크플로우)**:
- 기술적으로 보상 트랜잭션이 정상 동작해도, 사용자 경험은 불쾌할 수 있음
  (예: 재고 있다고 주문했는데 결제 실패로 취소 → 사용자 불만)
- 해결책: 비즈니스 정책으로 보완 — 재고 알림 등록, 할인 쿠폰 제공 등
- Saga의 기술적 보상 ≠ 고객 만족 보상, 두 레이어를 분리해서 설계해야 함

**최종 일관성(Eventual Consistency)과 격리 문제**:
- Saga는 로컬 트랜잭션이 분리되므로 강한 일관성(Strong Consistency) 불가
- 중간 상태를 다른 트랜잭션이 읽을 수 있음 → Lost Update, Dirty Read 가능
- 대응: 낙관적 락, 비격리 대책 전략 준비 필요

**Choreography 추적 보완 전략**: AOP 로깅, 분산 추적(Jaeger/Zipkin), 알림

**이력서 연결**: Feature Flag + ZooKeeper 무중단 마이그레이션 경험을 Saga의 단계적 롤백 전략과 연결 가능

**꼬리 질문 예시**:
- "2PC와 Saga의 차이는?" → 2PC는 동기 락(강한 일관성), Saga는 비동기 보상(최종 일관성). 마이크로서비스에서 2PC는 성능/가용성 문제로 지양.
- "Saga에서 동시성 이슈는?" → 다른 트랜잭션이 중간 상태를 읽을 수 있음. Lost Update 방지를 위해 각 서비스에서 낙관적 락 사용.
- "Orchestrator가 비즈니스 로직을 가지면 안 되는 이유는?" → 모든 서비스가 Orchestrator에 의존하게 되어 단일 장애점이 되고 변경 파급 범위가 커짐.

**면접 세션 피드백 (2026-04-02 1회차)**:
- 잘한 점: Choreography/Orchestration 구조 정확히 구분. Orchestration 결합도 트레이드오프 명확히 설명. Choreography 단점에 AOP/Observability 해결책까지 제시(시니어 수준).
- 보완:
  - Choreography 추가 단점: 순환 의존성 미언급
  - 보상 트랜잭션 멱등성(Idempotency) 반드시 언급 필요
  - Saga Log Table 패턴으로 추적 방식 공식화

> 출처: https://monday9pm.com/분산-트랜잭션-distributed-transaction-알아보기-d0a10ad5dd53

---

## 분산 트랜잭션 해결 방법 3가지(2PC, TC/C, Saga)를 비교하고 MSA에서 무엇을 선택하겠습니까?

**난이도**: 심화

**핵심 키워드**: 2PC, TC/C, Saga, 강한 일관성, 최종 일관성, 보상 트랜잭션, 코디네이터, 성능

**모범 답변 방향**:

| 항목 | 2PC | TC/C | Saga |
|---|---|---|---|
| 방식 | 글로벌 트랜잭션 (동기 락) | Try-Confirm/Cancel (보상) | 로컬 트랜잭션 연속 + 보상 |
| 일관성 | 강한 일관성 | 최종 일관성 | 최종 일관성 |
| 성능 | 매우 낮음 (단일 DB 대비 ~10배 저하) | 중간 | 높음 |
| 장애 시 영향 | 전체 시스템 락 → 코디네이터 다운 시 전체 마비 | Coordinator 재시도/타임아웃 처리 필요 | 서비스별 독립 실패 처리 |
| 구현 복잡도 | 낮음 (XA 표준 사용) | 중간 (Try/Confirm/Cancel 상태 구현) | 높음 (보상 트랜잭션 2n개 준비) |
| MSA 적합성 | 비적합 (가용성 문제) | 적합 (REST 기반 단순 분산 트랜잭션) | 사실상 표준 |

**2PC 상세**:
- **1단계 (Prepare)**: 코디네이터가 모든 참여 노드에 "커밋 가능?" 요청 → 모두 YES면 진행
- **2단계 (Commit/Abort)**: 전체 YES → 커밋, 하나라도 NO → 전체 중단
- **왜 MSA에서 안 쓰나**: 커밋 전까지 전체 노드에 락 → 코디네이터 장애 시 전체 블록. 단일 DB 대비 ~10배 성능 저하 (네트워크 IO + fsync 추가)

**TC/C (Try-Confirm/Cancel) 상세**:
- **Try**: 모든 서비스에 자원 예약 요청 (실제 커밋 X, 보류 상태)
- **Confirm**: 전체 Try 성공 → 각 서비스에 실제 작업 실행 요청
- **Cancel**: Try 중 실패 또는 타임아웃 → 역순으로 예약 취소
- 2PC와 달리 각 서비스가 별도 트랜잭션 — 락 범위 축소
- REST 기반 단순 분산 트랜잭션에 적합, 단 상태 관리 테이블 + 피봇(재시도) 로직 구현 필요

**Saga 선택 이유 (MSA 표준)**:
- 이벤트 기반 비동기 통신 → 서비스 간 느슨한 결합
- 각 서비스가 독립적으로 실패 처리 → 전체 시스템 장애 전파 없음
- Outbox Pattern과 결합해 이벤트 발행 신뢰성 확보
- 단점: 2n개 보상 트랜잭션 준비 부담, 최종 일관성으로 중간 상태 노출

**꼬리 질문 예시**:
- 2PC에서 코디네이터가 2단계 커밋 도중 다운되면 어떻게 되나요?
- TC/C에서 Confirm 요청을 보낸 후 일부 서비스가 응답하지 않으면 어떻게 처리하나요?
- Saga와 Outbox Pattern을 함께 쓰는 이유는 무엇인가요?

> 출처: https://monday9pm.com/분산-트랜잭션-distributed-transaction-알아보기-d0a10ad5dd53

---

## Transactional Outbox Pattern에서 Polling vs CDC 방식의 설계 판단

**난이도**: 심화

**핵심 키워드**: Outbox Table, Debezium CDC, Idempotency Key, At-Least-Once, Two Generals' Problem

**모범 답변 방향**:

**동작 원리**:
- **공통**: 주문 저장 + Outbox 테이블 insert를 동일 DB 트랜잭션으로 묶음 → 원자성 보장
- **Polling Publisher**: 별도 프로세스가 Outbox 테이블을 주기적으로 SELECT → 미발행 이벤트 Kafka 발행 → 발행 완료 후 DELETE/status 업데이트
- **CDC (Debezium)**: DB의 Binlog(MySQL) / WAL(PostgreSQL)을 읽어 변경사항을 실시간으로 Kafka로 스트리밍. Outbox 테이블 insert 감지 즉시 발행

**각 방식 비교**:

| 항목 | Polling Publisher | CDC (Debezium) |
|---|---|---|
| 구현 복잡도 | 낮음 (SQL + 스케줄러) | 높음 (Debezium, Kafka Connect 설정) |
| 지연 시간 | 폴링 주기만큼 지연 | 준실시간 |
| DB 부하 | SELECT 폴링 부하 존재 | Binlog 읽기 (복제 연결) |
| 적합 상황 | 소규모, 지연 허용 가능 | 대용량, 실시간 필요, 고가용성 |
| 장애 복구 | 재시작 시 미처리 행 재조회 | Kafka offset으로 재처리 위치 보장 |

**중복 메시지 처리 (At-Least-Once → 수신 측 멱등성)**:
- 각 이벤트에 고유 `idempotency_key` (= outbox row UUID) 포함
- 수신 서비스에서 처리 전 `processed_events` 테이블에 key 존재 여부 확인
- key 없으면 처리 + insert, key 있으면 스킵 → exactly-once 의미론 구현
- Redis로 최근 처리 key 캐싱 → DB 조회 부하 감소

**한계: 두 장군 문제와의 관계**:
- Outbox 패턴은 "내가 발행했다"는 사실은 보장하지만, "수신 서비스가 처리했다"는 확인은 보장 불가
- 두 장군 문제처럼 네트워크 장애 시 발행 확인 응답 자체가 유실될 수 있음
- 결국 At-Least-Once + 수신 측 멱등성으로 실용적 타협. 완전한 Exactly-once는 동일 DB 내 2PC 없이는 이론적으로 불가능

**꼬리 질문 예시**:
- Debezium을 사용할 때 Binlog retention 기간이 짧으면 어떤 문제가 생기나요?
- Outbox 테이블이 계속 쌓이면 어떻게 관리하나요?
- Saga 패턴과 Outbox 패턴은 어떻게 함께 사용되나요?

---

## 메시지 전달 보장 수준(At-most-once, At-least-once, Exactly-once)의 차이와 실무 적용

**난이도**: 중급

**핵심 키워드**: At-most-once, At-least-once, Exactly-once, 두 장군 문제, Effectively-once, 멱등성, 재처리 전략, 중복 제거

**모범 답변 방향**:

**3가지 전달 시멘틱**:

| 수준 | 의미 | 특징 | 사용 예 |
|---|---|---|---|
| **At-most-once** | 최대 1회 (유실 허용) | 재시도 없음. 가장 단순 | 로그 수집, 분석 이벤트 |
| **At-least-once** | 최소 1회 (중복 허용) | 재시도 + ACK. 중복 제거 필요 | 대부분의 비즈니스 이벤트 |
| **Exactly-once** | 정확히 1회 | 이론적으로 달성 불가능 (두 장군 문제) | — |

**두 장군 문제 (Two Generals' Problem)**:
- 신뢰할 수 없는 네트워크에서 두 시스템이 완전한 합의를 이루는 것은 이론적으로 불가능
- "메시지 전달 확인" → "확인의 확인" → 무한 재귀 → 결국 100% 보장 불가
- TCP 4-way handshake도 이 문제에서 자유롭지 않음 (Half Open Connection)
- **결론**: 완전한 Exactly-once는 불가능. 확률적으로 99%에 근접시키는 것이 현실적 목표

**Effectively-once — 실무적 해법**:
At-least-once + 아래 전략의 조합으로 Exactly-once에 *근접*한 보장 달성:

| 전략 | 역할 | 구현 주체 |
|---|---|---|
| **재처리 전략(Redelivery)** | 미전달 메시지 재전송 — Outbox Pattern 활용 | Producer |
| **타임아웃** | 재시도 횟수/시간 제한, 무한 재시도 방지 | Producer |
| **멱등성(Idempotency)** | 동일 메시지 중복 처리 차단 — 고유 ID 기반 | Producer + Consumer |
| **중복 제거(Deduplication)** | Consumer가 처리한 메시지 ID 보관 후 스킵 | Consumer |
| **순서 보장(Ordering)** | 시퀀스 번호 추적, 누락 시 재요청 | Producer + Consumer |

**실무 예시** — 주문 완료 후 이메일 발송:
- 이메일 서비스가 발송 성공 후 응답 유실 → 주문 서비스가 실패로 인식 → 재시도 → 이메일 중복 발송
- 해결: 요청마다 고유 `request_id` 부여 → 이메일 서비스가 동일 ID 수신 시 스킵 (중복 제거)

**꼬리 질문 예시**:
- Kafka Producer의 `enable.idempotence=true`는 어떤 수준의 중복을 방지하나요? Consumer까지 포함한 Exactly-once를 위해 추가로 필요한 것은?
- 두 장군 문제가 TCP에서도 완전히 해결되지 않는 이유를 4-way handshake를 예로 설명해주세요.
- At-least-once 환경에서 결제 중복 처리를 방지하는 실무 설계를 제시해주세요.

> 출처: https://monday9pm.com/메시지-전달-전략과-두-장군-문제-message-delivery-semantics-and-two-generals-problem-f8f1c7646c0b
> 출처: https://velog.io/@eastperson/Transaction-Outbox-Pattern-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0

---

## Graceful Shutdown이 무엇이고, 왜 필요하며, 어떻게 구현하나요?

**난이도**: 중급

**핵심 키워드**: SIGTERM, SIGKILL, Connection Draining, terminationGracePeriodSeconds, preStop Hook, 무중단 배포

**모범 답변 방향**:
- **정의**: 프로세스가 종료 신호를 받았을 때 즉시 강제 종료하지 않고, 진행 중인 요청/작업을 완료한 뒤 안전하게 종료하는 방식
- **필요 이유**: 배포/스케일 다운 시 처리 중인 요청이 강제 중단되면 데이터 불일치, 커넥션 오류, 메시지 유실 발생

**동작 흐름 (Kubernetes 기준)**:
```
1. Pod 종료 요청 발생
2. kubelet이 컨테이너에 SIGTERM 전송
3. 앱이 SIGTERM 감지 → 신규 요청 수신 중단 (Liveness 유지)
4. 진행 중인 요청 처리 완료 대기 (Connection Draining)
5. DB 커넥션 풀, 메시지 큐 컨슈머 등 리소스 정리 후 종료
6. terminationGracePeriodSeconds(기본 30초) 초과 시 SIGKILL로 강제 종료
```

**구현 시 주의점**:
- Dockerfile에서 `CMD`/`ENTRYPOINT`는 **Exec form** (`["app"]`) 사용 — Shell form이면 PID 1이 sh가 되어 SIGTERM이 앱에 전달 안 됨
- `preStop` 훅: SIGTERM 전에 실행. `sleep 5` 등으로 로드밸런서가 해당 Pod를 트래픽에서 제외할 시간 확보
- `terminationGracePeriodSeconds`는 처리 최대 시간 + 여유를 감안해 설정

**언어별 구현 (핵심)**:
- **Go**: `signal.NotifyContext` 또는 `signal.Notify(ch, syscall.SIGTERM)` → goroutine에서 shutdown 트리거
- **Java(Spring Boot)**: `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`
- **공통**: HTTP 서버는 새 연결 거부 후 기존 연결 처리 완료 후 종료

**꼬리 질문 예시**:
- Graceful Shutdown 중에도 Liveness Probe는 통과해야 하나요, 실패해야 하나요?
- Kafka Consumer를 Graceful Shutdown할 때 특별히 고려해야 할 것은 무엇인가요?
- `terminationGracePeriodSeconds`보다 처리 시간이 길 경우 어떻게 대응하나요?

> 출처: https://kkamji.net/posts/pod-graceful-shutdown/
> 출처: https://harrisonjung.medium.com/graceful-shutdown-96215ffe6b1b

---

## 동기/비동기, Blocking/Non-Blocking의 차이와 4가지 조합을 설명해주세요.

**난이도**: 중급

**핵심 키워드**: 제어권 반환, 결과 확인 주체, I/O 대기, Callback, Future/Promise

**모범 답변 방향**:

**핵심 구분 기준**:
- **Blocking/Non-Blocking**: 호출된 함수가 **제어권을 즉시 반환하는가** — 호출자가 대기하느냐의 문제
- **동기/비동기**: 결과를 **누가 확인하는가** — 호출자가 직접 기다리느냐(동기) vs 완료 알림을 받느냐(비동기)

**4가지 조합**:

| 조합 | 동작 | 실무 예시 |
|---|---|---|
| **동기 + Blocking** | 호출 후 결과 올 때까지 대기, 제어권도 없음 | 일반 JDBC 쿼리, `Thread.sleep()` |
| **동기 + Non-Blocking** | 제어권 즉시 반환받지만 결과를 직접 폴링 | `Future.isDone()` 반복 확인 |
| **비동기 + Blocking** | 제어권 없이 대기, 결과는 콜백으로 처리 | 잘못 구현된 비동기 (실질적으로 의미 없음) |
| **비동기 + Non-Blocking** | 제어권 즉시 반환, 결과는 콜백/이벤트로 알림 | Netty, WebFlux, Node.js, Kafka Consumer |

**실무 핵심**:
- **Spring MVC**: 동기 + Blocking — 스레드 하나당 요청 하나 처리. 스레드 고갈 위험
- **Spring WebFlux**: 비동기 + Non-Blocking — 적은 스레드로 대량 요청 처리 (Netty 기반)
- **Node.js**: 비동기 + Non-Blocking — 싱글 스레드 + Event Loop

**꼬리 질문 예시**:
- 동기 + Non-Blocking 조합이 실용적이지 않은 이유는 무엇인가요?
- Spring WebFlux를 선택해야 하는 상황과 그렇지 않은 상황을 구분해주세요.
- I/O Bound 작업과 CPU Bound 작업에서 각각 어떤 I/O 모델이 유리한가요?

> 출처: https://choi-geonu.medium.com/백엔드-개발자들이-알아야할-동시성-2
> 출처: https://velog.io/@nittre/블로킹-Vs.-논블로킹-동기-Vs.-비동기

---

## Circuit Breaker 패턴이 무엇이고 MSA에서 왜 필요한가요?

**난이도**: 중급

**핵심 키워드**: Closed/Open/Half-Open, 장애 전파 방지, Fail Fast, Fallback, resilience4j

**모범 답변 방향**:

**왜 필요한가**:
- MSA에서 A → B → C 서비스 호출 체인에서 C가 느려지면 → B의 스레드가 모두 대기 → A도 연쇄 지연 → **Cascading Failure(장애 전파)**
- Circuit Breaker는 특정 서비스가 반복 실패할 때 빠르게 차단(Fail Fast)해 장애 전파 방지

**3가지 상태**:

```
[Closed] → 실패율 임계치 초과 → [Open] → 일정 시간 후 → [Half-Open]
   ↑ 성공률 회복                                              ↓ 일부 요청 허용
   └──────────────────── 성공 ────────────────────────────────┘
                                         └── 여전히 실패 → [Open]
```

| 상태 | 동작 |
|---|---|
| **Closed** | 정상. 모든 요청 통과. 실패율 측정 중 |
| **Open** | 차단. 모든 요청 즉시 실패 응답(Fail Fast). 타임아웃 후 Half-Open으로 |
| **Half-Open** | 일부 요청만 통과. 성공 시 Closed 복귀, 실패 시 다시 Open |

**주요 설정값**:
- `failureRateThreshold`: Open으로 전환할 실패율 (%) — 예: 50%
- `waitDurationInOpenState`: Open 유지 시간 — 예: 60초
- `permittedNumberOfCallsInHalfOpenState`: Half-Open 시 허용 요청 수

**Fallback 전략**:
- Open 상태에서 기본값 반환, 캐시 데이터 반환, 에러 응답 등으로 우아하게 처리
- `resilience4j` (Java), `hystrix` (deprecated) 라이브러리 활용

**꼬리 질문 예시**:
- Circuit Breaker와 Retry를 함께 쓸 때 순서는 어떻게 해야 하나요?
- Half-Open에서 테스트 요청이 실패하면 즉시 Open으로 돌아가야 하나요?
- Circuit Breaker 상태를 여러 인스턴스 간에 공유하려면 어떻게 하나요?

> 출처: https://hudi.blog/circuit-breaker-pattern/
> 출처: https://seongwon.dev/MSA/20230426-서킷브레이커란/

---

## Rate Limiting 알고리즘을 비교하고 분산 환경에서 어떻게 구현하나요?

**난이도**: 중급

**핵심 키워드**: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window, Redis, 분산 Rate Limiter

**모범 답변 방향**:

**4가지 알고리즘 비교**:

| 알고리즘 | 원리 | 버스트 허용 | 구현 복잡도 | 적합 상황 |
|---|---|---|---|---|
| **Fixed Window** | 시간 창 단위로 요청 수 카운트 | 창 경계에서 2배 버스트 가능 | 낮음 | 단순한 API 보호 |
| **Sliding Window** | 현재 시점 기준 과거 N초 요청 수 카운트 | 없음 | 중간 | 정밀한 제한 필요 시 |
| **Token Bucket** | 토큰이 주기적으로 채워지고 요청마다 토큰 소비 | 버킷 크기만큼 허용 | 중간 | 버스트 허용, 일반적 API |
| **Leaky Bucket** | 큐에 요청을 쌓고 일정 속도로 처리 | 없음 (큐 초과 시 드롭) | 낮음 | 일정한 처리율 필요 시 |

**Fixed Window 취약점 예시**:
- 1분 100회 제한: 00:59~01:01에 각 100회 → 2초 안에 200회 처리됨

**분산 환경 구현 (Redis 활용)**:
- **Token Bucket**: Redis에 토큰 수와 마지막 충전 시각 저장 → Lua Script로 원자적 처리
- **Sliding Window**: Redis Sorted Set으로 요청 타임스탬프 관리 → 오래된 항목 자동 제거
- 단일 Redis 노드: SETNX + INCR → 원자성 보장
- Redis Cluster: 사용자/IP별 키를 동일 슬롯에 배치하거나 중앙 Rate Limiter 서비스 분리

**실무 도구**: AWS API Gateway, Nginx (limit_req), Spring Cloud Gateway, Bucket4j (Java)

**꼬리 질문 예시**:
- Token Bucket에서 버스트를 허용하면서도 평균 처리율을 제한하는 원리를 설명해주세요.
- Rate Limiter를 적용할 때 사용자 ID 기준과 IP 기준의 차이와 각각의 한계는?
- Redis Cluster에서 Lua Script 기반 Rate Limiter의 주의점은?

> 출처: https://www.mimul.com/blog/about-rate-limit-algorithm/
> 출처: https://api7.ai/blog/rate-limiting-guide-algorithms-best-practices

---

## MSA에서 Resilience(복원력)를 높이기 위한 패턴들을 설명해주세요.

**난이도**: 심화

**핵심 키워드**: Circuit Breaker, Retry, Timeout, Bulkhead, Rate Limiter, Fallback, resilience4j

**모범 답변 방향**:

**Resilience 핵심 패턴**:

| 패턴 | 역할 | 설정 포인트 |
|---|---|---|
| **Timeout** | 응답 무한 대기 방지 → 빠른 실패 | 비즈니스 SLA 기반 설정 |
| **Retry** | 일시적 장애 재시도 — Exponential Backoff + Jitter | 재시도 횟수, 간격, 재시도 가능 예외 한정 |
| **Circuit Breaker** | 반복 실패 서비스 차단 → 장애 전파 방지 | 실패율 임계치, Open 유지 시간 |
| **Bulkhead** | 서비스별 스레드 풀/세마포어 격리 → 한 서비스 장애가 전체 영향 방지 | 스레드 풀 크기, 대기 큐 |
| **Rate Limiter** | 요청 속도 제한 → 과부하 방지 | TPS 임계치 |
| **Fallback** | 장애 시 기본값/캐시 반환 → 우아한 저하(Graceful Degradation) | 비즈니스 정책 |

**Retry + Circuit Breaker 조합 순서**:
```
요청 → [Rate Limiter] → [Retry(3회)] → [Circuit Breaker] → 외부 서비스
```
- Retry가 Circuit Breaker 안쪽에 있으면 재시도가 실패율 카운트에 포함됨
- Circuit Breaker 바깥에 Retry를 두면 Open 상태에서도 재시도 → 비효율
- 권장: **Circuit Breaker → Retry** 순서로 래핑

**Bulkhead 패턴**:
- Thread Pool Isolation: 서비스별 스레드 풀 분리 → A 서비스 스레드 고갈이 B에 영향 없음
- Semaphore Isolation: 동시 요청 수 제한 (경량, 타임아웃 강제 어려움)

**Exponential Backoff + Jitter**:
- 재시도 간격: 1s → 2s → 4s → 8s (지수 증가)
- Jitter: 랜덤 추가 → Thundering Herd 방지 (모든 인스턴스가 동시에 재시도하는 문제)

**꼬리 질문 예시**:
- Bulkhead 없이 Hystrix만 쓰면 어떤 문제가 남아있나요?
- Fallback 응답이 항상 옳은가? 오래된 캐시 데이터를 반환하는 리스크는?
- resilience4j에서 여러 패턴을 체이닝할 때 어떤 순서가 권장되나요?

> 출처: https://learn.microsoft.com/ko-kr/azure/architecture/patterns/circuit-breaker
> 출처: https://hudi.blog/circuit-breaker-pattern/

---
