---
tags: [distributed-systems, cap-theorem, consistency, interview-questions]
related: [kafka, redis, zookeeper, kubernetes, mysql, postgresql, system-design]
---

# Distributed Systems — 면접 질문

→ [[home]] | 개념 정리: [[topics/distributed-systems/concepts]]

---

## WebSocket vs HTTP

**난이도**: 기초

**핵심 키워드**: full-duplex, HTTP Upgrade, 방화벽 투과성, STOMP, pub/sub, destination 라우팅, gorilla/websocket, Hub 패턴

**모범 답변 방향**:
HTTP와 WebSocket의 가장 큰 차이는 통신 방향과 연결 유지 여부입니다. HTTP는 클라이언트가 요청하면 서버가 응답하고 연결을 끊는 단방향 반이중(Half-Duplex) 구조로, 서버가 먼저 데이터를 보낼 수 없습니다. 반면 WebSocket은 한 번 연결되면 세션을 유지하면서 서버와 클라이언트가 동시에 메시지를 주고받을 수 있는 **full-duplex(전이중)** 통신을 지원합니다. 서버가 클라이언트의 요청 없이도 먼저 데이터를 push할 수 있기 때문에 실시간 채팅, 알림, 주식 시세처럼 서버 이벤트를 즉시 전달해야 하는 시나리오에 본질적으로 적합합니다. 연결 방식은 HTTP 1.1 Upgrade 헤더를 통해 `ws://` 또는 `wss://` 프로토콜로 전환하며, 기존 HTTP 포트(80/443)를 그대로 사용해 방화벽 설정 변경 없이 적용 가능합니다. 이 Upgrade 핸드셰이크가 완료되면 이후 통신은 HTTP가 아닌 WebSocket 프레임 단위로 이루어집니다.

STOMP는 WebSocket 위에 pub/sub 메시징 패턴을 추가하는 상위 프로토콜입니다. WebSocket 자체는 raw 바이트 스트림만 전달하기 때문에 "누가 어떤 채널을 구독하는지" 개념이 없습니다. STOMP는 `/topic/room1` 같은 destination 기반 라우팅, SUBSCRIBE/SEND 프레임 구조, Spring 메시지 브로커 연동을 제공해서 멀티 채팅방 라우팅을 간결하게 구현할 수 있게 합니다. 단, STOMP를 사용하지 않아도 서버 측에서 room_id 기반 Map으로 클라이언트를 그룹핑하는 방식으로 직접 라우팅 로직을 구현할 수 있습니다.

카테노이드에서 실시간 채팅 서버를 구축할 때는 STOMP 없이 gorilla/websocket으로 직접 구현했습니다. Hub 구조로 연결된 모든 클라이언트를 중앙에서 관리하고 broadcast하는 방식을 채택했고, 이를 통해 기존 Polling 방식 대비 채팅 레이턴시를 126배 개선했습니다. Polling은 클라이언트가 일정 주기로 새 메시지를 계속 요청해 불필요한 HTTP 연결 생성/해제 오버헤드가 누적되는 반면, WebSocket은 한 번 연결 후 메시지가 생겼을 때만 즉시 전달하므로 RTT와 서버 부하 모두 크게 줄어들었습니다.

**꼬리 질문 예시**:
- "WebSocket을 Long Polling 대신 선택하는 이유는?" → Long Polling은 서버가 응답할 때마다 연결을 맺고 끊어 오버헤드 큼. WebSocket은 한 번 연결 후 지속 유지 → 레이턴시 낮음
- "STOMP를 사용하지 않으면 멀티 채팅방은 어떻게 구현하나요?" → 서버에서 room_id 기반 Map으로 클라이언트 그룹핑하거나 Hub 패턴으로 직접 라우팅
- "wss://와 ws://의 차이는?" → wss는 TLS 암호화 적용, 프로덕션에서는 wss 필수

**면접 세션 피드백 (2026-04-16 4회차)**:
- 잘한 점: HTTP 1.1 Upgrade 메커니즘 정확. gorilla/websocket 실무 경험. 방화벽 투과성 장점 먼저 언급.
- 보완: full-duplex 키워드 미언급 (서버 push 능력이 선택 핵심 이유). STOMP 기능 전혀 모름 → pub/sub + destination 라우팅 구조 암기 필요. 카테노이드 126배 레이턴시 개선 수치 연결 안 됨.

---

## 멱등성 (Idempotency)

### Q. REST API 설계에서 멱등성이란 무엇이고, HTTP 메서드별로 어떻게 다른가요? 왜 중요한가요?

**난이도**: 기초

**핵심 키워드**: 멱등성, HTTP 스펙, GET/PUT/DELETE(멱등), POST/PATCH(비멱등), Idempotency-Key, Redis TTL, 네트워크 재시도

**HTTP 메서드별 멱등 여부 (스펙 기준)**:

