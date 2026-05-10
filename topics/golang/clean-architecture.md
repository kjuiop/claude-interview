---
tags: [golang, go, clean-architecture, architecture, hexagonal, solid]
related: [interface, error-handling, system-design, ddd-modular-monolith, hexagonal-architecture]
---

# Golang — Clean Architecture

→ [[home]] | [[topics/golang/interface]] | [[topics/golang/error-handling]] | [[topics/system-design/concepts]]

---

## 핵심 개념

Clean Architecture는 소프트웨어를 **동심원 레이어**로 나누고, 레이어 간 **의존성 방향을 엄격히 제한**해 변경 비용을 낮추는 아키텍처 패턴.

- 안쪽 레이어는 바깥쪽을 몰라야 한다 (의존성 역전)
- 비즈니스 로직이 프레임워크·DB에 오염되지 않는다
- 각 레이어를 독립적으로 테스트할 수 있다

---

## 4 레이어 구조

```
         ┌───────────────────────────────┐
         │        Delivery (HTTP/gRPC)    │  ← 가장 바깥
         │  ┌─────────────────────────┐   │
         │  │       UseCase            │   │
         │  │  ┌───────────────────┐  │   │
         │  │  │    Repository      │  │   │
         │  │  │  ┌─────────────┐  │  │   │
         │  │  │  │   Domain    │  │  │   │  ← 가장 안쪽
         │  │  │  └─────────────┘  │  │   │
         │  │  └───────────────────┘  │   │
         │  └─────────────────────────┘   │
         └───────────────────────────────┘
```

| 레이어 | 역할 | 의존 대상 |
|---|---|---|
| **Domain** | 비즈니스 엔티티, 핵심 규칙 | 없음 (가장 독립적) |
| **Repository** | 데이터 접근 인터페이스 + 구현 | Domain |
| **UseCase** | 비즈니스 기능 단위 (조건, 흐름) | Domain, Repository 인터페이스 |
| **Delivery** | HTTP/gRPC 등 외부 인터페이스 | UseCase |

---

## 권장 프로젝트 구조

### Feature-based (2025 권장)

기능별로 파일을 묶는 방식. 응집력이 높고 새 팀원도 빠르게 파악 가능.

```
project/
├── cmd/
│   ├── api/main.go          # API 서버 진입점
│   └── worker/main.go       # 백그라운드 워커
│
├── internal/
│   ├── domain/
│   │   ├── user/
│   │   │   ├── entity.go        # User 엔티티 & 비즈니스 규칙
│   │   │   └── repository.go    # Repository 인터페이스 (!)
│   │   └── product/
│   │       ├── entity.go
│   │       └── repository.go
│   │
│   ├── usecase/
│   │   ├── user/
│   │   │   ├── create_user.go
│   │   │   └── get_user.go
│   │   └── product/
│   │
│   ├── infrastructure/      # Repository 구현체
│   │   ├── postgres/
│   │   │   └── user_repo.go
│   │   └── redis/
│   │       └── cache_repo.go
│   │
│   ├── delivery/
│   │   ├── http/
│   │   │   ├── handler/
│   │   │   │   └── user_handler.go
│   │   │   └── middleware/
│   │   │       ├── auth.go
│   │   │       └── logging.go
│   │   └── grpc/
│   │
│   └── shared/              # 공유 유틸리티
│       ├── errors/
│       └── logger/
│
├── pkg/                     # 외부에서 import 가능
├── config/
├── migrations/
└── go.mod
```

> **핵심 포인트**: Repository **인터페이스**는 `domain/` 에, Repository **구현체**는 `infrastructure/` 에 위치.
> → UseCase는 인터페이스에만 의존 → DB가 바뀌어도 UseCase 코드 변경 없음.

### Type-based (비권장)

```
internal/
├── entity/      # 모든 도메인 엔티티
├── usecase/     # 모든 비즈니스 로직
├── repository/  # 모든 데이터 접근
└── handler/     # 모든 HTTP 핸들러
```

단점: 기능 하나 추가 시 4개 디렉터리 모두 수정. 파일 수가 많아지면 가독성 급락.

---

## 핵심 코드 패턴

### Domain 엔티티 + Repository 인터페이스

