---
tags: [golang, go, ddd, domain-driven-design, bounded-context, modular-monolith, hexagonal]
related: [hexagonal-architecture, clean-architecture, distributed-systems, system-design]
---

# Golang — DDD + Modular Monolith + Hexagonal

→ [[home]] | [[topics/golang/hexagonal-architecture]] | [[topics/golang/clean-architecture]] | [[topics/distributed-systems/concepts]]

---

## 핵심 아이디어

> "각 도메인(user, product, payment)을 높은 응집도로 묶고, 도메인 간 의존성은 Port/Adapter로 역전시킨다.
> 이렇게 하면 나중에 특정 도메인을 마이크로서비스로 분리할 때 인프라 레이어만 교체하면 된다."

이 패턴은 **Modular Monolith + DDD Bounded Context + Hexagonal Architecture** 의 조합이다.
Go에서도 이 패턴을 정확히 사용하며, 실무에서 점점 늘어나는 추세.

---

## Bounded Context — 도메인 경계 식별

**Bounded Context**: 각 도메인 모델이 명확한 경계를 가져야 한다는 DDD 원칙.
같은 개념도 도메인마다 다르게 해석된다.

| 도메인 | User를 부르는 이름 | 관심 데이터 |
|---|---|---|
| User | Member | ID, Email, Password, Balance |
| Order | Orderer | ID, ShippingAddress |
| Payment | Payer | ID, Balance, CardInfo |

→ `User` 엔티티를 하나로 공유하면 도메인마다 서로의 변경이 충돌.
→ 각 도메인이 자기 관점의 모델만 갖는 것이 핵심.

---

## 도메인 간 의존성 역전 패턴

**문제**: Payment 도메인이 User 도메인의 잔액을 차감해야 한다.
Payment가 User 패키지를 직접 import하면 강한 결합 → 분리 불가.

**해결**: Payment 도메인 안에 Port(인터페이스)를 정의하고, Adapter로 연결.

```
User Domain          │  Adapter Layer          │  Payment Domain
─────────────────────┼─────────────────────────┼───────────────────
UserService struct   │  UserAdapter struct      │  port.UserService
  GetUser()          │    GetUser() {           │  interface {
  DeductBalance()    │      user.GetUser()      │    GetUser()
                     │    }                     │    DeductBalance()
                     │                          │  }
```

**핵심**: Payment Domain은 User Domain 패키지를 import하지 않는다.
오직 자신이 정의한 `port.UserService` 인터페이스에만 의존한다.

---

## 프로젝트 구조 (Modular Monolith)

```
project/
├── cmd/
│   └── api/main.go              # Adapter 조립 + 서버 실행
│
├── internal/
│   ├── shared/                  # 도메인 간 공유 (최소화)
│   │   ├── event/               # 도메인 이벤트 정의
│   │   └── errors/              # 공통 에러 타입
│   │
│   ├── user/                    # User Bounded Context
│   │   ├── domain/
│   │   │   ├── user.go          # User 엔티티, 비즈니스 규칙
│   │   │   └── repository.go    # UserRepository interface
│   │   ├── application/
│   │   │   └── service.go       # Use case 구현
│   │   └── infrastructure/
│   │       ├── postgres/
│   │       │   └── repository.go
│   │       └── http/
│   │           └── handler.go
│   │
│   ├── product/                 # Product Bounded Context
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │
│   ├── payment/                 # Payment Bounded Context
│   │   ├── domain/
│   │   │   └── payment.go
│   │   ├── application/
│   │   │   └── service.go       # port.UserService 에만 의존
│   │   ├── port/                # ← 핵심: Payment가 정의하는 인터페이스
│   │   │   └── dependencies.go  # type UserService interface { ... }
│   │   └── infrastructure/
│   │       └── postgres/
│   │
│   └── adapters/                # 도메인 간 연결 (Wiring Layer)
│       ├── user_for_payment.go  # User → payment/port.UserService 구현
│       └── user_for_order.go    # User → order/port.UserService 구현
│
└── go.mod
```

**디렉터리 원칙:**
- `internal/{domain}/` — 이 도메인만 아는 코드. 외부에서 직접 import 불가
- `internal/{domain}/port/` — 이 도메인이 다른 도메인에게 기대하는 인터페이스
- `internal/adapters/` — 도메인 간 Port를 연결하는 글루 코드
- `internal/shared/` — 최소한의 공유 (이벤트 타입, 공통 에러만)

