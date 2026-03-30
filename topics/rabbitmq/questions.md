---
tags: [rabbitmq, messaging, amqp, interview-questions]
related: [kafka, distributed-systems]
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

---

## RabbitMQ Dead Letter Exchange 활용 패턴

**난이도**: 중급

**핵심 키워드**: DLX, Dead Letter Queue, nack, requeue, TTL, 재처리

**모범 답변 방향**:
- 처리 실패 메시지를 DLX → DLQ로 라우팅하여 유실 방지
- 재처리 로직 또는 모니터링/알림 연계
- TTL 만료 메시지도 DLX로 이동 가능

**꼬리 질문 예시**:
- "DLX 없이 nack하면 어떻게 되나요?" → requeue=true면 큐 앞으로 돌아와 무한 재시도 루프 발생 위험
- "Kafka의 DLQ와 비교하면?" → Kafka는 offset 재조정으로 재처리 가능, DLQ 패턴은 별도 topic으로 구현
