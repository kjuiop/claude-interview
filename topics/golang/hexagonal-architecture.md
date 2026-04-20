---
tags: [golang, go, hexagonal-architecture, ports-and-adapters, architecture]
related: [clean-architecture, interface, system-design, ddd-modular-monolith]
---

# Golang — Hexagonal Architecture (Ports & Adapters)

→ [[home]] | [[topics/golang/clean-architecture]] | [[topics/golang/interface]] | [[topics/system-design/concepts]]

---

## 왜 Go에서 Hexagonal이 더 자연스러운가

Go의 **암시적 인터페이스(Implicit Interface)**가 Hexagonal Architecture의 Port 개념과 자연스럽게 맞아떨어진다.

```go
// Port 정의 (core 안에 있음)
type UserRepository interface {
    Save(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id string) (*User, error)
}

// Adapter 구현 (infrastructure에 있음)
// — implements 선언 없이 메서드만 맞으면 자동으로 Port를 만족
type PostgresUserRepo struct { db *sql.DB }

func (r *PostgresUserRepo) Save(ctx context.Context, user *User) error { ... }
func (r *PostgresUserRepo) FindByID(ctx context.Context, id string) (*User, error) { ... }
```

**Go + Hexagonal이 잘 맞는 이유:**

1. **암시적 인터페이스** → Adapter가 Port를 `implements` 없이 자동 충족. 결합도 자연스럽게 낮아짐
2. **작은 인터페이스 선호** → Go 철학과 Hexagonal의 "딱 필요한 Port만 정의" 원칙이 일치
3. **패키지 단위 캡슐화** → `internal/core` 패키지가 외부 의존을 완벽하게 차단
4. **Clean Architecture의 4~5 레이어 경계**가 Go의 "단순함" 철학과 약간 충돌하는 반면, Hexagonal은 Core / Adapter 2가지만 구분해 더 직관적

참고: [[topics/golang/interface]]

---

## Ports & Adapters 핵심 개념

```
  Primary Adapters           Application Core              Secondary Adapters
  (Driving side — 외부가 Core 호출)         (Driven side — Core가 외부 호출)

  HTTP   ──→ [Primary Port]  ──→ [Domain]    ──→ [Secondary Port] ──→ Postgres
  gRPC   ──→ (UseCase 인터페이스)  [Service]    (Repository 인터페이스)  ──→ Redis
  CLI    ──→                              ──→                      ──→ Kafka
  Test   ──→                              ──→ [Mock Adapter]       (테스트용)
```

| 용어 | 의미 | Go에서 |
|---|---|---|
| **Port** | 경계면(인터페이스). Core가 외부와 소통하는 계약 | `interface` |
| **Primary Port** | 외부 → Core 방향. UseCase 인터페이스 | `type UserService interface { CreateUser(...) }` |
| **Secondary Port** | Core → 외부 방향. Repository/외부서비스 인터페이스 | `type UserRepository interface { Save(...) }` |
| **Primary Adapter** | Primary Port의 구현. 외부 요청을 Core 형식으로 변환 | HTTP Handler, gRPC Server |
| **Secondary Adapter** | Secondary Port의 구현. Core 요청을 외부 시스템 형식으로 변환 | Postgres Repo, Redis Cache |

---

## Clean Architecture vs Hexagonal Architecture 상세 비교

### 구조 비교

**Clean Architecture:**
```
Delivery → UseCase → Repository → Domain
(명확한 4개 레이어, 안쪽으로만 의존)
```

**Hexagonal Architecture:**
```
Primary Adapter → [Primary Port] → Core ← [Secondary Port] ← Secondary Adapter
(Core가 중심, 좌우로 Adapter가 붙음)
```

### 항목별 비교

| 항목 | Clean Architecture | Hexagonal Architecture |
|---|---|---|
| **레이어 수** | 4~5개 (Domain, Repository, UseCase, Delivery) | 2개 (Core, Adapter) |
| **Port 위치** | Repository 인터페이스가 Domain에, UseCase 인터페이스는 암묵적 | `core/port/in/`, `core/port/out/` 으로 명시적 분리 |
| **의존성 방향** | 안쪽 레이어로만 | 모든 방향이 Core를 향함 |
| **Go 적합성** | ★★★★☆ | ★★★★★ |
| **구조 명확성** | 레이어 경계가 명확 | Port 방향(in/out)이 명확 |
| **테스트 용이성** | Secondary Port Mock으로 UseCase 테스트 | 동일. Primary Port Mock으로 Adapter도 독립 테스트 가능 |
| **복잡도** | 레이어 간 Mapping 코드 증가 가능 | Core/Adapter 2분할로 단순 |
| **실무 선택** | 팀이 Clean Architecture에 익숙할 때 | 프레임워크 독립성 강조할 때 |