---

## 핵심 코드 패턴

### 1. 각 도메인의 Port 정의 (Payment 관점)

```go
// internal/payment/port/dependencies.go
package port

import "context"

// Payment 도메인이 User 도메인에게 기대하는 계약
// User 도메인 코드를 전혀 import하지 않음
type UserService interface {
    GetBalance(ctx context.Context, userID string) (int64, error)
    DeductBalance(ctx context.Context, userID string, amount int64) error
}

// Payment 도메인이 Product 도메인에게 기대하는 계약
type ProductService interface {
    GetPrice(ctx context.Context, productID string) (int64, error)
    DeductStock(ctx context.Context, productID string, qty int) error
}
```

### 2. Payment Service — Port에만 의존

```go
// internal/payment/application/service.go
package application

import (
    "project/internal/payment/domain"
    "project/internal/payment/port"  // 자신의 port만 import
)

type PaymentService struct {
    userSvc    port.UserService    // ← User 패키지 직접 import 없음
    productSvc port.ProductService // ← Product 패키지 직접 import 없음
    repo       domain.PaymentRepository
}

func (s *PaymentService) ProcessPayment(ctx context.Context, req ProcessPaymentRequest) error {
    // User 도메인 호출 — Port 인터페이스를 통해서만
    balance, err := s.userSvc.GetBalance(ctx, req.UserID)
    if err != nil {
        return fmt.Errorf("get balance: %w", err)
    }
    if balance < req.Amount {
        return domain.ErrInsufficientBalance
    }

    // Product 재고 차감
    if err := s.productSvc.DeductStock(ctx, req.ProductID, 1); err != nil {
        return fmt.Errorf("deduct stock: %w", err)
    }

    // 사용자 잔액 차감
    if err := s.userSvc.DeductBalance(ctx, req.UserID, req.Amount); err != nil {
        // 재고 복구가 필요 → 보상 트랜잭션 또는 Saga 패턴
        return fmt.Errorf("deduct balance: %w", err)
    }

    return s.repo.Save(ctx, domain.NewPayment(req.UserID, req.ProductID, req.Amount))
}
```

### 3. Adapter — User Domain을 Payment Port로 연결

```go
// internal/adapters/user_for_payment.go
package adapters

import (
    "context"
    userapp "project/internal/user/application"  // User 패키지 import는 여기서만
    "project/internal/payment/port"
)

// payment/port.UserService 를 구현하는 Adapter
type UserAdapterForPayment struct {
    svc *userapp.UserService
}

func NewUserAdapterForPayment(svc *userapp.UserService) port.UserService {
    return &UserAdapterForPayment{svc: svc}
}

func (a *UserAdapterForPayment) GetBalance(ctx context.Context, userID string) (int64, error) {
    user, err := a.svc.GetUser(ctx, userID)
    if err != nil {
        return 0, err
    }
    return user.Balance, nil
}

func (a *UserAdapterForPayment) DeductBalance(ctx context.Context, userID string, amount int64) error {
    return a.svc.DeductBalance(ctx, userID, amount)
}
```

### 4. 조립 (main.go) — 의존성 방향이 명확

```go
// cmd/api/main.go
func main() {
    db := connectDB()

    // 1. 각 도메인 초기화 (독립적)
    userRepo := userpostgres.NewUserRepository(db)
    userSvc := userapp.NewUserService(userRepo)

    productRepo := productpostgres.NewProductRepository(db)
    productSvc := productapp.NewProductService(productRepo)

    paymentRepo := paymentpostgres.NewPaymentRepository(db)

    // 2. Adapter로 연결 (도메인 간 의존성은 여기서만 발생)
    userAdapterForPayment := adapters.NewUserAdapterForPayment(userSvc)
    productAdapterForPayment := adapters.NewProductAdapterForPayment(productSvc)

    // 3. Payment 서비스 조립
    paymentSvc := paymentapp.NewPaymentService(
        userAdapterForPayment,    // port.UserService
        productAdapterForPayment, // port.ProductService
        paymentRepo,
    )

    // 4. HTTP 핸들러 등록
    mux := http.NewServeMux()
    mux.HandleFunc("POST /payments", handler.NewPaymentHandler(paymentSvc).Create)
    http.ListenAndServe(":8080", mux)
}
```