| 메서드 | 멱등 여부 | 이유 |
|---|---|---|
| GET | O | 서버 상태 변경 없음 |
| PUT | O | 전체 교체, N번 호출해도 마지막 상태로 수렴 |
| DELETE | O | 삭제 후 재요청 시 404지만 "없는 상태"는 동일 |
| POST | X | 동일 요청 N번 → N개 리소스 생성 |
| PATCH | X | 상대값 업데이트(`delta: +1`)면 호출마다 결과 다름 |

**멱등키(Idempotency Key) 패턴**:
```
클라이언트: Idempotency-Key: {uuid} 헤더 포함
서버: Redis에 {key: result} TTL과 함께 저장
재요청: 동일 키 → 저장된 결과 반환, 실제 로직 미실행
```
→ Stripe API가 이 패턴을 정확히 사용

**왜 중요한가**:
- 네트워크 타임아웃 → 클라이언트는 실패로 판단 → 재시도
- 실제로는 첫 요청이 느리게 처리 중 → 두 요청 모두 처리 → 중복 결제/데이터
- 분산 시스템에서 중복 요청은 언제든 발생 가능, 100% 방지 불가

**모범 답변 방향**:
멱등성이란 동일한 요청을 여러 번 반복해도 서버의 상태와 결과가 항상 동일한 것을 의미합니다. HTTP 스펙상 GET, PUT, DELETE는 멱등하고, POST와 PATCH는 비멱등입니다. GET은 서버 상태를 변경하지 않으니 당연히 멱등하고, PUT은 리소스 전체를 교체하는 방식이라 N번 호출해도 마지막 상태로 수렴합니다. DELETE는 첫 호출에 삭제되고 이후 404가 반환되지만, "리소스가 없는 상태"라는 결과 자체는 동일합니다. POST는 호출마다 새 리소스를 생성하고, PATCH는 `delta: +1` 같은 상대값 업데이트를 허용하므로 호출마다 결과가 달라져 비멱등입니다. 멱등성이 중요한 이유는 분산 시스템에서 네트워크 타임아웃이 발생하면 클라이언트는 요청이 실패했다고 판단해 재시도하지만, 실제로는 서버에서 처음 요청이 느리게 처리 중일 수 있기 때문입니다. 이때 멱등성이 없는 API라면 중복 결제나 중복 데이터 생성이 발생합니다. 이를 해결하는 대표적인 방법이 멱등키 패턴입니다. 클라이언트가 요청 시 `Idempotency-Key: uuid` 헤더를 함께 보내면, 서버는 Redis에 해당 키와 처리 결과를 TTL과 함께 저장합니다. 같은 키로 재요청이 들어오면 실제 로직을 실행하지 않고 캐시된 결과를 바로 반환합니다. Stripe API가 이 패턴을 사용하며, 결제처럼 중복이 절대 허용되지 않는 도메인에서 필수적입니다. TTL은 일반적으로 재시도가 유효한 기간인 24시간에서 7일 사이로 설정합니다. POST를 멱등하게 만들려면 클라이언트가 요청마다 고유 key를 생성해 헤더로 전달하고 서버에서 중복 체크를 수행하면 됩니다. 트레이드오프로는 멱등키를 Redis에 저장하는 방식이 부가적인 I/O를 유발하고 Redis 장애 시 일관성이 깨질 수 있으므로, 별도 DB 테이블에 저장하거나 Redis와 DB를 병행 운용하는 방식을 검토해야 합니다.

**꼬리 질문 예시**:
- "PATCH가 비멱등인 케이스를 예시로 보여주세요" → `{ "delta": +1 }` 상대값 업데이트는 호출마다 결과 다름
- "멱등키를 Redis에 저장할 때 TTL은 어떻게 설정하나요?" → 재시도 유효 시간(보통 24시간~7일) 기준
- "POST를 멱등하게 만들려면 어떻게 하나요?" → 클라이언트가 unique key 생성 후 멱등키 헤더로 전달, 서버에서 중복 체크

**면접 세션 피드백 (2026-04-20 2회차)**:
- DELETE 멱등성 설명 정확("404여도 상태 동일"). 재시도 시나리오 "출입문" 비유 명확.
- 보완: POST 비멱등인 이유를 "스펙 기준"으로 설명 안 함. PATCH 비멱등 미명시. 멱등키 Redis 저장 패턴 구체성 부족. 이력서 경험 연결 없음.

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

