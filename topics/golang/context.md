---
tags: [golang, go, context, cancellation, timeout]
related: [goroutine, channel]
---

# Golang — Context

→ [[home]] | [[topics/golang/goroutine]] | [[topics/golang/channel]]

---

## 개념

goroutine 간 **취소 신호, 데드라인, 요청 범위 값** 전달.

```go
// 함수 첫 번째 파라미터로 항상 전달
func fetchData(ctx context.Context, id string) (Data, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    // ctx가 취소되면 req도 자동 취소
}
```

**4가지 핵심 기능:**
1. Cancellation propagation — 부모 취소 시 자식 goroutine 전체 취소
2. Deadline/Timeout — `context.WithTimeout(ctx, 3*time.Second)`
3. Request-scoped values — `context.WithValue` (남용 금지)
4. Observability — trace ID 등 전파

**규칙:**
- struct 필드로 저장하지 말 것 (파라미터로만 전달)
- nil context 대신 `context.Background()` 또는 `context.TODO()` 사용

### 💬 면접 답변 형태로 읽기

Context는 goroutine 트리 전체에 취소 신호, 타임아웃, 요청 범위 값을 일관되게 전파하기 위해 사용하는 메커니즘입니다. 가장 핵심적인 역할은 취소 전파입니다. 부모 context가 취소되면 그 자식으로 파생된 모든 context도 자동으로 취소되어, 해당 context를 받은 goroutine들이 `ctx.Done()` 채널을 통해 종료 신호를 받을 수 있습니다. 예를 들어 HTTP 요청이 들어왔을 때 DB 쿼리, 외부 API 호출, 캐시 조회를 여러 goroutine에서 병렬로 처리한다면, 클라이언트가 연결을 끊는 순간 request context가 취소되고 그 취소가 모든 하위 작업까지 전파되어 불필요한 처리를 자동으로 중단시킵니다.

두 번째는 타임아웃과 데드라인입니다. `context.WithTimeout(ctx, 3*time.Second)` 으로 일정 시간 내에 작업이 완료되지 않으면 자동으로 취소되도록 설정할 수 있어서, 외부 API 호출이나 DB 쿼리에서 무한 대기를 방지합니다. WithTimeout과 WithDeadline의 차이는 WithTimeout이 현재 시각으로부터 상대적 시간을 받고, WithDeadline은 절대 시각을 받는다는 점입니다.

올바른 사용 방법에서 중요한 규칙이 두 가지 있습니다. 첫째, context는 반드시 함수의 첫 번째 파라미터로 전달해야 하며 struct 필드로 저장해서는 안 됩니다. struct에 저장하면 context의 수명을 제어하기 어려워지고, 어디서 어떻게 쓰이는지 추적이 힘들어집니다. Go 커뮤니티에서는 이를 안티패턴으로 명시하고 있습니다. 둘째, nil context를 넘기면 런타임 패닉이 발생할 수 있으므로, 아직 어떤 context를 써야 할지 모를 때는 `context.TODO()`, 최상위 진입점인 main이나 테스트에서는 `context.Background()` 를 사용합니다.

`context.WithValue`는 request-scoped 값, 예를 들어 trace ID나 user ID를 전파할 때 사용하지만 남용하면 안 됩니다. 함수 시그니처에 명시되지 않은 의존성이 context 안에 숨어 있으면 코드를 이해하기 어려워지고 타입 안전성도 없습니다. 비즈니스 로직에 필요한 파라미터는 명시적으로 함수 인자로 전달하고, context.WithValue는 미들웨어 레이어에서 전달하는 메타데이터 수준으로 제한하는 것이 좋습니다.

---

## 면접 질문

**Q. Gin 핸들러에서 goroutine을 띄울 때 c.Request.Context()를 그대로 넘기면 어떤 문제가 생기나요?**

HTTP 핸들러가 return하면 request context가 cancel된다. goroutine은 아직 실행 중인데 `ctx.Done()`이 닫혀 작업이 중단된다.
백그라운드 goroutine은 request lifecycle과 독립적이어야 하므로, `context.Background()` 또는 `context.WithTimeout(context.Background(), d)` 로 별도 context를 생성해 넘겨야 한다.

```go
// ❌ request 종료 시 goroutine도 강제 중단됨
go processAsync(c.Request.Context(), data)

// ✅ 독립적인 context 사용
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
go processAsync(ctx, data)
```

**defer cancel() 필수 패턴:**
WithCancel / WithTimeout / WithDeadline 모두 cancel 함수를 반환한다.
`defer cancel()`을 빠뜨리면 context 내부 goroutine과 채널 리소스가 해제되지 않아 요청마다 조금씩 메모리가 누수된다.

```go
ctx, cancel := context.WithTimeout(parentCtx, 3*time.Second)
defer cancel() // 반드시 호출
```

**면접 세션 피드백 (2026-04-13 1회차)**:
- WithCancel/WithTimeout/WithDeadline 큰 틀 맞음
- 취소 전파 방향 혼동: "취소를 못 받는 것" ❌ → "너무 일찍 취소되는 것" ✅ (request 종료 시 context cancel)
- defer cancel() 패턴 누락
- WithTimeout(상대 duration) vs WithDeadline(절대 time.Time) 차이 불명확

---

**Q. Context를 왜 사용하고, 어떻게 올바르게 전달하나요?**

Context는 goroutine 트리 전체에 취소 신호, 타임아웃, 요청 범위 값을 일관되게 전파하기 위해 사용합니다. 가장 핵심적인 역할은 취소 전파입니다. 부모 context가 취소되면 그 자식으로 파생된 모든 context도 자동으로 취소되어, 해당 context를 받은 goroutine들이 `ctx.Done()` 채널을 통해 종료 신호를 받을 수 있습니다. 두 번째는 타임아웃과 데드라인입니다. `context.WithTimeout` 으로 일정 시간 내에 작업이 완료되지 않으면 자동으로 취소되도록 설정할 수 있어서, 외부 API 호출이나 DB 쿼리에 유용합니다. 올바른 전달 방법에서 중요한 규칙이 두 가지 있습니다. 첫째, context는 반드시 함수의 첫 번째 파라미터로 전달해야 하며 struct 필드로 저장해서는 안 됩니다. struct에 저장하면 context의 수명을 제어하기 어려워지고, 어디서 어떻게 쓰이는지 추적이 힘들어집니다. 둘째, nil context를 넘기면 런타임 패닉이 발생할 수 있으므로, 아직 어떤 context를 써야 할지 모를 때는 `context.TODO()`, 최상위 진입점에서는 `context.Background()` 를 사용합니다. 실무에서는 HTTP handler에서 `r.Context()` 를 받아 DB 쿼리까지 전파하는 패턴을 씁니다. 클라이언트 연결이 끊기면 request context가 취소되고, 그 취소가 DB 쿼리까지 전달되어 불필요한 쿼리가 자동으로 중단됩니다.
