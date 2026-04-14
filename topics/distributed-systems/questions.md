---
tags: [distributed-systems, cap-theorem, consistency, interview-questions]
related: [kafka, redis, zookeeper, kubernetes, mysql, postgresql, system-design]
---

# Distributed Systems — 면접 질문

→ [[home]] | 개념 정리: [[topics/distributed-systems/concepts]]

---

## 2PC (Two-Phase Commit)

### Q. 2PC 동작 원리와 Blocking 문제, 실무에서 Saga 패턴을 선택하는 기준을 설명해주세요.

**등장인물:** Coordinator(진행자) + Participant(각 DB/서비스)

**Phase 1 — Prepare:**
```
Coordinator → 모든 Participant: "커밋 가능하냐?"
Participant → 트랜잭션 준비 + 락 잡고 대기
Participant → Coordinator: "OK" or "NO"
```

**Phase 2 — Commit:**
```
모두 OK → Coordinator: "커밋해라"
하나라도 NO → Coordinator: "롤백해라"
```

**Blocking 문제 (핵심 약점):**
- 모든 Participant가 OK를 보낸 상태에서 **Coordinator가 죽으면**
- Participant들이 락을 잡은 채로 커밋/롤백 결정 못함 → 무한 대기
- 다른 트랜잭션이 같은 데이터 접근 시 전부 대기 → 처리량 급락

**3PC:** Coordinator 없이 Participant끼리 결정할 수 있는 단계 추가 → Blocking 부분 해결. 네트워크 분리 상황에서는 여전히 불일치 가능.

**2PC vs Saga 선택 기준:**

| | 2PC | Saga |
|---|---|---|
| 락 | 전체 트랜잭션 동안 유지 | 없음 |
| Coordinator 장애 | Blocking | 영향 없음 |
| 일관성 | 강한 일관성(ACID) | 최종 일관성 |
| 롤백 | 자동 | 보상 트랜잭션 직접 구현 |
| 적합 환경 | 단일 DB, XA 지원 DB ≤2개, 짧은 트랜잭션 | MSA, 긴 트랜잭션, 가용성 우선 |

**면접 세션 피드백 (2026-04-12 4회차)**:
- 처음 접한 주제 — Phase 1/2 흐름 + Blocking 원인 암기 필요
- Saga 선택 이유(DB 부하) 방향은 알고 있었음 → 락 경합 + Blocking + MSA 환경으로 구체화 필요

---

## CAP 정리(CAP Theorem)란 무엇인가요? 왜 2개만 선택할 수 있나요?

**난이도**: 기초

**핵심 키워드**: Consistency, Availability, Partition Tolerance, CP, AP, ZooKeeper, Cassandra