### 핵심 차이: Primary Port의 유무

Clean Architecture에서 UseCase는 구체 클래스로 만들고 Delivery(Handler)가 직접 호출하는 경우가 많다.
Hexagonal에서는 UseCase도 **Primary Port(인터페이스)**로 정의 → Adapter가 인터페이스에 의존.

```go
// Clean Architecture 방식 (UseCase를 직접 참조)
type UserHandler struct {
    createUser *usecase.CreateUserUseCase  // 구체 타입
}

// Hexagonal 방식 (Primary Port 인터페이스에 의존)
type UserHandler struct {
    userService port.UserService  // 인터페이스 (Primary Port)
}
```

→ Hexagonal이 Primary Adapter(HTTP)도 Core를 몰라도 되게 만든다는 점에서 한 단계 더 분리됨.

---

## Go 프로젝트 구조 (Hexagonal)

```
project/
├── cmd/
│   ├── api/
│   │   └── main.go              # 조립(Wiring) + 서버 실행
│   └── worker/
│       └── main.go
│
├── internal/
│   ├── core/                    # Application Core — 프레임워크·DB를 모름
│   │   ├── domain/
│   │   │   ├── user.go          # User 엔티티, 비즈니스 규칙, 도메인 에러
│   │   │   └── product.go
│   │   │
│   │   ├── port/
│   │   │   ├── in/              # Primary Ports: 외부 → Core
│   │   │   │   ├── user_service.go    # type UserService interface { ... }
│   │   │   │   └── product_service.go
│   │   │   └── out/             # Secondary Ports: Core → 외부
│   │   │       ├── user_repository.go # type UserRepository interface { ... }
│   │   │       └── cache.go           # type Cache interface { ... }
│   │   │
│   │   └── service/             # Primary Port 구현 (순수 비즈니스 로직)
│   │       ├── user_service.go
│   │       └── product_service.go
│   │
│   └── adapter/
│       ├── in/                  # Primary Adapters (Driving): 외부 → Core 호출
│       │   ├── http/
│       │   │   ├── handler/
│       │   │   │   └── user_handler.go  # HTTP 요청 → port.UserService 호출
│       │   │   ├── middleware/
│       │   │   │   ├── auth.go
│       │   │   │   └── logging.go
│       │   │   └── router.go
│       │   └── grpc/
│       │       └── user_grpc.go
│       │
│       └── out/                 # Secondary Adapters (Driven): Core → 외부 호출
│           ├── postgres/
│           │   └── user_repo.go   # port.UserRepository 구현
│           ├── redis/
│           │   └── cache.go       # port.Cache 구현
│           └── kafka/
│               └── event_publisher.go
│
├── pkg/                         # 외부 공개 유틸리티
├── config/
├── migrations/
└── go.mod
```

**디렉터리 결정 원칙:**
- `core/` — 이 폴더는 어떤 외부 패키지도 import하면 안 됨 (`go.mod`의 라이브러리 제외)
- `core/port/in/` — HTTP, gRPC가 뭔지 모름. UseCase 인터페이스만 정의
- `core/port/out/` — DB가 뭔지 모름. 필요한 데이터 기능만 인터페이스로 표현
- `adapter/` — Core를 import 가능. 외부 라이브러리도 import 가능

---

## 핵심 코드 패턴

### 1. Domain 엔티티

```go
// internal/core/domain/user.go
package domain

import "errors"

var (
    ErrUserNotFound    = errors.New("user not found")
    ErrEmailExists     = errors.New("email already exists")
    ErrInvalidEmail    = errors.New("invalid email")
)

type User struct {
    ID    string
    Email string
    Name  string
}

func NewUser(name, email string) (*User, error) {
    if email == "" {
        return nil, ErrInvalidEmail
    }
    return &User{ID: generateID(), Name: name, Email: email}, nil
}
```