**모범 답변 방향**:
2PC는 분산 환경에서 여러 참여자가 하나의 트랜잭션을 원자적으로 커밋하거나 롤백하도록 보장하는 프로토콜입니다. Coordinator와 Participant라는 두 역할로 구성되며 두 단계로 동작합니다. Phase 1인 Prepare 단계에서 Coordinator는 모든 Participant에게 "커밋 가능하냐?"는 질문을 보내고, 각 Participant는 트랜잭션을 준비한 뒤 락을 잡고 대기하며 OK 또는 NO로 응답합니다. Phase 2인 Commit 단계에서 모든 Participant가 OK를 보내면 Coordinator가 커밋 명령을 내리고, 하나라도 NO이면 전체 롤백을 지시합니다. 강한 일관성을 보장한다는 것이 2PC의 핵심 장점이지만, 치명적인 Blocking 문제가 있습니다. 모든 Participant가 OK를 보내고 락을 잡고 대기하는 상황에서 Coordinator가 장애로 죽으면, Participant들은 커밋해야 할지 롤백해야 할지 결정을 못한 채 락을 잡고 무한 대기 상태에 빠집니다. 이 동안 같은 데이터에 접근하려는 다른 트랜잭션은 전부 블로킹되어 처리량이 급락합니다. 3PC는 Coordinator 없이 Participant끼리 결정할 수 있는 단계를 하나 더 추가해 Blocking 문제를 부분적으로 해소하지만, 네트워크 분리 상황에서는 여전히 불일치가 발생할 수 있습니다. MSA 환경에서는 2PC보다 최종 일관성을 허용하는 Saga 패턴을 사용하는 것이 일반적입니다. 2PC는 전체 트랜잭션 기간 동안 락이 유지되어 성능과 가용성 모두 희생하는 반면, Saga는 락 없이 서비스별 로컬 트랜잭션과 보상 트랜잭션 조합으로 최종 일관성을 달성하기 때문입니다. 2PC가 적합한 상황은 XA를 지원하는 DB가 2개 이하이고 트랜잭션이 짧으며 강한 일관성이 절대적으로 필요한 경우로 한정됩니다. 단일 DB 대비 ~10배 성능 저하가 불가피하기 때문에 MSA 설계에서는 가급적 비즈니스 경계를 서비스별 단일 DB 트랜잭션 내에서 처리할 수 있도록 설계하는 것이 우선입니다.

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
CAP 정리는 분산 시스템에서 Consistency, Availability, Partition Tolerance 세 가지 속성을 동시에 모두 보장할 수 없다는 이론입니다. 각 속성의 의미를 정확히 구분하는 것이 중요합니다. Consistency는 모든 노드가 동시에 동일한 데이터를 보는 것으로, 쓰기 이후의 읽기는 반드시 최신값을 반환해야 합니다. Availability는 어떤 요청에도 오류 없이 응답을 돌려준다는 것이고, 다만 그 값이 최신값이 아닐 수 있습니다. Partition Tolerance는 네트워크 단절이 발생하더라도 시스템이 계속 동작하는 능력입니다. 세 가지 중 두 개만 선택할 수밖에 없는 이유는 P의 특수성 때문입니다. 실제 분산 환경에서 네트워크 파티션은 언제든 발생할 수 있고, 이를 아예 포기한 시스템은 분산 시스템이라 부르기 어렵습니다. 따라서 P는 사실상 필수이고, 결국 선택지는 CP와 AP 두 가지가 됩니다. CP 시스템인 ZooKeeper나 HBase는 파티션이 발생하면 일부 요청에 응답을 거부하더라도 데이터 일관성을 지킵니다. 반면 AP 시스템인 Cassandra나 DynamoDB는 파티션 상황에서도 계속 응답하지만, 노드 간 동기화가 완료되기 전의 stale 데이터를 반환할 수 있습니다. 이 트레이드오프는 운영 중에도 실제로 체감됩니다. 샵라이브에서 실시간 라이브 스트리밍 서비스를 운영할 때 ZooKeeper를 사용했는데, 이것이 CP 시스템임을 이해하고 있었기 때문에 리더 선출 중 쓰기가 잠시 거부되는 상황을 미리 예상하고 재시도 로직을 설계할 수 있었습니다. CP와 AP 중 어느 쪽을 선택하느냐는 서비스의 특성에 달려 있습니다. 금융이나 예약처럼 데이터 정합성이 비즈니스 핵심인 경우에는 CP를 선택해 일관성을 우선시하고, 좋아요 수나 조회수처럼 약간의 불일치가 허용되며 높은 가용성이 중요한 경우에는 AP를 선택하는 것이 일반적인 판단 기준입니다. 참고로 CAP 정리는 2000년 Eric Brewer가 제안한 이론으로, 이후 PACELC 정리가 파티션이 없는 정상 상황에서도 Latency와 Consistency 간의 트레이드오프가 존재함을 보완했습니다.

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
MSA에서 DB가 서비스별로 분리되면 하나의 비즈니스 로직이 여러 로컬 트랜잭션으로 분리됩니다. 결제 서버에서 네트워크 오류가 발생하면 재고는 이미 차감됐는데 결제는 실패하는 데이터 정합성 붕괴가 생기고, 이를 막기 위해 2PC를 쓰면 전체 서비스에 락이 걸려 가용성을 희생해야 합니다. Saga 패턴은 이 문제를 n개 서비스의 로컬 트랜잭션을 순서대로 실행하고, 실패 시 실패 지점부터 역순으로 보상 트랜잭션(Compensating Transaction)을 실행해 최종 일관성을 확보하는 방식으로 해결합니다. n개 작업이면 보상 포함 최대 2n개 트랜잭션을 설계해야 합니다. 이벤트 기반 통신과 Outbox Pattern을 함께 사용해 비즈니스 로직 커밋과 이벤트 발행을 하나의 DB 트랜잭션으로 묶어 원자적으로 처리하는 것이 권장됩니다. 구현 방식은 Choreography(안무형)와 Orchestration(지휘형) 두 가지로 나뉩니다. Choreography는 각 서비스가 이벤트를 발행하고 다른 서비스가 구독해 자율적으로 반응하는 방식으로 결합도가 낮고 단순한 Saga에 적합하지만, 전체 흐름 추적이 어렵고 A→B→C→A 같은 순환 의존성 위험이 있으며 조용한 실패를 감지하기 어렵습니다. Orchestration은 중앙 Saga Orchestrator가 각 서비스를 순서대로 호출하고 실패 처리를 담당해 전체 흐름을 한 곳에서 파악하기 쉽고 복잡한 비즈니스 흐름에 적합하며 대부분의 상황에서 권장됩니다. 단, Orchestrator는 순서화만 담당하고 비즈니스 로직을 갖지 않아야 단일 장애점이 되지 않습니다. 또한 보상 트랜잭션은 네트워크 재시도로 중복 실행될 수 있으므로 반드시 멱등성을 보장해야 하고, Saga Log Table에 각 단계 상태를 기록해 실패 시 어느 단계까지 완료됐는지 확인하고 역순으로 보상할 수 있어야 합니다.

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

