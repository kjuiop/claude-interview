---
tags: [rabbitmq, messaging, amqp, interview-questions]
related: [kafka, distributed-systems, aws]
---

# RabbitMQ — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/rabbitmq/concepts]] | 비교: [[topics/kafka/questions]]

---

## RabbitMQ Exchange 타입과 Kafka 선택 기준

**난이도**: 중급

**핵심 키워드**: Direct/Fanout/Topic/Headers Exchange, routing key, wildcard, Ack, Kafka exactly-once, idempotent producer

**모범 답변 방향**:
RabbitMQ의 Exchange는 Producer가 보낸 메시지를 어떤 Queue로 보낼지 결정하는 라우팅 규칙입니다. 타입은 네 가지가 있는데, Direct는 routing key가 완전히 일치하는 큐에만 전달하고, Fanout은 routing key를 무시하고 바인딩된 모든 큐에 브로드캐스트합니다. Topic은 `*`(단어 하나)와 `#`(0개 이상 단어) 와일드카드 패턴으로 매칭하는 방식이라 멀티테넌트나 서비스별 이벤트 필터링에 유연하게 활용됩니다. Headers는 routing key 대신 메시지 헤더 속성으로 라우팅하는데, 복잡한 필터 조건이 필요할 때 씁니다. Kafka와 비교하면, 메시지 재처리가 필요하거나 exactly-once가 중요한 경우, 대용량 수평확장이 필요한 경우에는 Kafka가 적합합니다. 반면 복잡한 라우팅 로직이나 Task Queue, 작업 완료 보장이 중요한 경우에는 RabbitMQ가 더 자연스럽습니다. 카테노이드에서 트랜스코더 오케스트레이션을 구현할 때 Direct Exchange로 인스턴스별 전용 큐를 바인딩해서 작업을 분배하고, Polling 방식에서 Event-Driven으로 전환해 불필요한 헬스체크 70회/분을 제거한 경험이 있습니다.

**꼬리 질문 예시**:
- "Topic Exchange와 Fanout Exchange의 차이는?" → Fanout은 모든 큐에 무조건 전송, Topic은 패턴 매칭된 큐에만 전송
- "Kafka exactly-once를 구현하려면 어떻게 설정하나요?" → `enable.idempotence=true` (producer) + transaction API + `isolation.level=read_committed` (consumer)
- "RabbitMQ에서 메시지 유실을 방지하는 방법은?" → manual ack + Dead Letter Exchange + 메시지 persistent 설정

**면접 세션 피드백 (2026-03-30 1회차)**:
- 잘한 점: Kafka exactly-once 실무 설정 구체적 언급, RabbitMQ 큐 바인딩 유연성 실무 관점 설명
- 보완: Direct = "1:1"이 아니라 "routing key 완전 일치". Topic = "논리 주제"가 아니라 "와일드카드 패턴 매칭"이 핵심. `independent` → `idempotent producer`, `readcommit` → `isolation.level=read_committed`

**면접 세션 피드백 (2026-04-02 2회차)**:
- 잘한 점: Topic Exchange dot 구분자 패턴을 실제 예시로 설명
- 보완:
  - Direct Exchange 교정: "큐 이름 일치" → "routing key 완전 일치". 큐에 여러 routing key 바인딩 가능 → 1:1 아님
  - `*` 범위 교정: 단어 **정확히 1개**. `order.*`는 `order.create` 매칭, `order.create.success` 불일치
  - `#` 신규 암기: **0개 이상** 단어. `order.#`는 `order`, `order.create`, `order.create.success` 모두 매칭
  - 선택 기준 한 문장: "Direct는 정확한 1:1 라우팅, Topic은 패턴으로 여러 서비스에 유연하게 분배"

**면접 세션 피드백 (2026-04-13 2회차)**:
- 잘한 점: 4가지 Exchange 타입 정확히 구분. Topic 선택 이유(`ad.click`, `ad.#` 패턴 예시) 실용적. Kafka 비교에서 append-only 로그·파티션 수평확장·consumer group 독립 오프셋·재처리 키워드 모두 언급. 트랜스코딩/자막추출/암호화 실무 경험 연결 자연스럽고 설득력 있음.
- 보완: manual ACK/NACK + DLQ 조합 추가 ("트랜스코딩 실패 시 NACK → DLQ → 재처리" 패턴). Kafka 재처리 구체화 ("오프셋 리셋 → 집계 버그 수정 후 재집계").

