---
tags: [aws, sns, sqs, 면접질문, messaging]
related: [aws/concepts, kafka/questions, rabbitmq/questions]
---

# AWS SNS / SQS 면접 질문

→ 개념 상세: [[aws/concepts]]

---

## SNS와 SQS의 차이점을 설명해주세요.

**난이도**: 기초

**핵심 키워드**: Pub/Sub, Message Queue, Push vs Pull, 1:N vs 1:1

**모범 답변 방향**:
- SNS는 Pub/Sub 패턴의 알림 서비스로 Push 방식, 메시지를 다수 구독자에게 동시 전달
- SQS는 메시지 큐 서비스로 Pull(Polling) 방식, 메시지를 큐에 보관 후 소비자가 꺼내감
- 핵심 차이: SNS는 메시지를 보관하지 않아 구독자가 없으면 메시지 소멸, SQS는 최대 14일 보관
- 함께 쓰는 패턴: SNS가 이벤트 발행 → 여러 SQS 큐가 구독해 Fan-out 처리

**꼬리 질문 예시**:
- SNS 없이 SQS만 쓰면 안 되나요? 어떤 경우에 두 서비스를 함께 사용하나요?
- SNS 메시지 전달 실패 시 어떻게 처리하나요?

> 출처: https://aws-oncloudai.com/ko/aws-sns%EC%99%80-sqs%EC%9D%98-%EC%A3%BC%EC%9A%94-%EC%B0%A8%EC%9D%B4%EC%A0%90%EC%9D%80-%EB%AC%B4%EC%97%87%EC%9E%85%EB%8B%88%EA%B9%8C/

---

## SQS Standard Queue와 FIFO Queue의 차이점과 각각 언제 사용하나요?

**난이도**: 기초

**핵심 키워드**: 순서 보장, Exactly-once, 처리량, At-least-once

**모범 답변 방향**:
- Standard: 순서 미보장, 중복 전달 가능(At-least-once), 무제한 TPS → 이미지 리사이징, 로그 수집 등
- FIFO: 순서 완전 보장, 정확히 1회 처리(Exactly-once), 최대 300 TPS → 결제/주문/재고 처리 등
- FIFO는 처리량 제한이 있으므로 대용량 트래픽에 Standard + 멱등성 설계 조합이 현실적인 경우도 있음
- 멱등성 키(MessageDeduplicationId)로 FIFO에서 중복 메시지 방지

**꼬리 질문 예시**:
- 결제 처리에 Standard Queue를 쓰면 어떤 문제가 발생할 수 있나요?
- FIFO Queue 처리량 한계를 극복하려면 어떻게 설계하나요?

> 출처: https://cloud.in28minutes.com/aws-certification-amazon-sqs-simple-queuing-service

---

## Visibility Timeout이 무엇이며 왜 중요한가요?

**난이도**: 중급

**핵심 키워드**: 메시지 숨김, 중복 처리 방지, Consumer 처리 시간, 멱등성

**모범 답변 방향**:
- Consumer가 메시지를 꺼내간 뒤, 설정된 시간 동안 다른 Consumer에게 메시지를 숨기는 기능
- Consumer가 처리 완료 후 DeleteMessage를 호출하지 않거나 Timeout 전에 실패하면 메시지가 다시 visible
- Timeout이 너무 짧으면 처리 중 다른 Consumer가 같은 메시지를 가져가 중복 처리 발생
- 설정 기준: Consumer의 최대 처리 예상 시간보다 충분히 길게 설정 + 처리 성공 즉시 DeleteMessage 호출
- 멱등성 설계로 중복 처리가 발생해도 부작용이 없도록 보완

**꼬리 질문 예시**:
- Consumer가 처리 중 오래 걸릴 것 같으면 어떻게 대응할 수 있나요?
- Visibility Timeout과 DLQ의 maxReceiveCount는 어떤 관계인가요?

> 출처: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html

---

## Dead Letter Queue(DLQ)가 무엇이고 언제 사용하나요?

**난이도**: 중급

**핵심 키워드**: maxReceiveCount, 처리 실패 격리, 디버깅, 재처리

**모범 답변 방향**:
- 정해진 횟수(maxReceiveCount) 이상 처리 실패한 메시지를 원본 큐에서 격리해 저장하는 별도 큐
- 목적: 독성 메시지(Poison Message)가 큐를 막지 않도록 격리 + 원인 분석 후 재처리
- SQS DLQ(Consumer 처리 실패)와 SNS DLQ(구독자 전달 실패)를 구분해야 함
- 모니터링: `ApproximateNumberOfMessagesVisible` 지표로 CloudWatch 알람 설정
- 재처리: 원인 수정 후 `StartMessageMoveTask` API로 DLQ → 소스 큐 메시지 이동