**Choreography vs Orchestration 선택 기준 — 워크플로우 복잡도:**
- **단순한 흐름 (서비스 2~3개, 선형)** → Choreography: 각 서비스가 이벤트를 발행하고 다음 서비스가 구독하는 느슨한 결합. 중앙 조정자 없이 자율적.
- **복잡한 조건 분기 (서비스 5개 이상, 조건부 실행)** → Orchestration: 중앙 Orchestrator가 흐름을 제어. 전체 상태 추적·디버깅·조건 분기 처리가 명확.
- 판단 기준: "흐름을 그림으로 그렸을 때 한눈에 보이면 Choreography, 흐름도가 필요하면 Orchestration"

**일관성 트레이드오프 요약:**
- **2PC** = 강한 일관성(Strong Consistency). 전체 트랜잭션 동안 락 유지 → 정합성 보장, 성능·가용성 희생
- **Saga** = 최종 일관성(Eventual Consistency) + 보상 트랜잭션(Compensating Transaction). 락 없이 서비스별 로컬 트랜잭션 → 성능·가용성 유지, 중간 상태 노출 가능

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
Graceful Shutdown은 프로세스가 종료 신호를 받았을 때 즉시 강제 종료하지 않고, 진행 중인 요청과 작업을 모두 완료한 뒤 안전하게 종료하는 방식입니다. 배포나 스케일 다운 시 처리 중인 요청을 강제로 끊으면 데이터 불일치, 커넥션 오류, 메시지 유실이 발생하기 때문에 운영 환경에서는 반드시 필요합니다. Kubernetes에서 Pod를 종료할 때 kubelet은 먼저 SIGTERM 신호를 컨테이너에 전송하고, 앱은 이를 감지해 신규 요청 수신을 중단한 뒤 진행 중인 요청을 완료하는 Connection Draining 단계를 거칩니다. `terminationGracePeriodSeconds`(기본 30초) 내에 종료가 완료되지 않으면 SIGKILL로 강제 종료됩니다. 구현 측면에서 Dockerfile의 `CMD`/`ENTRYPOINT`는 반드시 Exec form(`["app"]`)을 사용해야 SIGTERM이 앱 프로세스에 직접 전달됩니다. Shell form을 사용하면 PID 1이 sh가 되어 신호가 앱에 도달하지 않습니다. `preStop` 훅에서 `sleep 5` 정도의 지연을 두면 로드밸런서가 해당 Pod를 트래픽에서 제외할 충분한 시간을 확보할 수 있습니다. Go에서는 `signal.NotifyContext`로 SIGTERM을 감지해 서버 shutdown을 트리거하고, Spring Boot에서는 `server.shutdown=graceful` 설정과 `spring.lifecycle.timeout-per-shutdown-phase`로 처리 중인 요청이 완료될 때까지 대기하도록 구성합니다. 트레이드오프로는 `terminationGracePeriodSeconds`가 너무 짧으면 요청이 강제 중단되고, 너무 길면 배포 시간이 늘어나므로 실제 최대 응답 시간 기준으로 여유 있게 설정해야 합니다.

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
Blocking/Non-Blocking과 동기/비동기는 구분 기준이 다릅니다. Blocking/Non-Blocking은 호출된 함수가 제어권을 즉시 반환하는가의 문제입니다. Blocking은 호출자가 결과를 받을 때까지 아무것도 할 수 없고, Non-Blocking은 제어권을 즉시 돌려받아 다른 작업을 계속할 수 있습니다. 동기/비동기는 결과를 누가 확인하는가의 문제입니다. 동기는 호출자가 직접 결과를 폴링하거나 기다리고, 비동기는 호출자가 기다리지 않고 완료 시 알림(Callback, Event)을 받습니다. 두 축은 독립적이므로 4가지 조합이 가능합니다. 가장 일반적인 동기 + Blocking은 스레드 하나가 I/O 완료까지 대기하는 구조로 Spring MVC가 이 모델입니다. 요청이 몰리면 스레드 풀이 고갈될 수 있어 대용량 트래픽에서 스케일링 비용이 증가합니다. 실무에서 가장 효율적인 조합은 비동기 + Non-Blocking입니다. 제어권을 즉시 반환받아 다른 작업을 하다가 I/O 완료 이벤트가 왔을 때 처리하는 방식으로, Spring WebFlux(Netty 기반), Node.js Event Loop, Kafka Consumer의 poll 루프가 이 모델입니다. 적은 수의 스레드로 대량 요청을 처리할 수 있어 I/O Bound 워크로드에서 유리합니다. 반면 동기 + Non-Blocking은 제어권을 받았지만 결과를 호출자가 직접 폴링해야 해서 CPU를 낭비하는 바쁜 대기(Busy Wait)가 발생할 수 있어 실용성이 낮습니다. 비동기 + Blocking은 이론적으로는 존재하지만 제어권도 없이 기다리면서 콜백만 기대하는 구조라 실질적으로 의미가 없습니다. 선택 기준은 I/O Bound 작업이 많고 동시 요청이 많다면 비동기 + Non-Blocking, CPU Bound 작업이 많다면 스레드 수를 늘리는 동기 + Blocking이 일반적으로 더 적합합니다.