```go
// internal/domain/user/entity.go
type User struct {
    ID        string
    Email     string
    Name      string
    CreatedAt time.Time
}

func NewUser(name, email string) (*User, error) {
    if email == "" {
        return nil, ErrInvalidEmail
    }
    return &User{ID: uuid.New().String(), Name: name, Email: email}, nil
}

// internal/domain/user/repository.go
type Repository interface {
    Save(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id string) (*User, error)
    ExistsByEmail(ctx context.Context, email string) (bool, error)
}
```

### UseCase — 비즈니스 흐름

```go
// internal/usecase/user/create_user.go
type CreateUserUseCase struct {
    userRepo domain.Repository  // 인터페이스에 의존 (!)
    logger   Logger
}

func NewCreateUserUseCase(repo domain.Repository, logger Logger) *CreateUserUseCase {
    return &CreateUserUseCase{userRepo: repo, logger: logger}
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, req CreateUserRequest) error {
    exists, err := uc.userRepo.ExistsByEmail(ctx, req.Email)
    if err != nil {
        return fmt.Errorf("check email existence: %w", err)
    }
    if exists {
        return ErrEmailAlreadyExists
    }

    user, err := domain.NewUser(req.Name, req.Email)
    if err != nil {
        return err
    }

    if err := uc.userRepo.Save(ctx, user); err != nil {
        uc.logger.Error("failed to save user", err)
        return fmt.Errorf("save user: %w", err)
    }
    return nil
}
```

### Infrastructure — Repository 구현체

```go
// internal/infrastructure/postgres/user_repo.go
type UserPostgresRepo struct {
    db *sql.DB
}

// domain.Repository 인터페이스를 구현
func (r *UserPostgresRepo) Save(ctx context.Context, user *domain.User) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO users (id, email, name) VALUES ($1, $2, $3)",
        user.ID, user.Email, user.Name,
    )
    return err
}
```

### Delivery — HTTP 핸들러

```go
// internal/delivery/http/handler/user_handler.go
type UserHandler struct {
    createUser *usecase.CreateUserUseCase
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }

    if err := h.createUser.Execute(r.Context(), req); err != nil {
        // 비즈니스 에러 → HTTP 상태코드 변환
        if errors.Is(err, usecase.ErrEmailAlreadyExists) {
            http.Error(w, err.Error(), http.StatusConflict)
            return
        }
        http.Error(w, "internal server error", http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusCreated)
}
```

### 의존성 조립 (main.go)

```go
// cmd/api/main.go
func main() {
    db := connectDB()

    // Infrastructure → UseCase → Delivery 순서로 조립
    userRepo := postgres.NewUserPostgresRepo(db)
    createUserUC := usecase.NewCreateUserUseCase(userRepo, logger)
    userHandler := handler.NewUserHandler(createUserUC)

    mux := http.NewServeMux()
    mux.HandleFunc("POST /users", userHandler.CreateUser)
    http.ListenAndServe(":8080", mux)
}
```

---

## 에러 처리 전략 (레이어별)

```
Domain      → 비즈니스 에러 정의 (ErrNotFound, ErrEmailExists)
Repository  → DB 에러를 도메인 에러로 변환 (sql.ErrNoRows → ErrNotFound)
UseCase     → 에러 wrap & 로깅
Delivery    → 도메인 에러를 HTTP 상태코드로 매핑
```

참고: [[topics/golang/error-handling]]

**면접 세션 피드백 (2026-03-30)**:
- 에러 전파 방향, 사용자 노출 분리 개념은 잘 이해
- Repository에서 sql.ErrNoRows → domain.ErrNotFound 변환 책임 누락 — 이 변환이 없으면 인프라 에러가 UseCase까지 누출됨
- `%v` vs `%w` 혼동 — `%v`는 에러 체인 끊김, `%w`는 체인 보존. `errors.Is()`가 동작하려면 `%w` 필수

---

## Clean Architecture vs Hexagonal Architecture

→ 상세 비교 및 Go 프로젝트 구조: [[topics/golang/hexagonal-architecture]]