### 2. Primary Port (UseCase 인터페이스)

```go
// internal/core/port/in/user_service.go
package in

import "context"

type CreateUserRequest struct {
    Name  string
    Email string
}

// Primary Port — HTTP Handler, gRPC Server 등이 이 인터페이스에 의존
type UserService interface {
    CreateUser(ctx context.Context, req CreateUserRequest) error
    GetUser(ctx context.Context, id string) (*UserResponse, error)
}
```

### 3. Secondary Port (Repository 인터페이스)

```go
// internal/core/port/out/user_repository.go
package out

import (
    "context"
    "project/internal/core/domain"
)

// Secondary Port — Service 구현체가 이 인터페이스에 의존
type UserRepository interface {
    Save(ctx context.Context, user *domain.User) error
    FindByID(ctx context.Context, id string) (*domain.User, error)
    ExistsByEmail(ctx context.Context, email string) (bool, error)
}
```

### 4. Service (Core 비즈니스 로직 + Primary Port 구현)

```go
// internal/core/service/user_service.go
package service

type UserService struct {
    userRepo out.UserRepository  // Secondary Port에 의존 (인터페이스)
    cache    out.Cache
}

// in.UserService (Primary Port) 를 구현
func (s *UserService) CreateUser(ctx context.Context, req in.CreateUserRequest) error {
    exists, err := s.userRepo.ExistsByEmail(ctx, req.Email)
    if err != nil {
        return fmt.Errorf("check email: %w", err)
    }
    if exists {
        return domain.ErrEmailExists
    }

    user, err := domain.NewUser(req.Name, req.Email)
    if err != nil {
        return err
    }

    return s.userRepo.Save(ctx, user)
}
```

### 5. Primary Adapter (HTTP Handler)

```go
// internal/adapter/in/http/handler/user_handler.go
package handler

type UserHandler struct {
    userService in.UserService  // Primary Port 인터페이스에 의존 (구체 타입 모름)
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req in.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }

    if err := h.userService.CreateUser(r.Context(), req); err != nil {
        switch {
        case errors.Is(err, domain.ErrEmailExists):
            http.Error(w, err.Error(), http.StatusConflict)
        default:
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
        return
    }
    w.WriteHeader(http.StatusCreated)
}
```

### 6. Secondary Adapter (Postgres Repository)

```go
// internal/adapter/out/postgres/user_repo.go
package postgres

type UserPostgresRepo struct { db *sql.DB }

// out.UserRepository (Secondary Port) 를 구현
func (r *UserPostgresRepo) Save(ctx context.Context, user *domain.User) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO users (id, email, name) VALUES ($1, $2, $3)",
        user.ID, user.Email, user.Name,
    )
    return err
}

func (r *UserPostgresRepo) ExistsByEmail(ctx context.Context, email string) (bool, error) {
    var count int
    err := r.db.QueryRowContext(ctx,
        "SELECT COUNT(1) FROM users WHERE email = $1", email,
    ).Scan(&count)
    return count > 0, err
}
```

### 7. 조립 (main.go)

```go
// cmd/api/main.go — Secondary Adapter → Service → Primary Adapter 순서로 조립
func main() {
    db := connectDB()
    redisClient := connectRedis()

    // Secondary Adapters (out)
    userRepo := postgres.NewUserPostgresRepo(db)
    cache := redis.NewRedisCache(redisClient)

    // Core Service (Primary Port 구현체)
    userSvc := service.NewUserService(userRepo, cache)

    // Primary Adapters (in)
    userHandler := handler.NewUserHandler(userSvc)

    mux := http.NewServeMux()
    mux.HandleFunc("POST /users", userHandler.CreateUser)
    http.ListenAndServe(":8080", mux)
}
```

---

## 테스트 전략

Hexagonal의 가장 큰 장점: **Core를 완전히 격리한 단위 테스트**

```go
// Core Service 단위 테스트 — DB, HTTP 없이
func TestCreateUser_EmailExists(t *testing.T) {
    mockRepo := &MockUserRepository{
        ExistsByEmailFn: func(ctx context.Context, email string) (bool, error) {
            return true, nil  // 이미 존재하는 이메일 시뮬레이션
        },
    }

    svc := service.NewUserService(mockRepo, nil)
    err := svc.CreateUser(context.Background(), in.CreateUserRequest{
        Email: "test@test.com", Name: "Test",
    })

    assert.ErrorIs(t, err, domain.ErrEmailExists)
}
```