**4가지 조합**:

| 조합 | 동작 | 실무 예시 |
|---|---|---|
| **동기 + Blocking** | 호출 후 결과 올 때까지 대기, 제어권도 없음 | 일반 JDBC 쿼리, `Thread.sleep()` |
| **동기 + Non-Blocking** | 제어권 즉시 반환받지만 결과를 직접 폴링 | `Future.isDone()` 반복 확인 |
| **비동기 + Blocking** | 제어권 없이 대기, 결과는 콜백으로 처리 | 잘못 구현된 비동기 (실질적으로 의미 없음) |
| **비동기 + Non-Blocking** | 제어권 즉시 반환, 결과는 콜백/이벤트로 알림 | Netty, WebFlux, Node.js, Kafka Consumer |

**실무 핵심**:
실무에서 Spring MVC는 동기 + Blocking 모델로 스레드 하나가 요청 하나를 전담합니다. 요청이 몰리면 스레드 고갈 위험이 있습니다. Spring WebFlux는 비동기 + Non-Blocking으로 Netty 기반에서 적은 스레드로 대량 요청을 처리합니다. Node.js도 비동기 + Non-Blocking이지만 싱글 스레드 + Event Loop 방식입니다.

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
MSA에서 A → B → C로 이어지는 서비스 호출 체인에서 C가 느려지면 B의 스레드가 모두 응답 대기 상태가 되고, A도 연쇄적으로 지연되는 Cascading Failure가 발생합니다. Circuit Breaker는 특정 서비스가 반복적으로 실패할 때 빠르게 차단(Fail Fast)해 이 장애 전파를 막는 패턴입니다. 전기 회로 차단기처럼 과부하 상태를 감지하면 회로를 열어 전류(요청)를 차단하고, 안정화된 후 다시 연결하는 방식을 소프트웨어에 적용한 것입니다. 상태는 Closed, Open, Half-Open 세 가지로 구성됩니다. 정상 상태인 Closed에서는 모든 요청을 통과시키며 실패율을 측정하다가, 실패율이 임계치(예: 50%)를 초과하면 Open으로 전환해 모든 요청을 즉시 실패 응답으로 반환합니다. 이때 외부 서비스에 추가 부하를 주지 않으므로 장애가 전파되지 않습니다. 설정한 대기 시간이 지나면 Half-Open 상태로 전환해 제한된 수의 요청만 통과시키고, 성공하면 Closed로 복귀하고 실패하면 다시 Open으로 돌아갑니다. Open 상태에서는 Fallback 전략으로 기본값이나 캐시 데이터를 반환해 서비스 저하를 우아하게 처리하는 것이 중요합니다. 샵라이브의 MultiCDN 운영 경험에서 CDN 장애 시 이 패턴을 적용했는데, CDN과 사용자 간 네트워크 품질이 개인마다 달라 자동 전환이 오히려 전체 시청자에게 재접속 충격을 줄 수 있었기 때문에 CloudWatch에서 에러율과 응답 지연을 모니터링하다가 수동으로 fallback CDN으로 전환하는 방식을 채택했습니다. Java에서는 resilience4j가 표준 라이브러리로 Circuit Breaker, Retry, Bulkhead를 함께 체이닝할 수 있습니다.

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

