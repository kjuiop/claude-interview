---
tags: [aws, sns, sqs, 면접질문, messaging, vpc, ec2, rds, infrastructure]
related: [aws/concepts, kafka/questions, rabbitmq/questions]
---

# AWS 면접 질문

## AWS 백엔드 인프라 아키텍처

### AWS 주요 서비스(EC2, S3, RDS, ElastiCache)의 역할 차이를 설명해주세요. 백엔드 배포 시 일반적인 아키텍처는 어떻게 구성하나요?

**난이도**: 기초

**핵심 키워드**: EC2(VM), S3(Object Storage), RDS(Managed RDBMS), ElastiCache(Redis/Memcached), VPC, Public/Private Subnet, ALB, NAT Gateway, Security Group, ASG, Multi-AZ

**모범 답변 방향**:
- **EC2**: 하이퍼바이저 위에서 실행되는 **가상 머신(VM)**. "물리 서버"가 아님 — 면접에서 주의
- **S3**: 객체 스토리지. 이미지/로그/영상 저장, 정적 웹사이트 호스팅, 데이터 레이크
- **RDS**: 패치/백업/복제를 AWS가 관리하는 관계형 DB. MySQL, Aurora, PostgreSQL, Oracle 등 지원
- **ElastiCache**: Redis 또는 Memcached 엔진의 관리형 캐시. 캐싱 외에 분산락/pub-sub/세션 스토어로도 활용

**일반적인 아키텍처 (트래픽 흐름)**:
```
인터넷 → [Internet Gateway] → ALB (Public Subnet)
         → EC2 (Private Subnet) ← Security Group: ALB SG만 허용
         → RDS / ElastiCache (Private Subnet)
Private Subnet → NAT Gateway (Public Subnet) → 인터넷 (아웃바운드)
```

**핵심 포인트**:
- **Public Subnet**: ALB, NAT Gateway, Bastion Host
- **Private Subnet**: EC2, RDS, ElastiCache
- **Security Group**: ALB SG에서만 EC2 인바운드 허용. "인증·인가"와 다름 — SG는 IP/포트 기반 방화벽
- **NAT Gateway**: Private Subnet EC2의 아웃바운드(패키지 설치, 외부 API 호출) 통신 담당
- **ALB 라우팅 기준**: path (`/api/*`), header, host — 지리적 위치 라우팅은 **Route 53**이 담당
- **ASG**: CPU 사용률, 커스텀 CloudWatch 메트릭 기준으로 최소/최대/희망 인스턴스 수 관리. Multi-AZ 배포로 가용성 확보

**꼬리 질문 예시**:
- EC2가 외부 npm 패키지를 설치해야 하는데 Private Subnet에 있다면 어떻게 구성하나요? → NAT Gateway
- ALB에서 특정 경로를 다른 Target Group으로 보내려면? → Listener Rule (path-based routing)
- Route 53과 ALB의 역할 차이는? → Route 53: DNS 레벨 라우팅(지리, 가중치, Failover). ALB: HTTP 레벨 라우팅(path, header)

**면접 세션 피드백 (2026-04-07)**:
- 잘한 점: Public/Private 분리 개념, ALB/EC2/RDS 배치 방향 맞음
- 보완: "물리 서버" → "가상 머신(VM)". "인증·인가" → "Security Group 인바운드 규칙". ALB 지리적 위치 → Route 53. NAT Gateway 미언급. Multi-AZ 미언급

---

# AWS SNS / SQS 면접 질문

→ 개념 상세: [[aws/concepts]]

---

## SNS와 SQS의 차이점을 설명해주세요.

**난이도**: 기초

**핵심 키워드**: Pub/Sub, Message Queue, Push vs Pull, 1:N vs 1:1

**모범 답변 방향**:

