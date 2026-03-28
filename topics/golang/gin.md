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

**모범 답변 방향**:
- `gin.Engine.ServeHTTP` → 라우터 매칭 → `HandlersChain` 슬라이스 결정
- `c.Next()` 호출 시 index를 증가시키며 다음 핸들러를 순차 실행
- `c.Next()` 이전 코드 = 전처리, 이후 코드 = 후처리 (양파 모델)
- 체인이 모두 소진되면 ResponseWriter가 flush되어 응답 반환

**꼬리 질문**:
- `c.Next()`를 호출하지 않으면 어떻게 되나요?
- `c.Abort()`와 `return`의 차이는 무엇인가요?

---

### Q2. c.Abort()와 return의 차이점은 무엇인가요?

**난이도**: 중급

**핵심 키워드**: index = abortIndex, 체인 중단, 현재 미들웨어 실행

**모범 답변 방향**:
- `return` 만 사용하면 현재 함수는 종료되지만 `c.Next()`가 루프를 계속 돌며 다음 핸들러를 실행함
- `c.Abort()`는 index를 `abortIndex(63)`으로 설정해 이후 모든 핸들러 실행을 막음
- 인증 실패, 권한 없음 등 체인을 완전히 끊어야 할 때 `c.Abort()` 또는 `c.AbortWithStatusJSON()` 사용

**꼬리 질문**:
- `c.Abort()` 이후에도 현재 미들웨어 코드가 실행되는 이유는?
- Recovery 미들웨어에서 panic을 잡은 후 왜 `c.Abort()`를 호출하나요?

---

### Q3. gin.New()와 gin.Default()의 차이와 실무에서의 선택 기준은?

**난이도**: 기초

**핵심 키워드**: Logger, Recovery, 커스텀 미들웨어

**모범 답변 방향**:
- `gin.Default()` = `gin.New()` + Logger + Recovery 자동 등록
- 실무에서는 `gin.New()`로 시작해 필요한 미들웨어만 직접 `Use()` 등록
- 이유: 커스텀 로거 (Zap 등) 사용, 별도 panic 핸들러, 불필요한 로그 제거

---

### Q4. 미들웨어에서 goroutine을 사용할 때 주의할 점은?

**난이도**: 심화

**핵심 키워드**: c.Copy(), race condition, Context 수명

**모범 답변 방향**:
- `*gin.Context`는 요청 처리가 끝나면 pool로 반환됨 → 비동기 goroutine에서 접근 시 race condition
- `c.Copy()`로 Context를 복사해 goroutine에 전달해야 안전
- 복사본은 `c.Next()`, `c.Abort()` 등 체인 제어 불가 (읽기 전용으로 사용)

**꼬리 질문**:
- `c.Set`/`c.Get`은 thread-safe한가요?
- goroutine 내에서 응답을 쓰면 안 되는 이유는?

### Q5. *gin.Context를 context.Context 파라미터에 그대로 넘기면 왜 안 되나요?

**난이도**: 중급

**핵심 키워드**: context.Context 인터페이스, c.Request.Context(), 레이어 분리, Done() nil

**모범 답변 방향**:
- `context.Context`는 4개 메서드를 요구하는 인터페이스. gin 버전에 따라 `*gin.Context`가 이를 완전 구현 안 할 수 있어 컴파일 에러 발생
- 구현해도 `Done()`이 nil 반환 → 요청 취소 신호 전파 안 됨 → 런타임 버그
- 올바른 방법: `c.Request.Context()` — HTTP 요청에 붙은 표준 context 사용
- 레이어 분리 원칙: Service/Repository는 `context.Context`만 알아야 함. `*gin.Context`를 내려보내면 Gin 의존성이 생겨 테스트 어려워짐

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