---

## 도메인 이벤트로 의존성 완전 제거

도메인 간 **순환 의존** 또는 **비동기 처리**가 필요할 때 Domain Event 사용.

```
Order 생성 → OrderCreatedEvent 발행 → Payment가 구독 → 결제 처리
```

```go
// internal/shared/event/events.go
package event

type OrderCreated struct {
    OrderID   string
    UserID    string
    ProductID string
    Amount    int64
}

// internal/order/application/service.go
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    order := domain.NewOrder(req.UserID, req.ProductID, req.Amount)
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }

    // 이벤트 발행 — Payment를 직접 호출하지 않음
    s.eventBus.Publish(ctx, event.OrderCreated{
        OrderID:   order.ID,
        UserID:    req.UserID,
        ProductID: req.ProductID,
        Amount:    req.Amount,
    })
    return nil
}

// internal/payment/infrastructure/event/subscriber.go
func (s *PaymentEventSubscriber) OnOrderCreated(ctx context.Context, e event.OrderCreated) error {
    return s.paymentSvc.ProcessPayment(ctx, ProcessPaymentRequest{
        UserID:    e.UserID,
        ProductID: e.ProductID,
        Amount:    e.Amount,
    })
}
```

**장점**: Order와 Payment가 서로를 import하지 않음 → 완전한 독립성

---

## 마이크로서비스로 분리하는 방법

이 구조의 최대 장점: **도메인 코드는 변경 없이** 인프라만 교체.

```
[현재 — Modular Monolith]
main.go → userSvc + paymentSvc 직접 조립
도메인 간 통신: in-process 함수 호출 (Adapter 통해)

[분리 후 — Microservices]
user-service/cmd/main.go → userSvc만
payment-service/cmd/main.go → paymentSvc만

도메인 간 통신: HTTP/gRPC Adapter로 교체
```

```go
// 기존 Adapter (in-process)
type UserAdapterForPayment struct {
    svc *userapp.UserService  // 같은 프로세스
}

// 분리 후 Adapter (HTTP 호출)
type UserHTTPAdapterForPayment struct {
    client *http.Client
    baseURL string
}

func (a *UserHTTPAdapterForPayment) GetBalance(ctx context.Context, userID string) (int64, error) {
    // user-service HTTP API 호출
    resp, err := a.client.Get(a.baseURL + "/users/" + userID + "/balance")
    // ... 파싱
}
```

**Port(인터페이스)는 그대로 — Adapter 구현체만 교체.**
Payment Domain 코드는 한 줄도 바뀌지 않는다.

---

## 도메인 간 트랜잭션 처리

Modular Monolith에서 같은 DB를 쓰면 ACID 트랜잭션 가능.
마이크로서비스 분리 후에는 **Saga 패턴** 또는 **Outbox 패턴** 필요.

```go
// 모놀리스: 하나의 DB 트랜잭션으로 묶기
func (s *PaymentService) ProcessPayment(ctx context.Context, req ...) error {
    tx, err := s.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    if err := s.userSvc.DeductBalanceWithTx(ctx, tx, req.UserID, req.Amount); err != nil {
        return err
    }
    if err := s.repo.SaveWithTx(ctx, tx, payment); err != nil {
        return err
    }

    return tx.Commit()
}
```

> 마이크로서비스 전환 시 이 부분이 가장 어렵다.
> 초기에는 Modular Monolith + 단일 DB로 시작해서 검증 후 분리하는 것이 현실적.

---

## Clean Architecture vs DDD Modular Monolith vs Hexagonal

| | Clean Architecture | Hexagonal | DDD Modular Monolith |
|---|---|---|---|
| **경계 단위** | 레이어 (UseCase, Delivery...) | Core vs Adapter | Bounded Context (도메인별) |
| **도메인 간 통신** | 명시적 설계 없음 | Port로 분리 가능 | Port/Adapter + 도메인 이벤트 |
| **마이크로서비스 분리** | 어려움 | 가능 | **설계 목표부터 포함** |
| **Go 적합성** | ★★★★☆ | ★★★★★ | ★★★★★ |
| **복잡도** | 낮음 | 중간 | 높음 |
| **팀 규모 적합** | 소~중형 | 중형 | 중~대형 |

