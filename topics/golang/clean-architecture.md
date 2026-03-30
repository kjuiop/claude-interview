---
tags: [golang, go, clean-architecture, architecture, hexagonal, solid]
related: [interface, error-handling, system-design]
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
- 비즈니스 로직(UseCase)을 DB·HTTP 프레임워크와 분리 → 각각 독립적 변경 가능
- Repository 인터페이스 덕분에 테스트 시 Mock으로 교체 가능 → DB 없는 단위 테스트
- 팀 규모가 크고 도메인 복잡도가 높을 때 적합. 소규모·단순 프로젝트엔 오버엔지니어링

**꼬리 질문**:
- Repository 인터페이스는 어느 레이어에 위치해야 하나요? (Domain 레이어 — 구현은 Infrastructure)
- UseCase와 Service의 차이는? (UseCase는 단일 비즈니스 기능, Service는 여러 기능 묶음)

---

**Q. 도메인 모델과 DB 모델을 왜 분리해야 하나요?**

**난이도**: 중급

**핵심 키워드**: 모델 분리, 비즈니스 규칙, ORM 오염, 변경 격리

**모범 답변 방향**:
- DB 모델: 저장 최적화 목적 (컬럼명, 인덱스, 관계)
- 도메인 엔티티: 비즈니스 규칙과 검증 로직 포함
- 분리 시 DB 스키마 변경이 비즈니스 로직에 영향 없음
- ORM 어노테이션·태그가 도메인 엔티티를 오염시키지 않음

**꼬리 질문**:
- 두 모델 간 변환(Mapping) 비용이 크면 어떻게 하나요? (Helper 함수, 소규모에선 통합 고려)

---

**Q. Go에서 의존성 주입(DI)을 어떻게 구현하나요?**

**난이도**: 기초

**핵심 키워드**: 생성자 함수, 인터페이스, 느슨한 결합, Mock

**모범 답변 방향**:
- Go 관례: `NewXXX(dep Interface) *XXX` 생성자 함수로 의존성 주입
- 외부 DI 프레임워크(Wire, fx) 없이 main.go에서 직접 조립도 가능
- 인터페이스에 의존하므로 테스트 시 Mock 구현체 주입 → DB 없는 단위 테스트
- main.go에서 Infrastructure → UseCase → Delivery 순서로 조립

**꼬리 질문**:
- google/wire나 uber/fx 같은 DI 도구를 언제 도입하나요? (의존성 그래프 복잡해질 때)

---

**Q. Feature-based와 Type-based 프로젝트 구조의 차이와 선택 기준은?**

**난이도**: 기초

**핵심 키워드**: 응집력, 가독성, 모듈화, 프로젝트 규모

**모범 답변 방향**:
- Feature-based: `internal/user/{entity, usecase, repository, handler}` — 기능별 응집력 높음, 새 팀원 파악 용이
- Type-based: `internal/{entity, usecase, repository, handler}` — 타입별 정렬, 기능 추가 시 여러 폴더 수정
- 2025 권장: Feature-based. 규모가 커질수록 Type-based는 파일 탐색 비용 급증

---

**Q. Clean Architecture에서 에러를 레이어별로 어떻게 처리해야 하나요?**

**난이도**: 중급

**핵심 키워드**: 에러 변환, 비즈니스 에러, HTTP 상태코드, errors.Is

**모범 답변 방향**:
- Domain: `ErrNotFound`, `ErrEmailExists` 등 비즈니스 에러 정의
- Repository: DB 에러(`sql.ErrNoRows`) → 도메인 에러로 변환
- UseCase: `fmt.Errorf("context: %w", err)` 로 wrap, 로깅
- Delivery: `errors.Is(err, domain.ErrNotFound)` → HTTP 404 매핑
- 레이어 간 에러 타입이 漏출되면 레이어 경계가 무너진다

참고: [[topics/golang/error-handling]]

---

## 참고 링크
- [go-clean-arch (bxcodec)](https://github.com/bxcodec/go-clean-arch) — 가장 유명한 Go Clean Architecture 레퍼런스
- [Wild Workouts (Three Dots Labs)](https://threedots.tech/post/introducing-clean-architecture/) — 실무 수준 예제
- [Clean Architecture in Go - Practical Guide (DEV)](https://dev.to/kittipat1413/structuring-a-go-project-with-clean-architecture-a-practical-example-3b3f)
- [Why Clean Architecture Struggles in Golang (DEV)](https://dev.to/lucasdeataides/why-clean-architecture-struggles-in-golang-and-what-works-better-m4g)
- [Clean Architecture in Go - pkritiotis.io](https://pkritiotis.io/clean-architecture-in-golang/)