**모범 답변**:
CAP 정리는 분산 시스템에서 Consistency, Availability, Partition Tolerance 세 가지 속성을 동시에 모두 보장할 수 없다는 이론입니다. 각 속성의 의미를 정확히 구분하는 것이 중요합니다. Consistency는 모든 노드가 동시에 동일한 데이터를 보는 것으로, 쓰기 이후의 읽기는 반드시 최신값을 반환해야 합니다. Availability는 어떤 요청에도 오류 없이 응답을 돌려준다는 것이고, 다만 그 값이 최신값이 아닐 수 있습니다. Partition Tolerance는 네트워크 단절이 발생하더라도 시스템이 계속 동작하는 능력입니다. 세 가지 중 두 개만 선택할 수밖에 없는 이유는 P의 특수성 때문입니다. 실제 분산 환경에서 네트워크 파티션은 언제든 발생할 수 있고, 이를 아예 포기한 시스템은 분산 시스템이라 부르기 어렵습니다. 따라서 P는 사실상 필수이고, 결국 선택지는 CP와 AP 두 가지가 됩니다. CP 시스템인 ZooKeeper나 HBase는 파티션이 발생하면 일부 요청에 응답을 거부하더라도 데이터 일관성을 지킵니다. 반면 AP 시스템인 Cassandra나 DynamoDB는 파티션 상황에서도 계속 응답하지만, 노드 간 동기화가 완료되기 전의 stale 데이터를 반환할 수 있습니다. 샵라이브에서 실시간 라이브 스트리밍 서비스를 운영할 때 ZooKeeper를 사용했는데, 이것이 CP 시스템임을 이해하고 있었기 때문에 리더 선출 중 쓰기가 잠시 거부되는 상황을 예상하고 재시도 로직을 설계할 수 있었습니다. CP와 AP 중 어느 쪽을 선택하느냐는 서비스의 특성에 달려 있는데, 금융이나 예약처럼 데이터 정합성이 비즈니스 핵심인 경우에는 CP를, 좋아요 수나 조회수처럼 약간의 불일치가 허용되는 경우에는 AP를 선택하는 것이 일반적인 판단 기준입니다.

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
MSA에서 DB가 서비스별로 분리되면 하나의 비즈니스 로직이 여러 로컬 트랜잭션으로 분리됩니다. 결제 서버 네트워크 오류가 발생하면 재고는 이미 차감됐는데 결제는 실패하는 데이터 정합성 붕괴가 생기고, 2PC를 쓰면 전체 서비스에 락이 걸려 가용성을 희생해야 합니다. Saga 패턴은 이 문제를 n개 서비스의 로컬 트랜잭션을 순서대로 실행하고, 실패 시 실패 지점부터 역순으로 보상 트랜잭션을 실행해 최종 일관성을 확보하는 방식으로 해결합니다. 이벤트 기반 통신과 Outbox Pattern을 함께 사용해 비즈니스 로직 커밋과 이벤트 발행을 원자적으로 처리하는 것이 권장됩니다. 구현 방식은 Choreography(안무형)와 Orchestration(지휘형) 두 가지로 나뉩니다. Choreography는 각 서비스가 이벤트를 발행하고 다른 서비스가 구독해 자율적으로 반응하는 방식으로 결합도가 낮고 단순한 Saga에 적합하지만, 전체 흐름 추적이 어렵고 A→B→C→A 같은 순환 의존성 위험이 있습니다. Orchestration은 중앙 Saga Orchestrator가 각 서비스를 순서대로 호출하고 실패 처리를 담당해 디버깅이 쉽고 복잡한 비즈니스 흐름에 적합하며 대부분의 상황에서 권장됩니다. 단, Orchestrator는 순서화만 담당하고 비즈니스 로직을 갖지 않아야 단일 장애점이 되지 않습니다.

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

