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
- 각 도메인이 다른 도메인에서 필요한 기능을 Port(인터페이스)로 정의
- 실제 연결은 Adapter 레이어에서 — 도메인 코드끼리 직접 import 없음
- 도메인 이벤트로 비동기 통신 처리 (순환 의존 방지)
- 이 구조 덕분에 결제 도메인을 마이크로서비스로 분리할 때 도메인 코드 변경 없이 Adapter만 HTTP로 교체

**꼬리 질문**:
- 도메인 간 트랜잭션은 어떻게 처리했나요? (같은 DB면 하나의 TX, 분리 시 Saga 패턴)
- 순환 의존성이 발생하면 어떻게 풀었나요? (도메인 이벤트, 공통 도메인 추출)

---

**Q. Bounded Context를 어떻게 식별하나요?**

**난이도**: 심화

**핵심 키워드**: 업무 영역, 언어(유비쿼터스 언어), 데이터 소유권, 팀 경계

**모범 답변 방향**:
- 같은 개념을 다르게 부르는 지점이 경계: User가 어떤 도메인에서는 Member, 어떤 도메인에서는 Payer
- 데이터 소유권: 이 데이터를 누가 소유하고 변경하는가
- 변경 빈도: 함께 자주 바뀌는 것들은 같은 Context, 따로 바뀌는 건 다른 Context
- 팀 경계: Conway의 법칙 — 팀 구조가 Context 경계를 자연스럽게 만든다

**꼬리 질문**:
- Bounded Context가 너무 작거나 크다는 것을 어떻게 판단하나요?
- User와 Order가 같은 Context여야 하는 경우도 있나요?

---

**Q. Modular Monolith에서 마이크로서비스로 분리할 때 어떤 과정을 거치나요?**

**난이도**: 심화

**핵심 키워드**: Strangler Fig, 점진적 분리, Adapter 교체, DB 분리

**모범 답변 방향**:
1. Port/Adapter 패턴으로 도메인 간 결합 제거 (이미 완료된 상태)
2. 도메인별로 스키마 분리 (같은 DB지만 Payment는 payment_schema만 접근)
3. 분리할 도메인을 별도 서비스로 추출 (domain 코드 이동)
4. in-process Adapter → HTTP/gRPC Adapter로 교체
5. DB 인스턴스 분리 (가장 복잡한 단계 — 데이터 마이그레이션)

**꼬리 질문**:
- DB를 분리할 때 데이터 일관성은 어떻게 보장하나요? (Saga 패턴, Outbox 패턴)

---

**Q. 도메인 이벤트와 Port/Adapter 중 어떤 것을 언제 사용하나요?**

**난이도**: 심화

**핵심 키워드**: 동기 vs 비동기, 결합도, 데이터 일관성, 순환 의존

**모범 답변 방향**:
- **Port/Adapter (동기)**: 즉각적인 응답이 필요할 때, A가 B의 결과를 바로 사용해야 할 때 (결제 시 잔액 확인)
- **도메인 이벤트 (비동기)**: 결과를 기다릴 필요 없을 때, 순환 의존이 발생할 때, 여러 도메인이 같은 이벤트에 반응해야 할 때 (주문 생성 → 알림, 결제, 재고 차감이 모두 반응)
- 실무: 핵심 흐름은 동기(Port), 부가 효과는 비동기(이벤트)로 분리

---

## 참고 링크
- [Developing Modular Monolith and Hexagonal Architecture in Go](https://notes.softwarearchitect.id/p/developing-modular-monolith-and-hexagonal)
- [Three Dots Labs — Microservices or Monolith: It's a Detail](https://threedots.tech/post/microservices-or-monolith-its-detail/)
- [GO Modular Monolith: Part I & II](https://medium.com/@arkjuniork.yudistira/go-modular-monolith-part-i-f963da742e81)
- [Domain-Driven Design with GoLang (Packt)](https://github.com/PacktPublishing/Domain-Driven-Design-with-GoLang)
- [Bounded Context 개념과 실무](https://leejaedoo.github.io/bounded_context/)