**MultiCDN 장애 대응 — Circuit Breaker 실무 적용 패턴** (2026-04-17 세션):
- CDN 장애 시 Circuit Breaker 흐름: Closed(정상) → Open(fallback CDN으로 전환) → Half-Open(원래 CDN 복구 확인) → Closed
- **수동 전환 이유**: CDN/네트워크 품질은 사용자마다 달라 자동 전환이 전체 시청자에게 재접속 충격을 줄 수 있음 → 라이브커머스에서 이 자체가 장애
- **장애 감지**: CloudWatch에서 CDN별 4xx/5xx 에러율·응답 지연 수집 → Grafana 대시보드로 전체 추이 시각화 → 특정 시청자 문제 vs 전체 추이 상승 구분
- **전환 구현**: 플레이어가 방송 진입 시 primary + fallback CDN URL 모두 보유 → 관리자 콘솔에서 전환 버튼 클릭 → 서버가 WebSocket으로 해당 방송 전체 시청자에게 reload 커맨드 broadcast → 플레이어 fallback CDN으로 재접속

**면접 세션 피드백 (2026-04-17 1회차)**:
- 잘한 점: 상태 전환 흐름과 수동 판단 이유를 비즈니스 맥락으로 정확히 설명. CloudWatch+Grafana 모니터링 흐름 선제 언급. WebSocket broadcast reload 구현 방식 정확.
- 보완: **표준 상태명** — "off/half-on/on" 대신 **Closed/Open/Half-Open** 사용 필수. 결과 마무리 문장 없음.

> 출처: https://hudi.blog/circuit-breaker-pattern/
> 출처: https://seongwon.dev/MSA/20230426-서킷브레이커란/

---

## Rate Limiting 알고리즘을 비교하고 분산 환경에서 어떻게 구현하나요?

**난이도**: 중급

**핵심 키워드**: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window, Redis, 분산 Rate Limiter