---

## 면접 질문

**Q. 도메인 간 의존성을 어떻게 관리했나요?**

**난이도**: 심화

**핵심 키워드**: Bounded Context, Port/Adapter, 의존성 역전, 순환 의존 방지

**모범 답변 방향**:
Modular Monolith에서 도메인 간 의존성을 관리하는 핵심은 도메인 코드끼리 직접 import하지 않는 것입니다. 각 도메인이 다른 도메인에서 필요한 기능을 Port(인터페이스)로 정의하고, 실제 연결은 Adapter 레이어에서 수행합니다. 예를 들어 Payment 도메인이 사용자 잔액이 필요하면, Payment 도메인 내에 `port.UserService` 인터페이스를 정의하고 Adapter가 이를 구현해서 User 도메인 서비스를 호출합니다. Payment가 User 도메인 패키지를 직접 import하지 않기 때문에 두 도메인의 결합도가 낮습니다. 이 구조 덕분에 User 도메인이 내부적으로 바뀌어도 Adapter의 메서드 시그니처만 맞으면 Payment 도메인 코드는 변경이 없습니다.

비동기 통신이 필요한 경우에는 도메인 이벤트를 사용합니다. "주문이 생성됐다"는 이벤트를 발행하면 알림·결제·재고 도메인이 각각 구독하기 때문에 순환 의존이 생기지 않습니다. Order 도메인이 Notification, Payment, Inventory 각각을 알 필요가 없어서 Order 도메인의 책임 범위가 명확하게 유지됩니다. 이 구조의 가장 큰 장점은 나중에 결제 도메인을 마이크로서비스로 분리할 때, 도메인 코드는 변경하지 않고 in-process Adapter만 HTTP/gRPC Adapter로 교체하면 된다는 점입니다. 카테노이드에서 채팅 서버를 개발할 때도 ZooKeeper 의존성을 Port로 추상화해서 도메인 코드가 ZooKeeper API에 직접 의존하지 않도록 했고, 덕분에 테스트 환경에서는 In-Memory Adapter로, 운영 환경에서는 실제 ZooKeeper 연결 Adapter로 교체해 사용했습니다. 이처럼 도메인 간 의존성을 Port로 관리하면 각 도메인을 독립적으로 테스트할 수 있고, Mock Adapter를 주입해 특정 시나리오를 재현하기도 쉽습니다. 의존성이 명시적인 인터페이스로만 드러나기 때문에 도메인 경계를 넘는 커플링을 쉽게 감지할 수 있고, 새 팀원이 합류했을 때 도메인 경계와 의존 관계를 빠르게 파악할 수 있습니다.

**꼬리 질문**:
- 도메인 간 트랜잭션은 어떻게 처리했나요? (같은 DB면 하나의 TX, 분리 시 Saga 패턴)
- 순환 의존성이 발생하면 어떻게 풀었나요? (도메인 이벤트, 공통 도메인 추출)

---

**Q. Bounded Context를 어떻게 식별하나요?**

**난이도**: 심화

**핵심 키워드**: 업무 영역, 언어(유비쿼터스 언어), 데이터 소유권, 팀 경계

**모범 답변 방향**:
Bounded Context를 식별하는 가장 직관적인 신호는 같은 개념을 다르게 부르는 지점입니다. "User"가 쇼핑몰 도메인에서는 Member이고, 결제 도메인에서는 Payer로 불린다면 그 지점이 경계입니다. 같은 언어(유비쿼터스 언어)를 공유하는 범위가 하나의 Context입니다. 이 언어 불일치를 발견하면 억지로 하나의 User 엔티티로 통합하려 하지 말고, 각 도메인이 자기 관점의 모델을 독립적으로 가져야 합니다.

