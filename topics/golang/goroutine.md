---
tags: [golang, go, goroutine, concurrency, scheduler]
related: [channel, context, concurrency]
---

# Golang — Goroutine

→ [[home]] | [[topics/golang/channel]] | [[topics/golang/context]] | [[topics/golang/concurrency]]

---

## 개념

Go의 **경량 스레드**. `go` 키워드로 시작.

- OS 스레드보다 훨씬 가볍 (~2KB per goroutine, 스레드는 ~1MB)
- Go 런타임이 M:N 스케줄링 (N개 goroutine → M개 OS 스레드)
- **G-P-M 모델**: Goroutine - Processor - Machine(OS thread)

---

## Goroutine Leak (면접 단골)

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

## 면접 질문

**Q. goroutine과 OS 스레드의 차이는?**
- OS 스레드: ~1MB 스택, OS가 스케줄링, 컨텍스트 스위칭 비용 큼
- goroutine: ~2KB 스택 (동적 확장), Go 런타임이 스케줄링, 수백만 개 생성 가능
- G-P-M 모델: N개 goroutine을 M개 OS 스레드에 M:N 매핑

---

**Q. goroutine leak이 발생하는 상황과 방지 방법은?**
- 원인: 아무도 읽지 않는 channel에 전송, 종료 신호 없는 무한 루프, context 미전파
- 방지: context.Context로 취소 신호 전달, select + ctx.Done() 패턴
- 탐지: `runtime.NumGoroutine()` 모니터링, `uber-go/goleak` 테스트

**모범 답변 (상세):**

> goroutine leak은 생성된 goroutine이 종료되지 않고 메모리를 계속 점유하는 현상입니다.
>
> 발생 원인은 크게 세 가지입니다.
> 첫째, **아무도 읽지 않는 channel에 전송하는 경우**입니다. unbuffered channel에 데이터를 보내는 goroutine은 수신자가 나타날 때까지 영원히 블로킹됩니다.
> 둘째, **종료 신호 없는 무한 루프**입니다. `for {}` 로 반복하는 goroutine에 멈추라는 신호를 주지 않으면 프로그램이 살아있는 한 계속 동작합니다.
> 셋째, **context 미전파**입니다. 부모 context가 취소됐는데 자식 goroutine이 해당 context를 받지 않으면 취소 신호가 전달되지 않습니다.
>
> 방지 방법은 goroutine을 생성할 때 반드시 종료 시점을 설계하는 것입니다.
>
> ```go
> func worker(ctx context.Context, ch <-chan Message) {
>     for {
>         select {
>         case msg := <-ch:
>             process(msg)
>         case <-ctx.Done():
>             return  // 취소 신호 수신 → 종료
>         }
>     }
> }
> ```
>
> 탐지: `uber-go/goleak` (테스트), `runtime.NumGoroutine()` + Prometheus (운영)

**꼬리 질문: goroutine leak을 테스트에서 검증하는 방법과, 운영 중 의심할 수 있는 징후는?**
- 테스트: `uber-go/goleak` — `defer goleak.VerifyNone(t)` 로 테스트 종료 시 잔존 goroutine 감지
- 운영 징후: Grafana에서 메모리 사용량 선형 증가 + goroutine 수(`go_goroutines`) 지속 증가
- 추가 징후: GC가 자주 돌아도 메모리가 회수되지 않는 패턴, 응답 지연 점진적 증가

---

## 참고 링크
- [Go Concurrency Patterns — Rob Pike](https://go.dev/talks/2012/concurrency.slide)
- [uber-go/goleak](https://github.com/uber-go/goleak)