SNS와 SQS는 근본적으로 다른 패턴을 구현합니다. SNS는 Pub/Sub 패턴의 알림 서비스로, Publisher가 Topic에 메시지를 발행하면 SNS가 구독자 전체에 Push 방식으로 동시 전달합니다. SQS, Lambda, HTTP, Email 등 다양한 엔드포인트를 구독자로 등록할 수 있습니다. 반면 SQS는 메시지 큐 서비스로 Pull(Polling) 방식이며, Consumer가 주기적으로 큐에서 메시지를 가져가 처리합니다. 핵심 차이는 메시지 보관 여부입니다. SNS는 메시지를 보관하지 않아 구독자가 없거나 전달에 실패하면 메시지가 소멸하지만, SQS는 Consumer가 명시적으로 삭제하기 전까지 최대 14일간 보관합니다. 실무에서는 이 두 서비스를 조합한 Fan-out 패턴을 많이 사용합니다. SNS Topic에 이벤트를 발행하면 여러 SQS 큐가 각각 구독해 이메일 발송, 재고 차감, 분석 저장을 독립적으로 처리하는 구조입니다.

**꼬리 질문 예시**:
- SNS 없이 SQS만 쓰면 안 되나요? 어떤 경우에 두 서비스를 함께 사용하나요?
- SNS 메시지 전달 실패 시 어떻게 처리하나요?

> 출처: https://aws-oncloudai.com/ko/aws-sns%EC%99%80-sqs%EC%9D%98-%EC%A3%BC%EC%9A%94-%EC%B0%A8%EC%9D%B4%EC%A0%90%EC%9D%80-%EB%AC%B4%EC%97%87%EC%9E%85%EB%8B%88%EA%B9%8C/

---

## SQS Standard Queue와 FIFO Queue의 차이점과 각각 언제 사용하나요?

**난이도**: 기초

**핵심 키워드**: 순서 보장, Exactly-once, 처리량, At-least-once

**모범 답변 방향**:

Standard Queue와 FIFO Queue는 순서 보장과 처리량 면에서 상반된 특성을 가집니다. Standard Queue는 순서를 보장하지 않고 At-least-once 전달 방식이라 중복 전달이 발생할 수 있지만, 처리량에 제한이 없어 이미지 리사이징, 로그 수집처럼 순서보다 처리량이 중요한 작업에 적합합니다. FIFO Queue는 메시지의 전송 순서를 완전히 보장하고 Exactly-once 처리를 지원하지만, 최대 300 TPS(배치 모드에서 3,000 TPS)라는 처리량 제한이 있습니다. 결제, 주문, 재고 차감처럼 순서와 중복 처리 방지가 중요한 작업에 사용합니다. 주의할 점은 FIFO의 처리량 제한으로 인해 대용량 트래픽 환경에서는 Standard Queue에 멱등성 설계를 조합하는 방식이 더 현실적인 선택이 될 수 있다는 것입니다. FIFO Queue에서는 `MessageDeduplicationId`로 5분 내 중복 메시지를 자동 제거할 수 있습니다.

**꼬리 질문 예시**:
- 결제 처리에 Standard Queue를 쓰면 어떤 문제가 발생할 수 있나요?
- FIFO Queue 처리량 한계를 극복하려면 어떻게 설계하나요?

> 출처: https://cloud.in28minutes.com/aws-certification-amazon-sqs-simple-queuing-service

---

## Visibility Timeout이 무엇이며 왜 중요한가요?

**난이도**: 중급

**핵심 키워드**: 메시지 숨김, 중복 처리 방지, Consumer 처리 시간, 멱등성

**모범 답변 방향**:

Visibility Timeout은 Consumer가 메시지를 큐에서 꺼내간 뒤 설정된 시간 동안 다른 Consumer에게 해당 메시지를 보이지 않게 숨기는 기능입니다. SQS에서 메시지를 Poll하면 바로 삭제되는 것이 아니라 처리 시간 동안 다른 Consumer가 가져가지 못하도록 invisible 상태가 됩니다. Consumer가 처리를 완료하면 `DeleteMessage`를 호출해 메시지를 영구 삭제해야 하며, Timeout 내에 삭제하지 못하면 메시지가 다시 visible 상태로 돌아옵니다. 이 기능이 중요한 이유는 At-least-once 전달 보장을 구현하면서도 동시 중복 처리를 방지하기 위해서입니다. Timeout이 너무 짧으면 처리 중인 메시지가 다시 visible되어 다른 Consumer가 동일 메시지를 가져가 중복 처리가 발생합니다. 반대로 너무 길면 Consumer가 실패했을 때 메시지 재처리가 오래 지연됩니다. 설정 기준은 Consumer의 최대 처리 예상 시간보다 충분히 길게 설정하고, 처리 성공 즉시 `DeleteMessage`를 호출하는 것입니다. 추가로 멱등성 설계를 적용해 중복 처리가 발생하더라도 부작용이 없도록 보완하는 것이 권장됩니다.