**면접 세션 피드백 (2026-04-16 3회차)**:
- 잘한 점: Direct/Fanout/Topic 동작과 사용 기준 정확. `*` vs `#` 와일드카드를 payment.*/payment.# 예시로 명확히 구분.
- 보완: Fanout의 "라우팅 키를 완전히 무시한다"는 표현 명시 필요. 실무 경험(카테노이드 트랜스코더) 연결 없음. Direct vs Topic 선택 기준("라우팅 키 고정·단순 → Direct, 계층 구조·확장 가능성 → Topic") 추가 필요.

---

## RabbitMQ Dead Letter Exchange 활용 패턴

**난이도**: 중급

**핵심 키워드**: DLX, Dead Letter Queue, nack, requeue, TTL, 재처리

**모범 답변 방향**:
Dead Letter Exchange는 RabbitMQ에서 처리에 실패하거나 TTL이 만료된 메시지를 유실시키지 않고 별도 큐로 보내는 패턴입니다. Consumer가 메시지를 처리하다 오류가 발생하면 `nack(requeue=false)`를 호출하고, 해당 메시지는 DLX를 거쳐 Dead Letter Queue로 라우팅됩니다. requeue=true를 사용하면 메시지가 원래 큐 앞으로 돌아와 무한 재시도 루프가 발생할 수 있어 실무에서는 DLX 연계가 안전합니다. TTL이 만료된 메시지도 동일하게 DLX로 이동하기 때문에, 한 번의 DLX 설정으로 실패와 만료를 모두 처리할 수 있습니다. DLQ에 쌓인 메시지는 별도 Consumer가 모니터링·알림을 보내거나, 원인 분석 후 원래 큐에 재발행하는 방식으로 처리합니다. 트레이드오프는 DLX 설정이 없으면 메시지가 단순히 버려지거나 무한 루프에 빠지지만, DLX를 도입하면 운영 복잡도가 높아지고 DLQ도 주기적으로 관리해야 한다는 점입니다.

**꼬리 질문 예시**:
- "DLX 없이 nack하면 어떻게 되나요?" → requeue=true면 큐 앞으로 돌아와 무한 재시도 루프 발생 위험
- "Kafka의 DLQ와 비교하면?" → Kafka는 offset 재조정으로 재처리 가능, DLQ 패턴은 별도 topic으로 구현
- "retry 횟수를 어떻게 추적하나요?" → RabbitMQ는 자체 추적 없음. `x-death` 헤더로 카운트하거나 Spring AMQP `RetryTemplate` + `RepublishMessageRecoverer` 사용

**x-dead-letter-exchange 설정 예시** (2026-04-16 세션 보완):
```java
@Bean
Queue workQueue() {
    return QueueBuilder.durable("work.queue")
        .withArgument("x-dead-letter-exchange", "dlx.exchange")
        .withArgument("x-message-ttl", 30000)  // TTL 만료도 DLX로
        .build();
}
```
- NACK + `requeue=false` → 자동으로 `x-dead-letter-exchange`로 라우팅
- Spring AMQP 재시도: `RetryTemplate`(횟수/백오프 설정) + `RepublishMessageRecoverer`(최종 실패 시 에러 큐로 발행)

**면접 세션 피드백 (2026-04-16 1회차)**:
- 잘한 점: NACK + 수동 ACK + 지수 백오프 + DLQ 알람 전체 흐름 올바르게 파악
- 보완: `x-dead-letter-exchange` 큐 선언 방법 모름. retry count 추적 방법(`x-death` 헤더, RetryTemplate) 모름. 이 두 가지는 "설계했다"고 말하면 바로 나오는 꼬리 질문.

**면접 세션 피드백 (2026-04-17 2회차)** ⚠️ 3회 연속 동일 포인트 막힘:
- 잘한 점: DLX 별도 선언 방향 정확. 헤더로 재시도 횟수 추적한다는 개념 올바름. DLQ 임계치 초과 시 전환 흐름 이해.
- 보완 (3회 연속 미해결):
  - **`x-dead-letter-exchange` 속성명** — 큐 선언 arguments에 `"x-dead-letter-exchange": "dlx.name"` 설정이 없으면 NACK 메시지가 DLX로 라우팅되지 않음. 이 속성명을 반드시 암기할 것
  - **NACK + `requeue=false` 조합** — `channel.basicNack(tag, false, false)` 또는 Spring AMQP `defaultRequeueRejected=false` + 예외 throw → 자동 DLX 라우팅
  - **헤더명 교정** — "dead-letter-header" ❌ → `x-death` ✅ (RabbitMQ가 DLX 통과 시 자동 추가, `count` 필드로 횟수 확인)
  - **Spring AMQP 선언 코드**: `QueueBuilder.durable("work.queue").withArgument("x-dead-letter-exchange", "dlx.exchange").build()` 코드 패턴 암기 필수
- 점수: 4/10 — 꼬리 질문 "잘 모르겠습니다" (3회 연속)
