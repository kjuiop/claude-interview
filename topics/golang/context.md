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

---

## 면접 질문

**Q. Context를 왜 사용하고, 어떻게 올바르게 전달하나요?**
- 목적: goroutine 취소 전파, 타임아웃/데드라인, request-scoped 값 전달
- 올바른 사용: 함수의 첫 번째 파라미터로 전달, struct 필드로 저장 금지
- 부모 context 취소 시 자식 goroutine 전체 자동 취소
- 실무: HTTP handler에서 `r.Context()` 받아서 DB 쿼리까지 전파 → 클라이언트 연결 끊기면 쿼리도 취소
