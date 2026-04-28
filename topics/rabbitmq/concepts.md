---
tags: [rabbitmq, messaging, amqp, backend]
related: [kafka, distributed-systems, zookeeper]
---

# RabbitMQ — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/rabbitmq/questions]] | 비교: [[topics/kafka/concepts]]

---

## Exchange 타입

Exchange는 Producer가 보낸 메시지를 어떤 Queue로 라우팅할지 결정하는 규칙이다.

| 타입 | 라우팅 방식 | 사용 케이스 |
|---|---|---|
| **Direct** | routing key **완전 일치** → 해당 큐로만 전송 | 특정 인스턴스/서비스에 작업 분배 |
| **Fanout** | routing key 무시 → **바인딩된 모든 큐**에 브로드캐스트 | 이벤트 알림, 캐시 무효화 전파 |
| **Topic** | `*.order.#` 같은 **와일드카드 패턴 매칭** | 멀티테넌트, 서비스별 이벤트 필터링 |
| **Headers** | routing key 대신 **메시지 헤더 속성**으로 라우팅 | 복잡한 필터 조건, routing key가 부족한 경우 |

**와일드카드 규칙 (Topic Exchange):**
- `*`: 단어 하나 대체 (e.g., `*.order.*`)
- `#`: 0개 이상의 단어 대체 (e.g., `order.#`)

**Direct Exchange가 1:1이 아닌 이유:**
같은 routing key를 여러 큐가 바인딩하면 1:N도 가능. "routing key 완전 일치"가 핵심 개념.

---

## 메시지 신뢰성 — Ack/Nack

- **auto ack**: 메시지 수신 즉시 큐에서 삭제 → 처리 실패 시 유실
- **manual ack**: Consumer가 명시적으로 `ack` 호출해야 삭제 → 처리 완료 보장
- **nack + requeue**: 처리 실패 시 큐에 재삽입 → Dead Letter Exchange로 연계 가능

---

## Dead Letter Exchange (DLX)

처리에 실패하거나 TTL이 만료된 메시지를 별도 큐로 보내는 패턴.

```
처리 실패/TTL 만료
  → Dead Letter Exchange
  → Dead Letter Queue
  → 모니터링/재처리/알림
```

---

## Kafka와의 차이

| 항목 | RabbitMQ | Kafka |
|---|---|---|
| 메시지 전달 | Push (브로커가 Consumer에게 푸시) | Pull (Consumer가 브로커에서 가져옴) |
| 메시지 보존 | Ack 후 삭제 | 설정한 retention 기간 동안 유지 |
| 재처리 | DLX로 제한적 | offset 재조정으로 자유롭게 재처리 |
| exactly-once | 어렵 (복잡한 설정 필요) | `enable.idempotence=true` + transaction + `isolation.level=read_committed` |
| 라우팅 유연성 | Exchange 타입으로 복잡한 라우팅 가능 | Topic + Partition으로 단순 분산 |
| 수평 확장 | 큐 추가로 확장 | 파티션 추가로 수평 확장 |
| 적합한 사용 | Task Queue, 복잡한 라우팅, 미들웨어 제어 | 대용량 이벤트 스트리밍, 로그, 재처리 필요한 데이터 |

---

## 실무 선택 기준

**RabbitMQ를 선택하는 경우:**
- 여러 미들웨어/서비스에 작업을 유연하게 분배해야 할 때 (Direct/Topic Exchange 활용)
- 작업 완료 보장이 중요한 Task Queue (manual ack)
- 복잡한 라우팅 조건이 필요한 경우

**Kafka를 선택하는 경우:**
- 데이터 정합성이 중요하고 exactly-once가 필요한 경우 (이커머스, 결제)
- 메시지 재처리가 필요한 경우 (offset 기반 재조정)
- 대용량 트래픽 + 수평 확장이 필요한 경우 (파티션)
- 로그/이벤트 소싱, 감사(audit) 데이터

**면접 세션 피드백 (2026-03-30 1회차)**:
- Kafka exactly-once 관련 실무 설정 언급 강점 (idempotent producer, transaction, read_committed)
- RabbitMQ 트랜스코더 오케스트레이션 연결 기회 — 카테노이드에서 Direct Exchange로 트랜스코더 인스턴스별 큐 바인딩 + Event-Driven 전환으로 분당 70회 헬스체크 제거
