---
tags: [golang, go, gin, middleware, http, web-framework]
related: [goroutine, context, error-handling, interface]
---

# Golang — Gin Middleware

→ [[home]] | [[topics/golang/goroutine]] | [[topics/golang/context]] | [[topics/golang/error-handling]]

---

## Gin Middleware 란

HTTP 요청이 라우트 핸들러에 도달하기 전/후에 실행되는 함수.
**Cross-cutting concern** (인증, 로깅, 복구, CORS 등)을 핸들러 로직과 분리하기 위해 사용.

---

## Request → Response 전체 흐름

```
[Client Request]
       ↓
[net/http.Server] — HTTP 파싱, 커넥션 관리
       ↓
[gin.Engine.ServeHTTP] — 라우터 매칭 (httprouter 기반)
       ↓
[HandlersChain 실행 시작]
       │
       ├─ Middleware 1 (전처리)   ← c.index = 0
       │      ↓ c.Next()
       ├─ Middleware 2 (전처리)   ← c.index = 1
       │      ↓ c.Next()
       ├─ Route Handler           ← c.index = 2
       │      ↓ (return)
       ├─ Middleware 2 (후처리)   ← c.Next() 이후 코드
       │      ↓
       └─ Middleware 1 (후처리)   ← c.Next() 이후 코드
               ↓
       [ResponseWriter flush]
       ↓
[Client Response]
```

**핵심**: `c.Next()` 기준으로 미들웨어가 양방향으로 실행됨 — **양파 모델 (Onion Model)**.

---

## HandlersChain 내부 구조

```go
// gin 내부 타입 정의
type HandlersChain []HandlerFunc
type HandlerFunc func(*Context)

// Context 내부
type Context struct {
    handlers HandlersChain  // [MW1, MW2, RouteHandler] 슬라이스
    index    int8            // 현재 실행 중인 핸들러 인덱스 (-1에서 시작)
    // ...
}
```

`gin.Engine.Use()` 또는 라우트 등록 시 핸들러들이 이 슬라이스에 append됨.

---

## c.Next() 동작 원리

```go
func (c *Context) Next() {
    c.index++
    for c.index < int8(len(c.handlers)) {
        c.handlers[c.index](c)  // 다음 핸들러 실행
        c.index++
    }
}
```

- `c.Next()` 호출 → index 증가 → 다음 핸들러 실행
- 다음 핸들러가 종료되면 → 호출한 미들웨어의 `c.Next()` 이후 코드 실행
- **재귀 없이 반복문으로 구현** → 스택 깊이 무관하게 안전

```go
// 실제 미들웨어 코드 구조
func LoggerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()          // ① 전처리

        c.Next()                     // ② 이후 체인 전부 실행

        duration := time.Since(start) // ③ 후처리 (응답 완료 후)
        log.Printf("duration: %v", duration)
    }
}
```

---

## c.Abort() 동작 원리

```go
const abortIndex int8 = math.MaxInt8 / 2  // 63

func (c *Context) Abort() {
    c.index = abortIndex  // index를 최대값으로 설정
}
```

- `c.Abort()` 호출 → 이후 체인의 핸들러가 **실행되지 않음**
- 단, 현재 미들웨어의 `c.Abort()` **이후 코드는 계속 실행됨**
- 응답을 함께 보내려면 `c.AbortWithStatus()` / `c.AbortWithStatusJSON()` 사용

```go
// 인증 미들웨어 예시
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if !isValid(token) {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "unauthorized",
            })
            return  // 현재 함수도 종료
        }
        c.Next()  // 인증 성공 시 체인 계속
    }
}
```

---

## 미들웨어 등록 범위

```go
r := gin.New()

// 1. 전역 — 모든 요청
r.Use(gin.Logger())
r.Use(gin.Recovery())

// 2. 그룹 — 특정 경로 그룹만
api := r.Group("/api")
api.Use(AuthMiddleware())
{
    api.GET("/users", handleGetUsers)
    api.POST("/users", handleCreateUser)
}

// 3. 개별 라우트 — 단일 핸들러에만
r.GET("/admin", AdminMiddleware(), handleAdmin)
```

최종 HandlersChain = `전역 MW 목록` + `그룹 MW 목록` + `라우트 MW 목록` + `라우트 핸들러`

---

## gin.Default() vs gin.New()