MSA에서 분산 트랜잭션을 처리하는 세 가지 방식은 각각 일관성과 성능 측면에서 뚜렷한 트레이드오프를 가집니다. 2PC는 코디네이터가 모든 참여 노드에 "커밋 가능?" 요청을 보내고(Prepare 단계), 전체가 YES면 실제 커밋, 하나라도 NO면 전체 롤백하는 방식입니다. 강한 일관성을 보장하지만, 커밋 전까지 모든 노드에 락이 걸리고 코디네이터 장애 시 전체 시스템이 블로킹되는 치명적 약점이 있습니다. 단일 DB 대비 ~10배 성능 저하도 피할 수 없어 MSA 환경에는 적합하지 않습니다. TC/C는 모든 서비스에 자원을 먼저 예약만 하고(Try), 전체 성공 시 실제 실행(Confirm), 실패 시 예약을 역순으로 취소(Cancel)하는 방식입니다. 2PC와 달리 각 서비스가 별도 트랜잭션으로 처리해 락 범위가 좁고, REST 기반 단순 분산 트랜잭션에 적합하지만 상태 관리 테이블과 피봇 재시도 로직을 별도로 구현해야 합니다. MSA에서 사실상 표준이 된 Saga는 이벤트 기반 비동기 통신으로 서비스 간 느슨한 결합을 유지하고, 각 서비스가 독립적으로 실패를 처리해 전체 시스템 장애 전파를 막습니다. Outbox Pattern과 결합하면 이벤트 발행 신뢰성까지 확보할 수 있습니다. 단, 실패 시 역순으로 실행해야 하는 보상 트랜잭션을 최대 2n개 준비해야 하고, 최종 일관성이므로 중간 상태가 외부에 노출될 수 있다는 점을 설계 시 감안해야 합니다.

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
Transactional Outbox Pattern은 비즈니스 로직과 메시지 발행 사이의 정합성 문제를 해결하는 패턴입니다. Kafka에 직접 발행하면 DB 커밋과 Kafka 발행 사이에 장애가 생겼을 때 둘 중 하나만 성공하는 불일치가 발생합니다. Outbox 패턴은 주문 저장과 Outbox 테이블 insert를 하나의 DB 트랜잭션으로 묶어 원자성을 보장하고, 별도 프로세스가 Outbox 테이블을 읽어 Kafka로 전달합니다. Polling Publisher 방식은 스케줄러가 주기적으로 미발행 이벤트를 SELECT해 Kafka에 발행하고 완료 후 삭제하는 방식으로 구현이 단순하지만 폴링 주기만큼 지연이 생기고 DB에 SELECT 부하가 발생합니다. CDC 방식은 Debezium이 MySQL의 Binlog나 PostgreSQL의 WAL을 실시간으로 읽어 Outbox 테이블 insert를 감지하는 즉시 Kafka로 스트리밍하기 때문에 준실시간 처리가 가능하고 장애 복구 시에도 Kafka offset으로 재처리 위치를 보장합니다. 두 방식 모두 At-least-once로 중복이 발생할 수 있어, 수신 측에서 처리한 이벤트 키를 `processed_events` 테이블에 저장하고 중복 수신 시 스킵하는 멱등성 처리가 필수입니다. Outbox 패턴은 "내가 발행했다"는 사실은 보장하지만 "수신 서비스가 처리했다"는 확인은 두 장군 문제로 인해 이론적으로 보장할 수 없고, 결국 At-least-once + 수신 측 멱등성의 조합이 실무적 최선입니다.

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
메시지 전달 보장 수준은 At-most-once, At-least-once, Exactly-once 세 가지로 나뉩니다. At-most-once는 재시도 없이 최대 1회만 전달해 유실을 허용하는 방식으로, 로그 수집처럼 일부 유실이 허용되는 분석 이벤트에 적합합니다. At-least-once는 ACK를 받을 때까지 재시도해 최소 1회 전달을 보장하지만 중복이 발생할 수 있어 수신 측에서 멱등성 처리가 필요합니다. Exactly-once는 정확히 1회 전달을 의미하지만, 두 장군 문제처럼 신뢰할 수 없는 네트워크에서 두 시스템이 완전한 합의를 이루는 것은 이론적으로 불가능합니다. 메시지 전달 확인의 확인을 무한히 요구하는 구조이기 때문입니다. 실무에서는 완전한 Exactly-once 대신 Effectively-once를 목표로 합니다. At-least-once 기반으로 재처리 전략, 타임아웃, 멱등성, 중복 제거, 순서 보장 전략을 조합해 Exactly-once에 근접한 보장을 달성하는 방식입니다. 트레이드오프는 보장 수준이 높아질수록 구현 복잡도와 성능 비용이 증가한다는 점이며, 서비스의 비즈니스 요구사항에 맞는 수준을 선택하는 것이 중요합니다.

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
Graceful Shutdown은 프로세스가 종료 신호를 받았을 때 즉시 강제 종료하지 않고, 진행 중인 요청과 작업을 모두 완료한 뒤 안전하게 종료하는 방식입니다. 배포나 스케일 다운 시 처리 중인 요청을 강제로 끊으면 데이터 불일치, 커넥션 오류, 메시지 유실이 발생하기 때문에 운영 환경에서는 반드시 필요합니다.

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
Dockerfile에서 `CMD`/`ENTRYPOINT`는 반드시 Exec form(`["app"]`)을 사용해야 합니다. Shell form을 쓰면 PID 1이 sh가 되어 SIGTERM이 앱에 직접 전달되지 않습니다. `preStop` 훅은 SIGTERM보다 먼저 실행되므로 `sleep 5` 등으로 로드밸런서가 해당 Pod를 트래픽에서 제외할 시간을 확보할 수 있습니다. `terminationGracePeriodSeconds`는 실제 처리 최대 시간에 여유를 더해 설정해야 하며, 이 시간을 초과하면 SIGKILL로 강제 종료됩니다. Go에서는 `signal.NotifyContext` 또는 `signal.Notify(ch, syscall.SIGTERM)`으로 신호를 감지해 goroutine에서 shutdown을 트리거하고, Spring Boot에서는 `server.shutdown=graceful`과 `spring.lifecycle.timeout-per-shutdown-phase=30s`를 설정해 처리 중인 요청을 완료한 뒤 종료합니다. 공통적으로 HTTP 서버는 신규 연결을 거부한 뒤 기존 연결 처리가 모두 완료된 후 종료해야 합니다.

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
MSA에서 A → B → C로 이어지는 서비스 호출 체인에서 C가 느려지면 B의 스레드가 모두 응답 대기 상태가 되고, A도 연쇄적으로 지연되는 Cascading Failure가 발생합니다. Circuit Breaker는 특정 서비스가 반복적으로 실패할 때 빠르게 차단(Fail Fast)해 이 장애 전파를 막는 패턴입니다. 정상 상태인 Closed에서는 모든 요청을 통과시키며 실패율을 측정하다가, 실패율이 임계치를 초과하면 Open으로 전환해 모든 요청을 즉시 실패 응답으로 반환합니다. 설정한 대기 시간이 지나면 Half-Open 상태로 전환해 일부 요청만 통과시키고, 성공하면 Closed로 복귀하고 실패하면 다시 Open으로 돌아갑니다. Open 상태에서 Fallback 전략으로 기본값이나 캐시 데이터를 반환해 서비스 저하를 우아하게 처리하는 것이 중요합니다.

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
Rate Limiting 알고리즘은 크게 Fixed Window, Sliding Window, Token Bucket, Leaky Bucket 네 가지로 나뉩니다. Fixed Window는 시간 창 단위로 요청 수를 카운트하는 가장 단순한 방식이지만, 창 경계에서 최대 2배 버스트가 허용되는 취약점이 있습니다. 예를 들어 1분에 100회 제한이라면 00:59초에 100회, 01:00초에 다시 100회로 2초 안에 200회가 통과될 수 있습니다. Sliding Window는 현재 시점 기준 과거 N초 요청 수를 카운트해 이 버스트 취약점을 해소하지만 구현이 복잡합니다. Token Bucket은 토큰이 주기적으로 채워지고 요청마다 토큰을 소비하는 방식으로 버킷 크기만큼 버스트를 허용하면서도 평균 처리율을 제한할 수 있어 일반적인 API에 가장 많이 씁니다. Leaky Bucket은 큐에 요청을 쌓고 일정 속도로만 처리해 버스트를 완전히 차단하고 균일한 처리율을 보장합니다. 분산 환경에서는 Redis를 활용하는데, Token Bucket이라면 토큰 수와 마지막 충전 시각을 Redis에 저장하고 Lua Script로 원자적으로 처리하고, Sliding Window라면 Sorted Set으로 요청 타임스탬프를 관리하며 오래된 항목을 주기적으로 제거합니다.

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
MSA에서 Resilience를 높이기 위한 핵심 패턴은 Timeout, Retry, Circuit Breaker, Bulkhead, Rate Limiter, Fallback 여섯 가지입니다. Timeout은 응답 무한 대기를 방지해 스레드가 묶이는 것을 막는 가장 기본적인 보호 장치입니다. Retry는 일시적 장애에 재시도하되, Exponential Backoff와 Jitter를 함께 써야 모든 인스턴스가 동시에 재시도하는 Thundering Herd 문제를 방지할 수 있습니다. Circuit Breaker는 반복 실패 서비스를 차단해 Cascading Failure를 막고, Bulkhead는 서비스별로 스레드 풀이나 세마포어를 분리해 한 서비스의 장애가 전체로 번지지 않도록 격리합니다. Rate Limiter는 과부하를 방지하고, Fallback은 장애 시 기본값이나 캐시 데이터를 반환해 서비스 저하를 우아하게 처리합니다. 이 패턴들을 체이닝할 때 순서가 중요한데, Retry를 Circuit Breaker 바깥에 두면 Open 상태에서도 재시도해 비효율적이고, Circuit Breaker를 Retry 바깥에 두면 재시도가 실패율 카운트에 포함되어 너무 빨리 Open이 됩니다. 권장 순서는 Rate Limiter → Circuit Breaker → Retry → 외부 서비스입니다.

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