두 번째 기준은 데이터 소유권입니다. "이 데이터를 누가 소유하고 변경하는가"가 명확하면 Context 경계도 명확합니다. 결제 정보는 Payment 도메인이 소유하고 Order 도메인은 읽기만 한다면, 이 둘은 다른 Context입니다. 소유권이 불명확하면 여러 도메인이 같은 데이터를 동시에 변경하면서 충돌이 생기고 결합도가 높아집니다. 세 번째는 변경 빈도입니다. 함께 자주 바뀌는 것들은 같은 Context에 두는 것이 자연스럽고, 독립적으로 바뀌는 것들은 분리하면 됩니다. 결제 로직이 바뀌어도 사용자 인증 로직은 변경이 없다면 두 기능은 다른 Context에 있어야 합니다. Conway의 법칙도 중요한데, 팀 구조가 Context 경계를 자연스럽게 만듭니다. 한 팀이 담당하는 범위가 Context의 실제 경계가 되는 경우가 많습니다. 반대로 Context 경계와 팀 경계가 불일치하면 팀 간 조율 비용이 높아집니다. Context를 너무 잘게 나누면 오히려 도메인 간 통신 코드가 늘어나고, 너무 크게 잡으면 경계 내부가 복잡해져서 장점이 사라집니다. 실무에서는 큰 Context로 시작해서 서로 다른 이유로 변경되는 부분이 생기면 그때 분리하는 방식이 현실적입니다. 처음부터 경계를 너무 잘게 나누면 통합 비용이 높아지고, 이후에 경계가 잘못됐다는 것을 알게 될 때 재구성하는 비용이 매우 큽니다. 반면 처음에 하나의 큰 모듈로 시작하면 도메인 언어와 경계가 명확해질 때 분리하는 방식을 택할 수 있어 더 유연합니다. 단, 분리할 때를 놓치면 강한 결합이 굳어지므로 정기적으로 도메인 경계를 검토하는 것이 중요합니다.

**꼬리 질문**:
- Bounded Context가 너무 작거나 크다는 것을 어떻게 판단하나요?
- User와 Order가 같은 Context여야 하는 경우도 있나요?

---

**Q. Modular Monolith에서 마이크로서비스로 분리할 때 어떤 과정을 거치나요?**

**난이도**: 심화

**핵심 키워드**: Strangler Fig, 점진적 분리, Adapter 교체, DB 분리

**모범 답변 방향**:
Modular Monolith에서 마이크로서비스로 분리할 때 한 번에 전부 분리하지 않고 점진적으로 진행합니다. 이 접근 방식을 Strangler Fig 패턴이라고도 하는데, 기존 시스템을 유지하면서 새 서비스를 조금씩 키워가는 방식입니다.

첫 번째 전제조건은 Port/Adapter 패턴으로 도메인 간 직접 의존이 이미 제거된 상태여야 한다는 것입니다. 도메인끼리 패키지를 직접 import하고 있다면 분리 자체가 불가능합니다. 두 번째로 도메인별로 스키마를 논리적으로 분리합니다. 같은 DB를 쓰더라도 Payment 도메인은 payment 접두사가 붙은 테이블만 접근하도록 제한해서 나중에 물리적으로 분리할 준비를 합니다. 세 번째로 분리할 도메인 코드를 별도 서비스 레포지토리로 추출하고, 네 번째로 in-process Adapter를 HTTP/gRPC Adapter로 교체합니다. Port(인터페이스)는 그대로이고 Adapter 구현체만 바뀌기 때문에 도메인 코드 자체는 변경이 없습니다. 샵라이브에서 DB 마이그레이션 과정에서도 비슷한 원리를 적용했습니다. 기존 Repository 구현체를 유지하면서 새 DB를 바라보는 Adapter를 추가해 점진적으로 전환했습니다. 가장 복잡한 마지막 단계는 DB 인스턴스를 물리적으로 분리하는 것인데, 데이터 마이그레이션과 일관성 보장이 필요해서 Saga 패턴이나 Outbox 패턴을 함께 사용합니다. Outbox 패턴은 도메인 이벤트를 같은 DB 트랜잭션으로 저장하고 별도 프로세스가 읽어서 발행하는 방식으로 at-least-once 보장을 제공합니다. 이렇게 단계를 나눠서 점진적으로 분리하면 서비스 전체를 한꺼번에 마이크로서비스로 전환하는 빅뱅 방식보다 리스크가 훨씬 낮습니다. 각 단계에서 운영 중에 동작을 검증하면서 진행할 수 있기 때문입니다. 실무에서 빅뱅 방식으로 전환하다 서비스 장애가 발생한 사례가 많기 때문에, 점진적 분리가 선호됩니다. 분리하기 전에 도메인 코드가 Port/Adapter 패턴으로 이미 격리되어 있는지 반드시 확인해야 하며, 격리되어 있지 않다면 분리보다 먼저 리팩토링에 시간을 투자해야 합니다. 이 사전 준비가 실제 분리 작업의 복잡도를 결정하는 가장 중요한 요소입니다.

