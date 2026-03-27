---
tags: [golang, go, concurrency, backend]
related: [distributed-systems, kubernetes, redis, kafka-rabbitmq]
---

# Golang — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/golang/questions]]

---

## 1. Go 언어 특징

- 정적 타입, 컴파일 언어 — 빠른 실행 속도
- **동시성이 언어 레벨에서 지원** (goroutine, channel)
- GC 내장 — 메모리 관리 자동화, Stop-the-world pause 1ms 이하
- 암시적 인터페이스 구현 (Duck typing)
- 단순한 문법 — 키워드 25개

---

## 2. Goroutine

Go의 **경량 스레드**. `go` 키워드로 시작.

- OS 스레드보다 훨씬 가볍 (~2KB per goroutine, 스레드는 ~1MB)
- Go 런타임이 M:N 스케줄링 (N개 goroutine → M개 OS 스레드)
- **G-P-M 모델**: Goroutine - Processor - Machine(OS thread)

### Goroutine Leak (면접 단골)
goroutine이 종료되지 않고 메모리를 점유하는 현상.

**원인:**
```go
// 1. 아무도 읽지 않는 unbuffered channel에 전송
ch := make(chan int)
go func() { ch <- 1 }()  // 수신자 없으면 영원히 블로킹

// 2. Context 없이 무한 루프
go func() {
    for { doWork() }  // 종료 신호 없음
}()
```

**방지:**
```go
// Context로 취소 신호 전달
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            doWork()
        }
    }
}
```

**탐지:**
```go
// 1. 테스트: uber-go/goleak
func TestWorker(t *testing.T) {
    defer goleak.VerifyNone(t)  // 테스트 종료 시 남은 goroutine 있으면 실패
    // 테스트 코드
}

// 2. 운영: goroutine 수를 메트릭으로 노출
runtime.NumGoroutine()  // 현재 살아있는 goroutine 수
// Prometheus go_goroutines 메트릭 → Grafana 대시보드에서 선형 증가 감지
```

---

## 3. Channel

goroutine 간 **통신 및 동기화** 수단. "공유 메모리 대신 통신으로 동기화"

### Unbuffered vs Buffered
```go
ch1 := make(chan int)     // unbuffered: 송수신 동시에 준비돼야 함
ch2 := make(chan int, 10) // buffered: 버퍼 가득 찰 때까지 non-blocking
```

- **Unbuffered**: 강한 동기화. 송신자가 수신자를 기다림
- **Buffered**: 느슨한 결합. 버퍼가 완충재 역할

### Select
여러 channel을 동시에 기다릴 때 사용:
```go
select {
case msg := <-ch1:
    // ch1에서 수신
case ch2 <- data:
    // ch2에 송신
case <-time.After(3 * time.Second):
    // 타임아웃
case <-ctx.Done():
    // 취소
}
```

### Channel 닫기 규칙
- 송신자만 close
- 닫힌 channel에 send → panic
- 닫힌 channel에서 receive → zero value + false

---

## 4. Mutex vs Channel

| 상황 | 권장 |
|------|------|
| 공유 상태 보호 (캐시, 카운터) | `sync.Mutex` |
| goroutine 간 데이터 전달 | `channel` |
| 소유권 이전 | `channel` |
| 결과 수집 | `channel` |

실무 패턴:
```go
// Mutex: 공유 상태 보호
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

// Channel: goroutine 간 통신 (이력서의 채팅 서버 개선 사례)
// Mutex 기반 broadcast → channel 기반 per-connection queue로 전환
// → Latency 13초에서 103ms로 개선
```

---

## 5. Context

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

## 6. Interface

**암시적 구현** — 메서드만 맞으면 자동으로 인터페이스를 구현한 것으로 간주.

```go
type Writer interface {
    Write([]byte) (int, error)
}

// File, Buffer, net.Conn 등이 모두 Writer를 구현
// 선언 없이 자동으로
```

**작은 인터페이스 선호 (Interface Segregation)**:
```go
// 나쁜 예: 너무 큰 인터페이스
type Storage interface {
    Read() / Write() / Delete() / List() / ...
}

// 좋은 예: 필요한 것만
type Reader interface { Read() }
type Writer interface { Write() }
```

**Empty interface{}**:
```go
// Go 1.18+ any 키워드
func process(v any) {
    switch t := v.(type) {
    case int:    ...
    case string: ...
    }
}
```

---

## 7. 메모리 관리 & Escape Analysis

**스택 vs 힙:**
- 스택: 함수 내 지역 변수 (빠름, GC 불필요)
- 힙: 참조로 전달되거나 함수 범위를 벗어나는 변수 (GC 대상)

**Escape Analysis 확인:**
```bash
go build -gcflags="-m" ./...
# "escapes to heap" 메시지가 성능 최적화 포인트
```

**GC 특징:**
- Concurrent tri-color mark-and-sweep
- STW pause 1ms 이하
- `GOGC` 환경변수로 GC 빈도 조정 (기본 100 = 힙 2배 되면 GC)

---

## 8. 에러 핸들링

```go
// 기본 패턴
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)  // %w로 wrap
}

// errors.Is / errors.As (Go 1.13+)
if errors.Is(err, ErrNotFound) { ... }

var dbErr *DatabaseError
if errors.As(err, &dbErr) { ... }
```

**핵심 원칙:**
- 한 곳에서만 처리 (로깅 또는 반환, 둘 다 하지 말 것)
- `%w`로 wrap하여 에러 체인 유지
- 외부로 노출 시 내부 상세 정보 제거 (보안)

---

## 9. Go 1.22~1.23 주요 변경사항

**Go 1.22 (2024.03) — 실무 영향 큰 변경:**
```go
// for loop 변수 스코프 수정 (오랜 버그 해결)
for i := range 10 {
    go func() { println(i) }()  // 이제 0~9 각각 출력 (이전엔 모두 10)
}

// HTTP 라우팅 개선
mux.HandleFunc("GET /users/{id}", handler)
id := r.PathValue("id")  // 메서드 매칭 + URL 파라미터
```

**Go 1.23 (2024.09):**
- Range-over-functions 정식 지원 (Iterator 패턴)
- `go mod tidy -diff` — 변경 사항만 확인

---

## 참고 링크
- [Effective Go](https://go.dev/doc/effective_go)
- [Go 1.22 Release Notes](https://go.dev/doc/go1.22)
- [Go Concurrency Patterns — Rob Pike](https://go.dev/talks/2012/concurrency.slide)
- [uber-go/goleak](https://github.com/uber-go/goleak)