**모범 답변 방향**:
Rate Limiting은 특정 시간 내 요청 수를 제한해 서비스 과부하와 악의적인 트래픽을 방지하는 패턴입니다. 알고리즘은 크게 Fixed Window, Sliding Window, Token Bucket, Leaky Bucket 네 가지로 나뉩니다. Fixed Window는 시간 창 단위로 요청 수를 카운트하는 가장 단순한 방식이지만, 창 경계에서 최대 2배 버스트가 허용되는 취약점이 있습니다. 예를 들어 1분에 100회 제한이라면 00:59초에 100회, 01:00초에 다시 100회로 2초 안에 200회가 통과될 수 있습니다. Sliding Window는 현재 시점 기준 과거 N초 요청 수를 카운트해 이 버스트 취약점을 해소하지만 모든 요청의 타임스탬프를 관리해야 하므로 메모리와 구현 복잡도가 높아집니다. Token Bucket은 버킷에 토큰이 일정 속도로 채워지고 요청마다 토큰을 소비하는 방식으로, 버킷 크기만큼 단기 버스트를 허용하면서도 평균 처리율을 제한할 수 있어 일반 API Rate Limiting에 가장 널리 씁니다. Leaky Bucket은 큐에 요청을 쌓고 일정 속도로만 처리해 버스트를 완전히 차단하고 균일한 처리율을 보장하며 스트리밍이나 큐 기반 처리에 적합합니다. 분산 환경에서는 인스턴스별로 카운터를 관리하면 총량이 초과될 수 있어 Redis를 중앙 저장소로 활용합니다. Token Bucket이라면 토큰 수와 마지막 충전 시각을 Redis에 저장하고 Lua Script로 원자적으로 처리하고, Sliding Window라면 Redis Sorted Set으로 요청 타임스탬프를 관리하며 현재 시점 기준 윈도우 밖의 항목을 주기적으로 제거합니다. Lua Script를 사용하는 이유는 Redis 명령 여러 개를 원자적으로 실행해 동시 요청에서 카운터 값이 어긋나는 Race Condition을 방지하기 위함입니다. 실무에서는 AWS API Gateway, Nginx `limit_req`, Spring Cloud Gateway, Java의 Bucket4j 라이브러리를 통해 손쉽게 적용할 수 있습니다.

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
MSA에서 Resilience(복원력)를 높이기 위한 핵심 패턴은 Timeout, Retry, Circuit Breaker, Bulkhead, Rate Limiter, Fallback 여섯 가지입니다. 각 패턴은 특정 유형의 장애를 막는 역할이 명확히 분리되어 있습니다. Timeout은 응답 무한 대기를 방지해 스레드가 묶이는 것을 막는 가장 기본적인 보호 장치로, SLA 기반으로 적절한 타임아웃을 설정하지 않으면 모든 스레드가 I/O 대기에 빠져 서비스 전체가 멈출 수 있습니다. Retry는 일시적 장애에 재시도하되, Exponential Backoff와 Jitter를 함께 써야 모든 인스턴스가 동시에 재시도해 장애 서비스에 순간적인 폭증 트래픽을 주는 Thundering Herd 문제를 방지할 수 있습니다. Circuit Breaker는 반복 실패 서비스를 감지해 빠르게 차단함으로써 Cascading Failure를 막고, Bulkhead는 서비스별로 스레드 풀이나 세마포어를 분리해 한 서비스의 장애가 전체 서비스 스레드 풀을 소진시키지 않도록 격리합니다. Rate Limiter는 과도한 요청으로부터 서비스를 보호하고, Fallback은 장애 시 기본값이나 캐시 데이터를 반환해 서비스 저하를 우아하게 처리합니다. 이 패턴들을 체이닝할 때는 순서가 중요합니다. Retry를 Circuit Breaker 안쪽에 두면 재시도가 실패율 카운트에 포함되어 Circuit Breaker가 너무 빨리 Open 상태로 전환되고, Retry를 Circuit Breaker 바깥에 두면 Open 상태에서도 재시도해 비효율적입니다. 권장 순서는 Rate Limiter → Circuit Breaker → Retry → 외부 서비스이며, Java에서는 resilience4j로 이 체이닝을 선언적으로 구성할 수 있습니다. 트레이드오프로는 패턴을 많이 쌓을수록 각 설정값(타임아웃, 실패율 임계치, 재시도 횟수 등)이 복잡하게 상호작용하므로, 실제 부하 테스트를 통해 설정값을 검증해야 합니다.

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
> Raft 같은 합의 알고리즘은 "과반수(quorum)가 동의한 값만 커밋"하는 방식으로 이 문제를 해결합니다. 한 노드가 리더가 되어 모든 쓰기를 받고, 과반수 이상의 팔로워가 복제 완료했을 때만 커밋으로 확정합니다. 5개 노드에서 3개가 동의해야 커밋되므로, 2개 노드가 죽어도 남은 3개로 계속 동작합니다. 반대로 3개가 죽어서 과반수를 잃으면 데이터 일관성을 보장할 수 없으므로 쓰기를 거부합니다. 가용성보다 일관성을 우선하는 CP 시스템의 특성입니다. 리더가 장애로 다운되면 팔로워들은 Election Timeout이 만료될 때 Candidate 상태로 전환해 투표를 요청하고, 과반수의 투표를 먼저 획득한 노드가 새 리더가 됩니다.
>
> etcd가 Kubernetes에서 사용되는 이유가 바로 여기에 있습니다. etcd는 Raft 기반의 분산 키-값 저장소로, Kubernetes 클러스터의 모든 상태, Pod 목록, Deployment 설정, Service 정의, ConfigMap 등을 저장합니다. etcd가 strong consistency를 보장하기 때문에 여러 kube-apiserver가 동시에 etcd를 읽어도 항상 동일한 클러스터 상태를 봅니다. etcd 노드 중 일부가 죽어도 과반수가 살아있다면 리더를 새로 선출해서 계속 동작하고, 과반수를 잃으면 읽기만 허용하고 쓰기는 거부합니다. 이 때문에 etcd를 3노드로 운영하면 1개 장애를 허용하고, 5노드로 운영하면 2개 장애까지 허용할 수 있습니다. 샵라이브에서 ZooKeeper를 사용할 때도 동일한 원리로 ZooKeeper 앙상블을 홀수로 구성해 과반수 쿼럼을 보장했고, 리더 선출 중 짧은 시간 쓰기가 불가할 수 있다는 점을 인지하고 클라이언트 재시도 로직을 함께 설계했습니다.

**꼬리 질문 예시**:
- etcd가 3노드인데 2개가 죽으면 Kubernetes는 어떻게 되나요? (기존 상태 조회만 가능, 새 배포·스케일링 불가)
- Raft 리더가 죽으면 어떤 과정으로 새 리더가 선출되나요? (Election Timeout 만료 → Candidate → 과반수 투표 획득 → Leader)

**면접 세션 피드백 (2026-04-13 2회차)**:
- 답변 없음 (주제 자체 모름) — 다음 세션에서 우선 출제 필요
- 핵심 암기 포인트: 과반수 quorum, 리더 선출, etcd = K8s 상태 저장소, 과반수 손실 시 쓰기 거부

---

## REST API 버저닝

**Q. REST API 버저닝 전략 3가지(URI, Header, Content-Type)를 비교하고, B2B SaaS에서 하위 호환성을 어떻게 관리하겠나요?**