- `core/` 테스트: Mock Secondary Adapter 주입 → DB 없이 비즈니스 로직만 검증
- `adapter/in/` 테스트: Mock Primary Port(Service) 주입 → HTTP 요청/응답 형식만 검증
- `adapter/out/` 테스트: 실제 DB 연결 (통합 테스트)

참고: [[topics/golang/goroutine]], [[topics/golang/context]]

---

## 실무 트레이드오프

### Hexagonal 선택 기준

**적합한 경우:**
- 여러 프로토콜 지원 (HTTP + gRPC + CLI + 테스트)
- DB 교체 가능성이 있거나 멀티 DB (Postgres + Redis)
- 엄격한 Core 격리 테스트가 중요한 경우
- 팀이 아키텍처 패턴에 익숙하고 장기 유지보수 계획이 있는 경우

**Clean Architecture 선택 기준:**
- 팀이 이미 Clean Architecture에 익숙한 경우
- 레이어 경계가 명확한 편이 팀 커뮤니케이션에 유리한 경우

**둘 다 피해야 할 경우:**
- 팀 5명 이하, 기능이 단순한 CRUD 서비스 → 2~3 레이어로 단순화

---

## 면접 질문

**Q. Hexagonal Architecture에서 Port와 Adapter의 역할을 설명해주세요.**

**난이도**: 중급

**핵심 키워드**: Port(인터페이스), Primary/Secondary, Driving/Driven, 의존성 역전

**모범 답변 방향**:
Hexagonal Architecture에서 Port는 Core의 경계면, 즉 Core가 외부와 맺는 계약입니다. Go에서는 인터페이스로 표현됩니다. Port는 방향에 따라 두 종류로 나뉩니다. Primary Port는 외부(HTTP Handler, CLI, gRPC Server)가 Core를 호출하는 창구로, UseCase 인터페이스가 여기에 해당합니다. Secondary Port는 반대로 Core가 외부 시스템(DB, Cache, Kafka)을 호출할 때 사용하는 창구로, Repository나 EventPublisher 인터페이스가 여기에 해당합니다. Adapter는 이 Port를 실제 기술로 구현한 것입니다. HTTP Handler가 Primary Adapter, Postgres Repository 구현체가 Secondary Adapter입니다. 핵심은 Core가 Port 인터페이스에만 의존하기 때문에, Adapter를 교체해도 Core 코드는 변경할 필요가 없다는 점입니다. 의존성 역전 원칙(DIP)에 의해 모든 의존성은 항상 Core를 향합니다. Adapter가 Core를 알지만, Core는 Adapter를 전혀 모릅니다.

**꼬리 질문**:
- Primary Adapter와 Secondary Adapter의 차이는? (방향 — 누가 누구를 호출하느냐)
- Test를 위한 Mock도 Adapter인가요? (네. Mock Secondary Adapter를 주입해 Core를 격리 테스트)

**면접 세션 피드백 (2026-03-30)**:
- Port=추상화/Adapter=구현체 구분, 결합도 제거 목적은 정확히 이해
- Primary/Secondary 방향 구분 미숙 — 구분 기준은 **방향(누가 Core를 호출/구현하느냐)**. 레이어 위치가 아님
- HTTP Handler = Primary Adapter (Primary Port를 호출하는 구현체)
- Go 암시적 인터페이스 연결 미언급 — `implements` 없이 메서드 시그니처만 맞으면 Port 자동 충족이 핵심
- 컴파일 타임 검증: `var _ port.UserRepository = (*PostgresUserRepo)(nil)` 패턴 암기

**면접 세션 피드백 (2026-04-12 1회차)**:
- 잘한 점: Port가 도메인에 선언되는 구조, Go 암시적 인터페이스 특성, In-Memory Adapter로 DB 없이 테스트 이점, 외부 의존성 교체 용이성 모두 언급. 의존성 방향 핵심 파악.
- 보완:
  - "외부가 내부를 모른다" 오류: **Adapter(외부)는 Port(내부 경계)를 알아야 함**. 정확한 표현: "의존성이 항상 바깥→안쪽(Adapter → Port ← Domain). Domain은 Adapter를 모른다."
  - In-Memory Adapter 개념 오류: "expected value 반환" = Mock처럼 들림. In-Memory Adapter는 `map[int64]User`에 **실제로 저장/읽는 stateful 구현체**. 고정값 반환이 아님.
  - DIP 키워드 미언급: "Dependency Inversion Principle — 의존성이 항상 고수준 정책(도메인)을 향한다" 함께 언급해야 면접관 인상에 남음.
  - 코드 패턴 미언급: interface 선언 → struct 구현 → 메서드 구현 흐름을 말로라도 설명 필요