**꼬리 질문 예시**:
- Consumer가 처리 중 오래 걸릴 것 같으면 어떻게 대응할 수 있나요?
- Visibility Timeout과 DLQ의 maxReceiveCount는 어떤 관계인가요?

> 출처: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html

---

## Dead Letter Queue(DLQ)가 무엇이고 언제 사용하나요?

**난이도**: 중급

**핵심 키워드**: maxReceiveCount, 처리 실패 격리, 디버깅, 재처리

**모범 답변 방향**:

Dead Letter Queue는 정해진 횟수(maxReceiveCount) 이상 처리에 실패한 메시지를 원본 큐에서 격리해 저장하는 별도 큐입니다. 주목적은 독성 메시지(Poison Message)가 계속 재처리 시도를 반복하며 큐를 막는 것을 방지하고, 원인 분석 후 수정해서 재처리할 수 있도록 격리하는 것입니다. 주의할 점은 SQS DLQ와 SNS DLQ를 구분해야 한다는 것입니다. SQS DLQ는 Consumer가 처리에 실패했을 때 격리하는 반면, SNS DLQ는 구독자에게 전달 자체가 실패했을 때 격리하며 구독별로 각각 설정합니다. 운영 측면에서는 DLQ의 `ApproximateNumberOfMessagesVisible` 지표를 CloudWatch 알람으로 모니터링해 문제를 즉시 감지하는 것이 중요합니다. 원인 수정 후 재처리 시에는 `StartMessageMoveTask` API를 사용해 DLQ에서 소스 큐로 메시지를 이동할 수 있습니다.

**꼬리 질문 예시**:
- maxReceiveCount를 몇으로 설정하는 것이 적절한가요?
- DLQ의 메시지를 어떻게 재처리하나요? 직접 코드로 구현한 경험이 있나요?

> 출처: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html

---

## SNS + SQS Fan-out 패턴을 설명하고, 직접 설계해보세요.

**난이도**: 중급

**핵심 키워드**: Fan-out, 비동기 처리, 느슨한 결합, 확장성, OCP

**모범 답변 방향**:

Fan-out 패턴은 SNS Topic에 이벤트를 발행하면 여러 SQS 큐가 구독해 각 Consumer가 독립적으로 처리하는 구조입니다. 예를 들어 주문 완료 이벤트가 발행되면 이메일 발송 큐, 재고 차감 큐, 분석 데이터 저장 큐가 동시에 각자의 작업을 처리합니다. 이 패턴의 핵심 장점은 Consumer 간의 완전한 격리입니다. 이메일 발송 Consumer가 장애가 나도 재고 차감 처리에는 영향이 없습니다. 또한 새로운 Consumer를 추가할 때 기존 코드를 변경하지 않고 SNS 구독만 추가하면 되므로 OCP 원칙에 부합합니다. 각 SQS에 개별 DLQ를 설정해 실패 처리를 독립화하고, SNS 필터 정책으로 구독자마다 필요한 이벤트 메시지만 선택적으로 수신하도록 구성할 수 있습니다. 만약 SNS 없이 SQS를 여러 개 직접 쓰면 Publisher가 모든 큐에 직접 메시지를 보내야 해 새 Consumer 추가 시 Publisher 코드를 수정해야 하는 강한 결합이 발생합니다.

**꼬리 질문 예시**:
- Fan-out 패턴에서 SNS 없이 SQS를 여러 개 직접 쓰면 어떤 문제가 있나요?
- [[kafka]]의 Consumer Group과 SNS Fan-out의 차이점은 무엇인가요?

> 출처: https://awstip.com/the-sns-to-sqs-fan-out-pattern-lessons-learned-in-prod-8936a3158d80

---

## SQS와 Kafka를 비교했을 때 각각 어떤 상황에 선택하나요?

**난이도**: 심화

**핵심 키워드**: 메시지 리플레이, 파티션, 소비자 그룹, 관리 비용, 처리량

**모범 답변 방향**:

SQS와 Kafka의 선택은 메시지 리플레이 필요성과 운영 부담을 기준으로 결정합니다. SQS는 완전 관리형 서비스로 운영 부담이 없고 AWS 생태계(Lambda, SNS)와 긴밀하게 연동됩니다. 메시지를 소비자가 처리 완료 후 삭제하는 단순 큐 패턴에 적합하며, 과거 이벤트 재처리(리플레이)가 불필요한 환경에 잘 맞습니다. 반면 Kafka는 메시지를 보존 기간 동안 유지하기 때문에 리플레이가 가능하며, 여러 Consumer Group이 같은 메시지를 독립적으로 소비하는 패턴을 지원합니다. 파티션을 통해 순서를 보장하면서도 높은 처리량을 낼 수 있어 대용량 스트리밍 처리에 강합니다. 반면 SQS FIFO는 순서를 보장하지만 TPS 제한이 있습니다. 운영 측면에서 SQS는 AWS가 전적으로 관리하지만 Kafka는 클러스터를 직접 운영해야 하며, AWS MSK를 사용하면 관리형으로 운영할 수 있습니다. 이벤트 리플레이 요구사항이 크지 않은 환경에서는 SNS+SQS가 Kafka보다 운영 부담이 적어 적합합니다.

**꼬리 질문 예시**:
- 실시간 채팅 메시지 저장/전달 시스템을 설계한다면 SQS와 Kafka 중 무엇을 선택하고 그 이유는?
- SQS에서 메시지 리플레이가 필요한 상황이 생기면 어떻게 대응하나요?

> 출처: https://aws.amazon.com/ko/blogs/korea/choosing-between-messaging-services-for-serverless-applications/
> 출처: https://reintech.io/blog/building-resilient-systems-aws-sqs-sns

**면접 세션 피드백 (2026-04-07 3회차)**:
- 잘한 점: SNS fan-out → 여러 SQS 구독 패턴 즉시 설계. Outbox/Inbox 패턴, 멱등성(MessageDeduplicationId) 언급.
- 보완: **SQS 핵심 가치** — visibility timeout(처리 중 중복 소비 방지) + DLQ(실패 격리) 반드시 추가. **선택 결론 마무리** — *"와그처럼 이벤트 리플레이 요구사항이 크지 않은 환경에서는 SNS+SQS가 Kafka보다 운영 부담이 적어 적합"*

---

## SQS에서 메시지 중복 처리를 방지하려면 어떻게 설계해야 하나요?

**난이도**: 심화

**핵심 키워드**: 멱등성(Idempotency), Exactly-once, FIFO, MessageDeduplicationId, 분산 환경

**모범 답변 방향**:

SQS에서 메시지 중복 처리 방지는 다층적으로 설계해야 합니다. 근본 원인은 SQS Standard가 At-least-once 전달 방식이라 네트워크 재전송 등으로 동일 메시지가 여러 번 전달될 수 있다는 점입니다. 첫 번째 방어선은 FIFO Queue를 사용하는 것으로, `MessageDeduplicationId`를 통해 5분 내 동일 ID의 중복 메시지를 자동으로 제거합니다. 단, TPS 제한이 있으므로 대용량 환경에서는 한계가 있습니다. 두 번째 방어선은 애플리케이션 레벨 멱등성 구현입니다. 메시지를 처리하기 전에 DB 또는 Redis에 MessageId를 저장하고, 이미 처리된 ID가 들어오면 스킵하는 방식입니다. 세 번째 방어선은 DB 레벨의 멱등성으로, `Upsert`나 조건부 업데이트를 사용해 중복 실행해도 결과가 동일하도록 설계합니다. 예를 들어 주문 상태를 UPDATE할 때 현재 상태가 예상한 상태일 때만 업데이트하도록 조건을 추가합니다. 운영 차원에서는 Visibility Timeout을 충분히 설정하고 처리 완료 즉시 `DeleteMessage`를 호출해 중복 노출 가능성 자체를 최소화하는 것이 중요합니다.

**꼬리 질문 예시**:
- 결제 처리 서비스에서 멱등성을 어떻게 보장하나요? 구체적인 설계를 설명해주세요.
- Redis를 활용한 멱등성 키 관리의 장단점은?

> 출처: https://amitech.medium.com/complete-beginners-guide-to-aws-sqs-simple-queue-service-in-2025-5d9528eee1b3