| 항목 | Clean Architecture | Hexagonal (Ports & Adapters) |
|---|---|---|
| 레이어 구분 | 명확한 4~5개 동심원 | 포트(인터페이스) + 어댑터(구현) |
| 의존성 방향 | 안쪽으로만 | 중심 도메인 방향 |
| Go 적합성 | ★★★★☆ | ★★★★★ |
| 복잡도 | 약간 높음 | 실용적 |
| 특징 | 레이어 경계 명확 | 프레임워크 교체 용이 |

> Go에서는 Hexagonal Architecture가 더 실용적이라는 평가가 많음.
> 실무에서는 두 패턴을 혼용하는 경우가 대부분.

---

## 실무 트레이드오프

### 장점
- 레이어별 독립적인 단위 테스트 가능
- DB 교체 (PostgreSQL → MySQL) 시 인터페이스 구현체만 교체
- 비즈니스 로직이 프레임워크 버전에 종속되지 않음

### 단점 및 주의점

**1. 데이터 Mapping 오버헤드**
- Domain Model → DB Model → API Response Model 변환 코드 증가
- 소규모 프로젝트에서는 모델 통합 고려 가능

**2. 과도한 추상화 함정**
```go
// 나쁜 예: 교체 가능성이 없는데 인터페이스 남발
type Logger interface { Log(msg string) }

// 실제로 교체될 가능성이 있는 것만 인터페이스화
type UserRepository interface { ... }  // DB 교체 가능 → 인터페이스 O
```

**3. 소규모 프로젝트 오버엔지니어링**
- 팀 5명 이하, 기능 단순한 경우 → 레이어 2~3개로 단순화 고려
- "필요할 때 복잡도를 추가"하는 Go 철학 준수

**4. 동시성 처리**
- Repository 레이어에서 goroutine 사용 시 data race 주의
- UseCase에서 비동기 조정, Context 전파 필수
- 참고: [[topics/golang/goroutine]], [[topics/golang/context]]

---

## 면접 질문

**Q. Clean Architecture를 왜 사용하고, 어떤 상황에서 적합한가요?**

**난이도**: 중급

**핵심 키워드**: 관심사 분리, 의존성 역전, 테스트 용이성, 유지보수

**모범 답변 방향**:
Clean Architecture를 사용하는 핵심 이유는 비즈니스 로직을 DB나 HTTP 프레임워크 같은 외부 기술에서 분리하기 위해서입니다. 분리되면 DB를 MySQL에서 PostgreSQL로 바꾸거나, HTTP 프레임워크를 교체해도 비즈니스 로직은 변경할 필요가 없습니다. 이것이 가능한 이유는 의존성 역전 원칙(DIP) 덕분입니다. UseCase는 Repository 구현체가 아닌 Repository 인터페이스에 의존하고, 인터페이스는 Domain 레이어에 정의됩니다. 그래서 Infrastructure 레이어에서 구현체를 바꿔도 UseCase 코드는 변경이 없습니다.

또한 Repository 인터페이스가 있기 때문에 테스트 시 실제 DB 대신 Mock 구현체를 주입할 수 있어서 DB 없이도 단위 테스트를 빠르게 실행할 수 있습니다. 샵라이브에서 DB 마이그레이션 작업을 진행할 때 이 구조의 실질적인 이점을 체감했습니다. Repository 인터페이스가 Domain에 선언되어 있어서 DB 구현체만 교체하면 됐고, 비즈니스 로직은 전혀 건드리지 않아도 됐습니다. 적합한 상황은 팀 규모가 크고 비즈니스 도메인이 복잡해서 여러 사람이 독립적으로 작업해야 하는 경우입니다. 레이어별로 역할이 명확히 분리되어 있어서 Domain 팀과 Infrastructure 팀이 병렬로 작업하더라도 인터페이스 계약만 지키면 충돌이 없습니다. 반대로 소규모 서비스나 단순한 CRUD 기능 위주라면 레이어 분리와 Mapping 코드가 오히려 개발 속도를 늦추는 오버엔지니어링이 될 수 있습니다. Go 철학처럼 지금 필요한 복잡도만 도입하고, 비즈니스 로직이 복잡해질 때 단계적으로 레이어를 추가하는 방식이 현실적입니다. 중요한 것은 Clean Architecture가 어느 시점에서나 강제로 도입할 구조가 아니라, 코드베이스가 커지고 팀이 병렬 작업이 많아질 때 자연스럽게 선택하게 되는 구조라는 점입니다. 초기부터 완벽하게 구현하려다 개발 속도를 잃는 것보다, 핵심 원칙(인터페이스를 통한 의존성 역전, 비즈니스 로직의 기술 분리)만 지키면서 점진적으로 구조를 정리하는 방식이 실무에서 더 효과적입니다.