| | `gin.Default()` | `gin.New()` |
|---|---|---|
| Logger | ✅ 기본 포함 | ❌ 없음 |
| Recovery | ✅ 기본 포함 | ❌ 없음 |
| 사용 시 | 빠른 개발/테스트 | 커스텀 미들웨어 직접 구성 |

실무에서는 `gin.New()` + 직접 구성이 일반적 (불필요한 로거 제거, 커스텀 복구 처리).

---

## 주요 내장 미들웨어

### gin.Logger()
요청 메서드, 경로, 상태코드, 처리 시간을 stdout에 출력.

### gin.Recovery()
panic을 recover()로 잡아 500 응답 반환 → 서버 다운 방지.

```go
func Recovery() HandlerFunc {
    return func(c *Context) {
        defer func() {
            if err := recover(); err != nil {
                c.AbortWithStatus(http.StatusInternalServerError)
            }
        }()
        c.Next()
    }
}
```

---

## 실전 패턴: 미들웨어에서 값 전달

```go
// 미들웨어에서 값 설정
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := extractUserID(c)
        c.Set("userID", userID)  // Context에 저장
        c.Next()
    }
}

// 핸들러에서 값 읽기
func handleGetProfile(c *gin.Context) {
    userID, _ := c.Get("userID")
    // ...
}
```

`c.Set` / `c.Get` 은 내부적으로 `sync.RWMutex`로 보호됨 → goroutine-safe.

---

## *gin.Context vs context.Context — 왜 그대로 넘기면 컴파일 에러가 나나?

### 핵심 원인: 인터페이스 불완전 구현 (또는 버전별 차이)

`context.Context`는 Go 표준 라이브러리의 **인터페이스**로, 4개의 메서드를 요구한다:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

`*gin.Context`는 gin 프레임워크의 **구조체**다. 버전에 따라 다르게 동작한다:

| gin 버전 | context.Context 구현 여부 | Done() 동작 |
|---|---|---|
| v1.7.7 이전 | ❌ 구현 안 됨 | — |
| v1.7.7 이후 | ✅ 구현됨 | 하지만 `nil` 반환 (요청 취소 전파 안 됨) |

**컴파일 에러가 나는 경우**: 이전 버전 gin을 사용하거나, `*gin.Context`가 4개 메서드를 모두 구현하지 않은 상태에서 `context.Context` 타입 파라미터에 넘길 때.

```go
// RoomService 인터페이스
type RoomService interface {
    CreateRoom(ctx context.Context, name string) (*Room, error)
}

// Gin 핸들러 — 잘못된 사용
func handleCreateRoom(c *gin.Context) {
    service.CreateRoom(c, "room1")  // ❌ 컴파일 에러 또는 버그
}
```

### 올바른 방법: c.Request.Context()

```go
func handleCreateRoom(c *gin.Context) {
    ctx := c.Request.Context()         // ✅ HTTP 요청에 붙은 표준 context.Context
    service.CreateRoom(ctx, "room1")
}
```

**왜 `c.Request.Context()`가 올바른가:**
- `http.Request.Context()`는 클라이언트 연결이 끊기면 자동으로 취소되는 표준 context
- `gin.Context.Done()`은 nil을 반환 → 요청 취소 신호를 받지 못함
- 표준 `context.Context`로 전달해야 DB 쿼리, HTTP 호출 등이 클라이언트 연결 끊김에 반응할 수 있음

### 레이어 경계에서의 원칙

```go
// Handler Layer — gin 의존
func (h *Handler) CreateRoom(c *gin.Context) {
    var req CreateRoomRequest
    c.ShouldBindJSON(&req)

    ctx := c.Request.Context()  // ← 여기서 context 추출
    room, err := h.roomService.CreateRoom(ctx, req.Name)  // Service는 context.Context만 알면 됨
    // ...
}

// Service Layer — gin 모름, 순수 Go
type RoomService interface {
    CreateRoom(ctx context.Context, name string) (*Room, error)
}
```

**원칙**: Service/Repository 계층은 `context.Context`만 알아야 한다. `*gin.Context`를 서비스 레이어까지 내려보내면 Gin에 대한 의존성이 생겨 테스트가 어려워지고 레이어 분리가 깨진다.

### 인터페이스 관점에서의 교훈

Go의 인터페이스는 **암시적 구현**이다. `*gin.Context`가 `context.Context`의 4개 메서드를 모두 구현하면 자동으로 인터페이스를 만족한다. 하지만 메서드 시그니처만 맞고 **동작이 올바르지 않으면** (Done()이 nil 반환) 컴파일은 통과해도 런타임 버그가 생긴다.