**난이도**: 기초

**핵심 키워드**: URI 버저닝, Header(X-API-Version), Content-Type(Accept: vnd), Sunset 헤더, Vary 헤더, 하위 호환성, 고객사별 버전 DB 매핑

**3가지 전략 비교**:

| 방식 | 예시 | 장점 | 단점 |
|---|---|---|---|
| **URI** | `/v1/products` | 직관적, 브라우저 테스트 쉬움, Swagger 분리 용이 | URL 중복, REST 원칙(동일 리소스=동일 URL) 위반 |
| **Header** | `X-API-Version: 2` | URL 클린, RESTful | 브라우저 테스트 어려움, CDN Vary 헤더 설정 필요 |
| **Content-Type** | `Accept: application/vnd.myapi.v2+json` | 가장 RESTful | 복잡, 실무에서 거의 미사용 |

**모범 답변 방향**:

API 버저닝은 클라이언트와의 계약을 유지하면서 서버 API를 진화시키기 위한 전략입니다. 대표적인 세 가지 방식은 URI 버저닝, 요청 헤더 버저닝, Content-Type 버저닝입니다. URI 버저닝은 `/v1/products`, `/v2/products` 처럼 경로에 버전을 포함하는 방식으로, 브라우저에서 바로 테스트할 수 있고 Swagger 문서를 버전별로 분리하기 쉬워 실무에서 가장 많이 쓰입니다. 다만 동일 리소스에 여러 URL이 생기므로 엄격한 REST 원칙과는 맞지 않습니다. 헤더 버저닝은 `X-API-Version: 2` 헤더로 버전을 전달하는 방식이며 URL을 깔끔하게 유지할 수 있지만, CDN이 버전 헤더를 포함해 캐시를 구분하도록 Vary 헤더를 함께 설정해야 하고 브라우저에서 직접 테스트하기 어렵습니다. Content-Type 버저닝은 `Accept: application/vnd.myapi.v2+json` 처럼 미디어 타입에 버전을 담는 방식으로 가장 RESTful하지만 복잡도가 높아 실무에서는 거의 사용하지 않습니다.

B2B SaaS 환경에서는 하위 호환성 관리가 특히 중요합니다. B2B 고객사는 사내 시스템을 API와 연동해놓기 때문에, 서버 측에서 API를 변경하면 고객사 시스템 장애로 이어질 수 있습니다. 핵심 원칙은 네 가지입니다. 첫째, 기존 필드를 절대 제거하지 않습니다. 새로운 필드는 Optional로만 추가하며, 기존 클라이언트가 새 필드를 모르더라도 기존 동작에 영향이 없어야 합니다. 둘째, 폐기 예정인 API에는 응답 헤더에 `Sunset: 2026-12-31` 을 포함시켜 클라이언트가 언제까지 마이그레이션해야 하는지 명시합니다. 셋째, 고객사별로 현재 사용 중인 API 버전을 DB에 매핑해 의존성을 추적합니다. 이를 통해 특정 버전을 폐기할 때 어느 고객사에 영향이 가는지 미리 파악할 수 있습니다. 넷째, 버전별 라우팅은 API Gateway에서 처리해 비즈니스 로직과 분리합니다.

카테노이드에서 채팅 서버 API를 운영할 때도 클라이언트 앱이 다양한 버전으로 배포되어 있어, 구버전 클라이언트가 신버전 서버 API와도 동작해야 했습니다. 이때 응답 필드를 제거하지 않고 추가만 하는 원칙이 실제로 중요하게 작용했습니다. 샵라이브 라이브 스트리밍 플랫폼에서도 외부 파트너사가 API를 연동하고 있었기 때문에, 일방적인 스펙 변경 없이 버전을 유지하는 것이 계약 상의 의무이기도 했습니다. 이런 경험에서 URI 버저닝이 팀 내 소통과 문서화 측면에서 가장 실용적이라는 것을 느꼈고, B2B 계약이 있는 API는 Sunset 헤더와 고객사 의존성 추적을 병행해야 안전하게 버전을 관리할 수 있다고 생각합니다.

**B2B 하위 호환성 관리 원칙**:
1. 기존 필드 제거 금지 — 새 필드만 Optional로 추가
2. 폐기 예정 API → 응답 헤더에 `Sunset: 2026-12-31` 추가
3. 고객사별 사용 버전을 DB에 매핑해 의존성 추적
4. API Gateway에서 버전별 라우팅 처리 (비즈니스 로직과 분리)

**면접 세션 피드백 (2026-04-21 3회차)**:
- Header 버저닝을 고객사 인증 키와 혼동 — X-API-Version이 버전 명시용 헤더임을 암기
- B2B 하위 호환성 관리 전혀 미언급 — Sunset 헤더·고객사별 버전 DB 매핑·필드 추가만 허용 원칙 암기 필요

---