**꼬리 질문**:
- Repository 인터페이스는 어느 레이어에 위치해야 하나요? (Domain 레이어 — 구현은 Infrastructure)
- UseCase와 Service의 차이는? (UseCase는 단일 비즈니스 기능, Service는 여러 기능 묶음)

---

**Q. 도메인 모델과 DB 모델을 왜 분리해야 하나요?**

**난이도**: 중급

**핵심 키워드**: 모델 분리, 비즈니스 규칙, ORM 오염, 변경 격리

**모범 답변 방향**:
도메인 모델과 DB 모델을 분리하는 이유는 두 모델의 목적이 다르기 때문입니다. DB 모델은 저장 최적화에 초점을 두어 컬럼명, 인덱스 힌트, 외래키 관계를 표현합니다. 도메인 엔티티는 비즈니스 규칙과 검증 로직을 포함하며, 메서드를 통해 상태 변경을 제어합니다. 분리하면 DB 스키마가 바뀌어도 비즈니스 로직에 영향을 주지 않습니다. 예를 들어 User 테이블에 컬럼이 추가되거나 정규화가 바뀌어도 도메인의 User 엔티티는 그대로 유지됩니다.

또한 ORM을 사용할 때 GORM의 태그나 JPA 어노테이션이 도메인 엔티티에 붙지 않아서 도메인 코드가 ORM 프레임워크에 오염되지 않습니다. 이것이 실무에서 중요한 이유는 ORM 프레임워크 버전이 바뀌거나 다른 ORM으로 교체할 때, 도메인 로직이 함께 영향을 받는 문제를 방지하기 때문입니다. 샵라이브에서 DB 마이그레이션 작업을 진행할 때 도메인 엔티티와 DB 모델이 분리되어 있었기 때문에 스키마 변경이 비즈니스 레이어까지 파급되지 않았습니다. 분리의 또 다른 이점은 도메인 엔티티를 통해 상태 변화를 명시적으로 제어할 수 있다는 점입니다. `user.Deactivate()`처럼 도메인 언어로 조작하는 메서드를 정의하면, DB 쿼리를 직접 호출하는 것보다 비즈니스 의도가 명확합니다. 트레이드오프는 두 모델 간 변환 코드(Mapping)가 늘어난다는 점인데, Helper 함수나 변환 전용 mapper를 두어 처리하거나, 소규모 서비스라면 두 모델을 통합해서 ORM 모델이 도메인 역할도 겸하게 하는 방식을 고려해야 합니다. 복잡도와 유지보수 비용을 비교해서 상황에 맞는 트레이드오프를 선택하는 것이 중요합니다. 실무에서 실제로 DB를 교체할 일은 많지 않지만, 테스트 시 Mock을 주입하는 이점 하나만으로도 모델 분리를 정당화하기에 충분한 경우가 많습니다. DB 의존성 없이 빠른 단위 테스트를 유지하는 것이 장기적으로 개발 생산성에 큰 기여를 합니다. 또한 도메인 모델이 분리되어 있으면 ORM 없이 순수 Go 구조체로 비즈니스 로직을 표현할 수 있어서, 비즈니스 규칙이 DB 스키마 변경에 끌려다니지 않습니다. 장기적으로 모델 분리는 비즈니스 로직의 가독성과 테스트 커버리지 모두를 높이는 실용적 선택입니다.

**꼬리 질문**:
- 두 모델 간 변환(Mapping) 비용이 크면 어떻게 하나요? (Helper 함수, 소규모에선 통합 고려)

---

**Q. Go에서 의존성 주입(DI)을 어떻게 구현하나요?**

**난이도**: 기초

**핵심 키워드**: 생성자 함수, 인터페이스, 느슨한 결합, Mock