→ 참고: [[topics/golang/interface#개념]] | [[topics/golang/context#개념]]

> 출처: https://github.com/gin-gonic/gin/issues/1734

---

## 주의할 점

1. **`c.Next()` 생략** — 명시적으로 호출하지 않으면 다음 핸들러가 실행되지 않음
   - `c.Abort()`와 달리 단순 누락이므로 버그 추적 어려움
2. **미들웨어에서 goroutine 사용 시** — `c.Copy()`로 Context 복사 필요
   ```go
   cCopy := c.Copy()
   go processAsync(cCopy)
   ```
3. **응답 중복 쓰기** — `c.JSON()` 이후 `c.Abort()` 없이 계속 실행 시 헤더 중복 오류
4. **등록 순서** — 미들웨어 순서가 실행 순서와 직결됨 (Logger → Auth → Handler 권장)

---

## 면접 질문

### Q1. Gin의 미들웨어 실행 흐름을 Request부터 Response까지 설명해주세요.

**난이도**: 중급

**핵심 키워드**: HandlersChain, index, c.Next(), 양파 모델

**모범 답변**:

Gin의 미들웨어 실행 흐름은 HandlersChain이라는 슬라이스를 중심으로 동작합니다. HTTP 요청이 들어오면 `gin.Engine.ServeHTTP`가 httprouter 기반으로 라우터를 매칭하고, 해당 경로에 등록된 전역 미들웨어 + 그룹 미들웨어 + 라우트 핸들러를 하나의 HandlersChain 슬라이스로 결합합니다. 실행은 `c.index`라는 int8 값을 증가시키면서 슬라이스를 순서대로 호출하는 방식입니다. 각 미들웨어에서 `c.Next()`를 호출하면 index가 증가하며 다음 핸들러가 실행됩니다. `c.Next()` 이전 코드가 전처리, `c.Next()` 이후 코드가 후처리로 동작하기 때문에 미들웨어가 요청 전과 응답 후 모두에 개입하는 양파 모델이 완성됩니다. 체인이 모두 소진되면 ResponseWriter가 flush되어 클라이언트에 응답이 반환됩니다.

**꼬리 질문**:
- `c.Next()`를 호출하지 않으면 어떻게 되나요?
- `c.Abort()`와 `return`의 차이는 무엇인가요?

---

### Q2. c.Abort()와 return의 차이점은 무엇인가요?

**난이도**: 중급

**핵심 키워드**: index = abortIndex, 체인 중단, 현재 미들웨어 실행

**모범 답변**:

`return`과 `c.Abort()`는 동작 범위가 다릅니다. 미들웨어에서 `return`만 사용하면 현재 함수는 종료되지만, `c.Next()`가 루프 방식으로 구현되어 있기 때문에 루프가 계속 돌면서 다음 핸들러들이 그대로 실행됩니다. 반면 `c.Abort()`는 `c.index`를 내부 상수인 `abortIndex(63)`으로 설정해버려서, 이후의 모든 핸들러를 건너뛰고 체인을 완전히 중단시킵니다. 단, `c.Abort()` 이후에도 현재 미들웨어 함수 내부의 남은 코드는 계속 실행되므로, 응답을 보낸 뒤에는 명시적으로 `return`까지 함께 써야 합니다. 인증 실패나 권한 없음처럼 이후 핸들러를 아예 실행해서는 안 되는 상황에서는 반드시 `c.Abort()` 또는 응답까지 함께 보내는 `c.AbortWithStatusJSON()`을 사용해야 합니다.

**꼬리 질문**:
- `c.Abort()` 이후에도 현재 미들웨어 코드가 실행되는 이유는?
- Recovery 미들웨어에서 panic을 잡은 후 왜 `c.Abort()`를 호출하나요?

---

### Q3. gin.New()와 gin.Default()의 차이와 실무에서의 선택 기준은?

**난이도**: 기초

**핵심 키워드**: Logger, Recovery, 커스텀 미들웨어

**모범 답변**:

`gin.Default()`는 `gin.New()`에 Logger와 Recovery 미들웨어를 자동으로 등록한 편의 생성자입니다. 빠른 프로토타입 개발이나 테스트 환경에서는 편리하지만, 실무에서는 `gin.New()`로 시작해 필요한 미들웨어만 직접 `Use()`로 등록하는 방식이 일반적입니다. 이유는 세 가지입니다. 첫째, 기본 Logger는 stdout에 텍스트 형식으로 출력하는데, 실무에서는 구조화 로깅을 위해 Zap이나 zerolog 같은 커스텀 로거를 사용합니다. 둘째, Recovery 미들웨어도 팀의 에러 리포팅 시스템이나 Sentry와 연동하는 커스텀 버전이 필요한 경우가 많습니다. 셋째, 불필요한 미들웨어는 매 요청마다 오버헤드를 발생시키므로 실제로 필요한 것만 등록하는 편이 낫습니다.

---

### Q4. 미들웨어에서 goroutine을 사용할 때 주의할 점은?

**난이도**: 심화

**핵심 키워드**: c.Copy(), race condition, Context 수명

**모범 답변**:

미들웨어나 핸들러에서 goroutine을 띄울 때 `*gin.Context`를 그대로 goroutine에 넘기면 race condition이 발생할 수 있습니다. `*gin.Context`는 요청 처리가 완료되면 Gin 내부 pool로 반환되어 재사용되는데, 비동기 goroutine이 이 시점에도 해당 Context에 접근하면 이미 다른 요청이 사용 중인 Context를 건드리는 상황이 됩니다. 올바른 방법은 `c.Copy()`로 Context의 스냅샷을 만들어 goroutine에 전달하는 것입니다. 복사본은 `c.Next()`나 `c.Abort()` 같은 체인 제어 메서드를 사용할 수 없어 읽기 전용으로만 활용해야 하지만, 요청 데이터와 파라미터에는 안전하게 접근할 수 있습니다. 또한 goroutine 내에서 응답을 쓰는 것도 금물인데, 핸들러가 이미 응답을 완료한 뒤 goroutine이 `c.JSON()`을 호출하면 헤더 중복 쓰기 오류가 발생합니다.

**꼬리 질문**:
- `c.Set`/`c.Get`은 thread-safe한가요?
- goroutine 내에서 응답을 쓰면 안 되는 이유는?

### Q5. *gin.Context를 context.Context 파라미터에 그대로 넘기면 왜 안 되나요?

**난이도**: 중급

**핵심 키워드**: context.Context 인터페이스, c.Request.Context(), 레이어 분리, Done() nil

**모범 답변**:

`*gin.Context`를 `context.Context` 파라미터에 그대로 넘기면 두 가지 문제가 생깁니다. 첫째, `context.Context`는 `Deadline()`, `Done()`, `Err()`, `Value()` 4개 메서드를 요구하는 인터페이스인데, 구버전 Gin에서는 `*gin.Context`가 이를 완전히 구현하지 않아 컴파일 에러가 발생합니다. 둘째, v1.7.7 이후 Gin은 4개 메서드를 구현하지만 `Done()`이 nil을 반환하기 때문에, 컴파일은 통과해도 클라이언트 연결이 끊겨도 취소 신호가 전파되지 않는 런타임 버그가 생깁니다. 올바른 방법은 핸들러 레이어에서 `c.Request.Context()`로 표준 `context.Context`를 추출해 서비스 레이어로 넘기는 것입니다. `http.Request.Context()`는 클라이언트 연결이 끊기면 자동으로 취소되므로 DB 쿼리나 외부 API 호출이 클라이언트 상태에 반응할 수 있습니다. 레이어 분리 측면에서도 Service와 Repository는 `context.Context`만 알아야 합니다. `*gin.Context`를 서비스 레이어까지 내려보내면 Gin에 대한 의존성이 생겨 단위 테스트에서 `*gin.Context`를 직접 생성해야 하는 번거로움이 발생하고 레이어 경계가 무너집니다.

**꼬리 질문**:
- `c.Request.Context()`와 `context.Background()`의 차이는 무엇인가요?
- Service 레이어에 `*gin.Context`를 넘기면 왜 테스트가 어려워지나요?
- gin.Context의 `c.Set`/`c.Get`으로 저장한 값은 `c.Request.Context().Value()`로 꺼낼 수 있나요?

> 출처: https://github.com/gin-gonic/gin/issues/1734

---

> 출처:
> - https://leapcell.io/blog/gin-framework-middleware-deep-dive-from-logging-to-recovery
> - https://www.geeksforgeeks.org/go-language/middleware-in-gin-golang/
> - https://medium.com/@arjun.devb25/understanding-handlerschain-in-go-with-gin-a-powerful-middleware-pattern-for-scalable-apis-215771f62a5c
> - https://pkg.go.dev/github.com/gin-gonic/gin