**꼬리 질문 예시**:
- maxReceiveCount를 몇으로 설정하는 것이 적절한가요?
- DLQ의 메시지를 어떻게 재처리하나요? 직접 코드로 구현한 경험이 있나요?

> 출처: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html

---

## SNS + SQS Fan-out 패턴을 설명하고, 직접 설계해보세요.

**난이도**: 중급

**핵심 키워드**: Fan-out, 비동기 처리, 느슨한 결합, 확장성, OCP

**모범 답변 방향**:
- SNS Topic에 이벤트 발행 → 여러 SQS 큐가 구독 → 각 Consumer가 독립적으로 처리
- 예시: 주문 완료 이벤트 → SQS-이메일 발송 큐, SQS-재고 차감 큐, SQS-분석 저장 큐
- 장점: Consumer 장애 격리(한 큐 장애가 다른 큐에 미영향), 새 Consumer 추가 시 기존 코드 변경 불필요
- 각 SQS에 개별 DLQ 설정으로 실패 처리 독립화
- SNS 필터 정책으로 구독자마다 필요한 이벤트만 수신 가능

**꼬리 질문 예시**:
- Fan-out 패턴에서 SNS 없이 SQS를 여러 개 직접 쓰면 어떤 문제가 있나요?
- [[kafka]]의 Consumer Group과 SNS Fan-out의 차이점은 무엇인가요?

> 출처: https://awstip.com/the-sns-to-sqs-fan-out-pattern-lessons-learned-in-prod-8936a3158d80

---

## SQS와 Kafka를 비교했을 때 각각 어떤 상황에 선택하나요?

**난이도**: 심화

**핵심 키워드**: 메시지 리플레이, 파티션, 소비자 그룹, 관리 비용, 처리량

**모범 답변 방향**:
- **SQS 선택**: 완전 관리형으로 운영 부담 없음, 메시지 보관 후 소비하는 단순 큐 패턴, AWS 생태계 연동(Lambda 등)
- **Kafka 선택**: 메시지 리플레이 필요, 대용량 스트리밍 처리, 여러 Consumer Group이 같은 메시지를 독립적으로 소비
- SQS는 메시지를 처리 후 삭제하므로 **리플레이 불가** — 과거 이벤트 재처리가 필요하면 Kafka
- Kafka는 파티션으로 순서 보장 + 높은 처리량, SQS FIFO는 순서 보장하지만 TPS 제한
- 운영 측면: SQS는 AWS가 관리, Kafka는 클러스터 직접 운영 (MSK로 관리형 사용 가능)

**꼬리 질문 예시**:
- 실시간 채팅 메시지 저장/전달 시스템을 설계한다면 SQS와 Kafka 중 무엇을 선택하고 그 이유는?
- SQS에서 메시지 리플레이가 필요한 상황이 생기면 어떻게 대응하나요?

> 출처: https://aws.amazon.com/ko/blogs/korea/choosing-between-messaging-services-for-serverless-applications/
> 출처: https://reintech.io/blog/building-resilient-systems-aws-sqs-sns

---

## SQS에서 메시지 중복 처리를 방지하려면 어떻게 설계해야 하나요?

**난이도**: 심화

**핵심 키워드**: 멱등성(Idempotency), Exactly-once, FIFO, MessageDeduplicationId, 분산 환경

**모범 답변 방향**:
- **근본 원인**: SQS Standard는 At-least-once 전달이므로 동일 메시지가 여러 번 올 수 있음
- **FIFO Queue**: MessageDeduplicationId로 5분 내 중복 자동 제거 — 단, TPS 제한 존재
- **애플리케이션 레벨 멱등성**: 처리 전 DB/Redis에 MessageId 저장, 이미 처리된 ID면 스킵
- **DB Upsert/조건부 업데이트**: 상태 기반 조건 체크로 중복 실행해도 결과 동일하게 설계
- Visibility Timeout 적절히 설정 + 처리 완료 즉시 DeleteMessage 호출로 중복 노출 최소화

**꼬리 질문 예시**:
- 결제 처리 서비스에서 멱등성을 어떻게 보장하나요? 구체적인 설계를 설명해주세요.
- Redis를 활용한 멱등성 키 관리의 장단점은?

> 출처: https://amitech.medium.com/complete-beginners-guide-to-aws-sqs-simple-queue-service-in-2025-5d9528eee1b3