**모범 답변 방향**:
Go에서 의존성 주입은 생성자 함수 패턴으로 구현합니다. `func NewUserService(repo UserRepository, cache Cache) *UserService` 처럼 의존성을 인터페이스 타입으로 받아서 구조체에 저장하는 방식입니다. 인터페이스에 의존하기 때문에 테스트할 때는 실제 구현체 대신 Mock을 주입할 수 있어 DB 없이도 단위 테스트가 가능합니다. Mock 구현체는 testify/mock이나 수동으로 작성한 구조체로 만들 수 있으며, 의존성이 인터페이스로만 노출되어 있기 때문에 테스트 코드가 운영 코드의 구현 세부사항을 전혀 몰라도 됩니다.

조립 순서는 `main.go`에서 Infrastructure(Repository, Cache) → UseCase → Delivery(Handler) 순서로 진행합니다. 각 단계의 생성자에 이전 단계에서 만든 인터페이스 구현체를 넘겨주는 방식입니다. 이 조립 로직이 한 곳에 집중되어 있어서 의존성 그래프 전체를 한눈에 파악할 수 있습니다. 의존성이 단순할 때는 외부 DI 프레임워크 없이 직접 조립해도 충분하고, 오히려 이 방식이 Go다운 단순함을 유지합니다. 의존성 그래프가 복잡해지면 `google/wire`나 `uber/fx` 같은 도구를 도입해서 코드 생성이나 reflect 기반으로 자동 조립하는 방식을 고려합니다. wire는 컴파일 타임에 의존성 그래프를 검증하고 조립 코드를 생성하기 때문에 런타임 오류가 없고, fx는 런타임에 reflect로 처리하지만 라이프사이클 관리(OnStart, OnStop)가 내장되어 있어 graceful shutdown 처리가 편리합니다. 카테노이드에서 채팅 서버의 의존성 그래프가 복잡해졌을 때 수동 조립 코드에서 wire로 마이그레이션해서 컴파일 타임 검증의 이점을 활용했습니다. 의존성 주입 패턴에서 중요한 또 다른 점은 전역 변수나 싱글톤을 피하는 것입니다. 전역 상태를 사용하면 테스트 격리가 어려워지고 동시성 버그가 생기기 쉽습니다. 생성자 함수로 명시적으로 의존성을 주입하면 각 테스트가 독립적으로 자신만의 인스턴스를 갖기 때문에 병렬 테스트도 안전하게 실행할 수 있습니다.

**꼬리 질문**:
- google/wire나 uber/fx 같은 DI 도구를 언제 도입하나요? (의존성 그래프 복잡해질 때)

---

**Q. Feature-based와 Type-based 프로젝트 구조의 차이와 선택 기준은?**

**난이도**: 기초

**핵심 키워드**: 응집력, 가독성, 모듈화, 프로젝트 규모

**모범 답변 방향**:
Feature-based 구조는 `internal/user/{entity, usecase, repository, handler}` 처럼 기능(도메인) 단위로 파일을 묶는 방식입니다. 새 팀원이 "User 관련 코드는 어디 있나?"라는 질문에 바로 한 폴더를 찾아가면 된다는 장점이 있고, 기능을 추가하거나 삭제할 때도 한 폴더에서 작업이 끝납니다. 코드 응집도가 높아서 한 기능에 관련된 모든 코드를 한 곳에서 볼 수 있고, 기능 간 의존성도 파일 import를 통해 명확하게 드러납니다.

