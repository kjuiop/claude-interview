---
tags: [aws, sns, sqs, messaging, cloud]
related: [kafka, rabbitmq, distributed-systems]
---

# AWS SNS / SQS 핵심 개념

## SNS (Simple Notification Service)

**Pub/Sub 방식의 완전관리형 알림 서비스**. 발행자(Publisher)가 **Topic**에 메시지를 보내면, 해당 토픽을 구독(Subscribe)한 엔드포인트 전체에 동시에 메시지를 전달한다.

### 주요 특징
- **Push 방식**: SNS가 구독자에게 능동적으로 메시지를 밀어넣음
- **1:N 팬아웃**: 하나의 메시지를 여러 구독자(SQS, Lambda, HTTP, Email, SMS 등)에게 동시 전달
- **메시지 보관 없음**: 구독자가 없거나 전달 실패 시 메시지 소멸 (DLQ 설정 시 보존 가능)
- **전달 보장**: 최소 1회 전달 (at-least-once), 구독자별 재시도 정책 설정 가능
- **필터 정책**: 구독자마다 메시지 속성 기반 필터링으로 관심 메시지만 수신

### 지원 구독 프로토콜
| 프로토콜 | 사용 예 |
|---|---|
| SQS | 비동기 처리, Fan-out 패턴 |
| Lambda | 서버리스 이벤트 처리 |
| HTTP/HTTPS | 웹훅 기반 외부 시스템 연동 |
| Email / SMS | 사람에게 직접 알림 |
| Mobile Push | iOS/Android 푸시 알림 |

---

## SQS (Simple Queue Service)

**완전관리형 메시지 큐 서비스**. 메시지를 큐에 저장하고, 소비자(Consumer)가 폴링으로 메시지를 가져가 처리한다. [[kafka]]의 Consumer 그룹과 유사하지만 단일 소비자 모델이 기본이다.

### 주요 특징
- **Pull(Polling) 방식**: 소비자가 주기적으로 큐에서 메시지를 요청
- **1:1 처리**: 하나의 메시지는 하나의 소비자만 처리 (그룹 내 경쟁 소비)
- **메시지 보관**: 소비자가 삭제하기 전까지 최대 **14일** 보관
- **Visibility Timeout**: 메시지를 꺼내간 동안 다른 소비자에게 숨김 (기본 30초, 최대 12시간)
- **At-least-once**: 처리 후 명시적 삭제 필요, 멱등성 설계 권장

### Standard Queue vs FIFO Queue

| 항목 | Standard | FIFO |
|---|---|---|
| 순서 보장 | 미보장 (Best-effort) | 완전 보장 |
| 중복 전달 | 발생 가능 (At-least-once) | 정확히 1회 (Exactly-once) |
| 처리량 | 무제한 TPS | 최대 300 TPS (배치 시 3,000) |
| 사용 예 | 이미지 리사이징, 로그 수집 | 결제, 주문 처리, 재고 차감 |

### Visibility Timeout 동작 원리
```
Consumer A가 메시지 수신
  └─ 메시지가 30초간 다른 Consumer에게 invisible
  └─ 30초 내 처리 + DeleteMessage 호출 → 메시지 영구 삭제
  └─ 30초 초과 → 메시지 다시 visible (재처리 가능)
```
→ Consumer 처리 시간보다 Visibility Timeout을 충분히 길게 설정해야 중복 처리를 방지할 수 있다.

---

## Dead Letter Queue (DLQ)

처리에 반복 실패한 메시지를 격리하는 별도 큐. **maxReceiveCount** 초과 시 원본 큐에서 DLQ로 이동.

### 설정 원칙
- Standard 큐 → Standard DLQ, FIFO 큐 → FIFO DLQ (타입 일치 필수)
- 소스 큐와 DLQ는 **같은 리전, 같은 계정**에 위치
- DLQ 메시지를 모니터링해 알람 설정 (`ApproximateNumberOfMessagesVisible` 지표 사용)
- 원인 분석 후 재처리 시 `StartMessageMoveTask` API로 DLQ → 소스 큐 이동

### SNS DLQ vs SQS DLQ
- **SQS DLQ**: Consumer 처리 실패 시 메시지 격리
- **SNS DLQ**: 구독자에게 전달 실패 시 메시지 격리 (구독별로 설정)

---

## Fan-Out 패턴 (SNS + SQS 조합)

실무에서 가장 많이 사용하는 패턴. SNS가 이벤트를 발행하면 여러 SQS 큐가 각자 독립적으로 처리.

```
[Publisher]
    │
    ▼
[SNS Topic]
    ├──▶ [SQS Queue A] → [Consumer A: 이메일 발송]
    ├──▶ [SQS Queue B] → [Consumer B: 재고 업데이트]
    └──▶ [SQS Queue C] → [Consumer C: 분석 데이터 저장]
```

**장점**:
- 각 Consumer가 독립적인 속도/재시도 정책으로 처리
- Consumer 하나가 장애나도 다른 큐에 영향 없음
- 새 Consumer 추가 시 기존 코드 변경 불필요 (OCP)

> 출처: https://awstip.com/the-sns-to-sqs-fan-out-pattern-lessons-learned-in-prod-8936a3158d80

---

## SNS vs SQS 핵심 비교

| 항목 | SNS | SQS |
|---|---|---|
| 패턴 | Pub/Sub (Push) | Message Queue (Pull) |
| 수신자 수 | 1:N (다수 구독자 동시) | 1:1 (경쟁 소비) |
| 메시지 보관 | 없음 (전달 즉시 소멸) | 최대 14일 |
| 처리 순서 | 미보장 | FIFO 큐 사용 시 보장 |
| 주 사용 사례 | 이벤트 브로드캐스트, Fan-out | 작업 큐, 비동기 처리 |
| [[kafka]] 비교 | 토픽/구독자 개념 유사 | 파티션 없음, 리플레이 불가 |
| [[rabbitmq]] 비교 | Exchange 팬아웃과 유사 | 큐 구조 동일 개념 |

> 출처: https://awsfundamentals.com/blog/aws-sns-vs-sqs-what-are-the-main-differences
> 출처: https://melonicedlatte.com/2023/11/20/141900.html