## 분산 시스템에서 합의(Consensus)가 필요한 이유는 무엇이고, Raft는 어떻게 동작하나요?

**난이도**: 기초

**핵심 키워드**: 네트워크 파티션, quorum, 과반수, 리더 선출, etcd, Kubernetes 클러스터 상태

**핵심 개념**:
- 문제: 네트워크 파티션 시 노드마다 다른 값 → "어느 값이 진실?" 결정 불가
- 해결: **과반수(quorum)가 동의한 값만 커밋** → 일관성 보장
- Raft: 리더가 쓰기 수신 → 과반수 팔로워가 복제 완료 → 커밋 확정
- 노드 장애: 과반수 생존 → 리더 재선출 후 정상 동작, 과반수 사망 → 쓰기 거부

**etcd와 Kubernetes 연결**:
- etcd = Raft 기반 분산 KV 저장소 → Kubernetes 클러스터 전체 상태 저장
- 여러 kube-apiserver가 etcd를 읽어도 항상 동일한 상태 보장 (strong consistency)
- etcd 3노드 중 1개 사망 → 과반수(2개) 생존 → 정상 동작
- etcd 3노드 중 2개 사망 → 과반수 손실 → 읽기만 가능, 쓰기 거부

**모범 답변 (3분 이상 말하기 형태)**:
> 분산 시스템에서 여러 노드가 동일한 상태를 유지해야 할 때 가장 큰 문제는 네트워크 파티션입니다. 노드 간 통신이 잠깐 끊기면 각 노드가 서로 다른 값을 갖게 됩니다. 어느 노드의 값이 정답인지 결정하는 기준이 없으면 각자 자기 값이 맞다고 판단해 시스템 일관성이 깨집니다. 최신 쓰기 시간으로 결정하는 방식도 한계가 있는데, 노드마다 시계가 조금씩 달라서 신뢰할 수 없습니다.
>
> Raft 같은 합의 알고리즘은 "과반수(quorum)가 동의한 값만 커밋"하는 방식으로 이 문제를 해결합니다. 한 노드가 리더가 되어 모든 쓰기를 받고, 과반수 이상의 팔로워가 복제 완료했을 때만 커밋으로 확정합니다. 5개 노드에서 3개가 동의해야 커밋되므로, 2개 노드가 죽어도 남은 3개로 계속 동작합니다. 반대로 3개가 죽어서 과반수를 잃으면 데이터 일관성을 보장할 수 없으므로 쓰기를 거부합니다. 가용성보다 일관성을 우선하는 CP 시스템의 특성입니다.
>
> etcd가 Kubernetes에서 사용되는 이유가 바로 여기에 있습니다. etcd는 Raft 기반의 분산 키-값 저장소로, Kubernetes 클러스터의 모든 상태, Pod 목록, Deployment 설정, Service 정의, ConfigMap 등을 저장합니다. etcd가 strong consistency를 보장하기 때문에 여러 kube-apiserver가 동시에 etcd를 읽어도 항상 동일한 클러스터 상태를 봅니다. etcd 노드 중 일부가 죽어도 과반수가 살아있다면 리더를 새로 선출해서 계속 동작하고, 과반수를 잃으면 읽기만 허용하고 쓰기는 거부합니다.

**꼬리 질문 예시**:
- etcd가 3노드인데 2개가 죽으면 Kubernetes는 어떻게 되나요? (기존 상태 조회만 가능, 새 배포·스케일링 불가)
- Raft 리더가 죽으면 어떤 과정으로 새 리더가 선출되나요? (Election Timeout 만료 → Candidate → 과반수 투표 획득 → Leader)

**면접 세션 피드백 (2026-04-13 2회차)**:
- 답변 없음 (주제 자체 모름) — 다음 세션에서 우선 출제 필요
- 핵심 암기 포인트: 과반수 quorum, 리더 선출, etcd = K8s 상태 저장소, 과반수 손실 시 쓰기 거부

---