**꼬리 질문**:
- DB를 분리할 때 데이터 일관성은 어떻게 보장하나요? (Saga 패턴, Outbox 패턴)

---

**Q. 도메인 이벤트와 Port/Adapter 중 어떤 것을 언제 사용하나요?**

**난이도**: 심화

**핵심 키워드**: 동기 vs 비동기, 결합도, 데이터 일관성, 순환 의존

**모범 답변 방향**:
Port/Adapter(동기)와 도메인 이벤트(비동기) 선택 기준은 "A가 B의 결과를 즉시 사용해야 하는가"입니다. 결제 시 잔액 확인처럼 A가 B의 응답을 받아야 다음 단계로 진행할 수 있다면 Port/Adapter 동기 호출이 맞습니다. 잔액이 부족하면 결제를 거부해야 하기 때문에 이 결과는 즉시 필요합니다. 동기 호출에서 실패하면 에러를 즉시 반환할 수 있어서 처리 흐름이 단순합니다.

반면 주문이 생성됐을 때 알림 발송, 포인트 적립, 재고 차감처럼 여러 도메인이 같은 이벤트에 반응해야 한다면 도메인 이벤트가 적합합니다. 이벤트 방식은 Order 도메인이 Notification, Point, Inventory 각각을 알 필요 없이 "OrderCreated" 이벤트만 발행하면 되어 순환 의존을 원천 차단합니다. 새로운 구독자가 추가되어도 Order 도메인 코드는 변경이 없습니다. 카테노이드 채팅 서버에서도 메시지 저장 후 알림 발송, 읽음 처리 이벤트 등 부가 기능은 비동기 이벤트로 분리해서 핵심 메시지 저장 흐름의 응답 시간에 영향을 주지 않게 했습니다. 실무 원칙은 핵심 흐름은 동기(Port/Adapter)로, 부가 효과는 비동기(도메인 이벤트)로 분리하는 것입니다. 핵심 흐름에 비동기를 쓰면 실패 처리와 일관성 보장이 복잡해집니다. 이벤트가 유실되면 재처리 메커니즘이 필요하고, Outbox 패턴이나 Kafka 같은 메시지 브로커를 도입해야 하므로 운영 복잡도가 크게 높아집니다. 따라서 비즈니스적으로 즉각적인 일관성이 필요한 흐름에는 동기를, 최종적 일관성으로 충분한 부가 기능에는 비동기를 적용합니다. 이 원칙을 지키면 비동기 처리의 복잡도를 부가 기능에만 한정할 수 있어서, 핵심 흐름의 코드를 단순하고 예측 가능하게 유지할 수 있습니다. 카테노이드에서 메시지 전송 흐름의 핵심 경로는 동기로 유지하고, 알림 발송과 통계 집계는 비동기 이벤트로 분리해서 응답 시간 목표를 안정적으로 달성했습니다. 이 원칙을 미리 정하지 않으면 팀마다 기준 없이 동기와 비동기를 혼용하게 되어 시스템 전체의 일관성 보장 수준을 파악하기 어려워집니다. 아키텍처 초기에 어느 흐름이 즉각적 일관성을 필요로 하는지 명확히 정의해두는 것이 중요합니다.

---

## 참고 링크
- [Developing Modular Monolith and Hexagonal Architecture in Go](https://notes.softwarearchitect.id/p/developing-modular-monolith-and-hexagonal)
- [Three Dots Labs — Microservices or Monolith: It's a Detail](https://threedots.tech/post/microservices-or-monolith-its-detail/)
- [GO Modular Monolith: Part I & II](https://medium.com/@arkjuniork.yudistira/go-modular-monolith-part-i-f963da742e81)
- [Domain-Driven Design with GoLang (Packt)](https://github.com/PacktPublishing/Domain-Driven-Design-with-GoLang)
- [Bounded Context 개념과 실무](https://leejaedoo.github.io/bounded_context/)