Type-based 구조는 `internal/{entity, usecase, repository, handler}` 처럼 타입별로 묶는 방식입니다. 같은 타입(예: 모든 UseCase)을 한눈에 보기는 좋지만, 새 기능을 추가하면 여러 폴더를 동시에 수정해야 해서 규모가 커질수록 파일 탐색 비용이 급증합니다. User 기능 하나를 추가하면 entity, usecase, repository, handler 폴더를 모두 열어야 하는데, 이 비용이 팀 규모가 커질수록 더 크게 느껴집니다. 현재는 Feature-based를 권장하는 추세입니다. Go 공식 프로젝트 레이아웃 제안이나 실무에서 많이 참고하는 go-clean-arch 레퍼런스도 Feature-based 또는 그와 유사한 방식을 사용합니다. 단, 팀이 이미 Type-based에 익숙하고 프로젝트가 작다면 굳이 바꿀 필요는 없습니다. 구조를 바꾸는 비용보다 익숙한 구조에서 빠르게 개발하는 이점이 더 클 수 있습니다. 실무에서는 Feature-based로 시작해서 한 폴더가 너무 커지면 그때 하위 폴더로 분리하는 방식이 현실적입니다. 구조는 수단이지 목적이 아닙니다. 팀이 코드를 찾고, 이해하고, 수정하는 데 드는 시간이 최소화되는 구조가 좋은 구조입니다. Feature-based가 권장되는 이유도 결국 그 기준에 부합하기 때문입니다. 기능 단위로 묶으면 변경의 영향 범위가 한 폴더 내에서 끝나고, 코드 리뷰 시에도 해당 기능의 모든 레이어를 한 PR에서 확인할 수 있어 컨텍스트 전환 비용이 줄어듭니다.

---

**Q. Clean Architecture에서 에러를 레이어별로 어떻게 처리해야 하나요?**

**난이도**: 중급

**핵심 키워드**: 에러 변환, 비즈니스 에러, HTTP 상태코드, errors.Is

**모범 답변 방향**:
Clean Architecture에서 에러는 레이어를 넘어갈 때 반드시 해당 레이어의 언어로 변환해야 합니다. Domain 레이어에서는 `ErrNotFound`, `ErrEmailExists` 같은 비즈니스 에러를 정의합니다. 이 에러들은 특정 DB 기술에 의존하지 않는 순수한 비즈니스 개념입니다.

Repository 레이어에서는 DB 에러(예: `sql.ErrNoRows`)를 받아 Domain 에러로 변환합니다. `sql.ErrNoRows`를 그대로 UseCase로 올리면 UseCase가 DB 기술에 의존하게 되어 레이어 경계가 무너집니다. 변환 방식은 `errors.Is(err, sql.ErrNoRows)`로 감지한 뒤 `return nil, domain.ErrNotFound`로 교체하는 것입니다. 이것은 래핑이 아니라 교체라는 점이 핵심입니다. 래핑하면 `sql.ErrNoRows`가 체인에 남아서 UseCase가 `errors.Is(err, sql.ErrNoRows)`로 DB 에러를 감지할 수 있게 됩니다. 그래서 인프라 에러는 반드시 교체해야 합니다. UseCase 레이어에서는 `fmt.Errorf("createUser: %w", err)` 형태로 컨텍스트를 추가해 wrap하고, 로깅이 필요하면 이 레이어에서 처리합니다. 에러를 로깅하고 다시 반환하면 상위 레이어에서 또 로깅해 중복 로그가 쌓이므로 로깅하거나 반환하거나 둘 중 하나만 해야 합니다. Delivery 레이어(HTTP Handler)에서는 `errors.Is(err, domain.ErrNotFound)`로 에러 종류를 판별해 HTTP 상태코드(404, 409 등)로 변환합니다. 에러 타입이 레이어 경계를 넘어 누출되면, 예를 들어 `*pq.Error` 같은 DB 드라이버 에러가 Handler까지 올라오면, 레이어 분리가 사실상 무너진 것과 같습니다.

참고: [[topics/golang/error-handling]]

---

## 참고 링크
- [go-clean-arch (bxcodec)](https://github.com/bxcodec/go-clean-arch) — 가장 유명한 Go Clean Architecture 레퍼런스
- [Wild Workouts (Three Dots Labs)](https://threedots.tech/post/introducing-clean-architecture/) — 실무 수준 예제
- [Clean Architecture in Go - Practical Guide (DEV)](https://dev.to/kittipat1413/structuring-a-go-project-with-clean-architecture-a-practical-example-3b3f)
- [Why Clean Architecture Struggles in Golang (DEV)](https://dev.to/lucasdeataides/why-clean-architecture-struggles-in-golang-and-what-works-better-m4g)
- [Clean Architecture in Go - pkritiotis.io](https://pkritiotis.io/clean-architecture-in-golang/)