---

**Q. Go의 암시적 인터페이스가 Hexagonal Architecture와 왜 잘 맞나요?**

**난이도**: 심화

**핵심 키워드**: 암시적 구현, Duck typing, 결합도, Port 자동 충족

**모범 답변 방향**:
Go의 암시적 인터페이스는 `implements` 선언 없이 메서드 시그니처만 일치하면 인터페이스를 자동으로 구현하는 Duck typing 방식입니다. 이게 Hexagonal Architecture와 잘 맞는 이유는, Hexagonal에서 Secondary Adapter(예: Postgres 구현체)가 Core에 정의된 Port(예: UserRepository 인터페이스)를 구현할 때 별도의 등록이나 선언이 필요 없기 때문입니다. 패키지가 달라도 메서드 시그니처만 맞으면 자동으로 Port를 충족합니다. Java에서는 `implements UserRepository`처럼 명시해야 해서 Adapter 패키지가 Port 패키지를 import해야 하는 순환 의존 위험이 생깁니다. Go에서는 이런 문제가 없어 Core와 Adapter 사이의 결합도가 자연스럽게 낮아집니다. 컴파일 타임에 인터페이스 충족 여부를 확인하고 싶을 때는 `var _ port.UserRepository = (*PostgresUserRepo)(nil)` 패턴을 사용합니다. 이 선언이 컴파일되면 PostgresUserRepo가 UserRepository를 완전히 구현했다는 보장이 됩니다.

**꼬리 질문**:
- 인터페이스 준수 여부를 컴파일 타임에 확인하고 싶다면? (`var _ port.UserRepository = (*PostgresUserRepo)(nil)`)

---

**Q. Clean Architecture와 Hexagonal Architecture 중 어떤 것을 선택하고 왜인가요?**

**난이도**: 심화

**핵심 키워드**: 트레이드오프, 팀 친숙도, 프로토콜 다양성, 복잡도

**모범 답변 방향**:
두 패턴 중 정답은 없고 팀 상황에 따라 다릅니다. Hexagonal이 유리한 경우는 HTTP, gRPC, CLI처럼 여러 프로토콜을 동시에 지원해야 하거나, Core 격리 테스트가 중요한 경우, 또는 DB나 외부 서비스를 교체할 가능성이 있는 장기 유지보수 프로젝트입니다. Clean Architecture가 적합한 경우는 팀이 이미 Clean Architecture에 익숙해서 온보딩 비용이 낮거나, 레이어 경계를 명확히 나누는 방식이 팀 커뮤니케이션에 더 잘 맞을 때입니다. 실무에서는 두 패턴을 혼용하는 경우도 많습니다. Clean Architecture의 레이어 이름(Domain, UseCase, Repository)을 유지하면서 Hexagonal의 Port/Adapter 개념을 적용해 인터페이스를 명시적으로 구분하는 방식입니다. 반대로 소규모 서비스나 팀 5명 이하의 단순 CRUD 서비스라면 두 패턴 모두 오버엔지니어링이 될 수 있어서 2~3 레이어 단순 분리로 충분합니다.

**꼬리 질문**:
- 실제로 어떤 패턴을 써봤나요? (본인 경험 기반 답변)

---

## 참고 링크
- [Hexagonal Architecture (Alistair Cockburn 원문)](https://alistair.cockburn.us/hexagonal-architecture/)
- [Go + Hexagonal - Three Dots Labs](https://threedots.tech/post/introducing-clean-architecture/)
- [Hexagonal Architecture in Go - DEV Community](https://dev.to/bagashiz/building-restful-api-with-hexagonal-architecture-in-go-1mij)
- [Ports & Adapters Pattern](https://www.dossier-andreas.net/software_architecture/ports_and_adapters.html)
