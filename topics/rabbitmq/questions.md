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
- Exchange 타입은 "라우팅 방식" 기준으로 설명
  - Direct = routing key 완전 일치
  - Fanout = 모든 바인딩 큐에 브로드캐스트
  - Topic = 와일드카드 패턴 매칭 (`*`, `#`)
  - Headers = 헤더 속성 기반 라우팅
- Kafka vs RabbitMQ: 재처리/exactly-once/수평확장 → Kafka, 복잡한 라우팅/Task Queue → RabbitMQ
- 이력서 연결: 트랜스코더 오케스트레이션 → Direct Exchange로 인스턴스별 작업 분배 + Event-Driven 전환

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

---

## RabbitMQ Dead Letter Exchange 활용 패턴

**난이도**: 중급

**핵심 키워드**: DLX, Dead Letter Queue, nack, requeue, TTL, 재처리

**모범 답변 방향**:
Dead Letter Exchange는 RabbitMQ에서 처리에 실패하거나 TTL이 만료된 메시지를 유실시키지 않고 별도 큐로 보내는 패턴입니다. Consumer가 메시지를 처리하다 오류가 발생하면 `nack(requeue=false)`를 호출하고, 해당 메시지는 DLX를 거쳐 Dead Letter Queue로 라우팅됩니다. requeue=true를 사용하면 메시지가 원래 큐 앞으로 돌아와 무한 재시도 루프가 발생할 수 있어 실무에서는 DLX 연계가 안전합니다. TTL이 만료된 메시지도 동일하게 DLX로 이동하기 때문에, 한 번의 DLX 설정으로 실패와 만료를 모두 처리할 수 있습니다. DLQ에 쌓인 메시지는 별도 Consumer가 모니터링·알림을 보내거나, 원인 분석 후 원래 큐에 재발행하는 방식으로 처리합니다. 트레이드오프는 DLX 설정이 없으면 메시지가 단순히 버려지거나 무한 루프에 빠지지만, DLX를 도입하면 운영 복잡도가 높아지고 DLQ도 주기적으로 관리해야 한다는 점입니다.

**꼬리 질문 예시**:
- "DLX 없이 nack하면 어떻게 되나요?" → requeue=true면 큐 앞으로 돌아와 무한 재시도 루프 발생 위험
- "Kafka의 DLQ와 비교하면?" → Kafka는 offset 재조정으로 재처리 가능, DLQ 패턴은 별도 topic으로 구현
