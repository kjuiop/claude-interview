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

## G-P-M 스케줄러 상세 구조

Go 런타임은 **M:N 스케줄링** — N개 goroutine을 M개 OS 스레드에 매핑. 중간에 P(Processor)가 핵심 역할.

### 3개 구성 요소

| 요소 | 이름 | 역할 |
|---|---|---|
| **G** | Goroutine | 실행할 코드 + 스택 (~2KB, 동적 확장) |
| **P** | Processor | 논리 프로세서. `GOMAXPROCS` 개수만큼 존재. 각 P는 **Local Run Queue(LRQ)** 보유 |
| **M** | Machine(OS Thread) | 실제 CPU에서 실행. P를 획득해야만 G를 실행 가능 |

```
[G] [G] [G]      ← goroutine들 (수천~수백만 개 가능)
     ↓
[P0: LRQ=[G,G]] [P1: LRQ=[G,G]]    ← GOMAXPROCS=2
     ↓                ↓
    [M0]            [M1]            ← OS Thread
     ↓                ↓
   [CPU Core]      [CPU Core]
```

### Goroutine 상태 (3가지)

**사용자 이해 보정**: "전부 park 상태로 대기" ← 이건 절반만 맞음. 3가지 상태가 있음:

| 상태 | 이름 | 설명 |
|---|---|---|
| **Runnable** | 실행 준비 | P의 LRQ(또는 Global Run Queue)에 있음. M이 선택하길 기다리는 중. **park 아님** |
| **Executing** | 실행 중 | M 위에서 실제로 CPU를 쓰고 있음 |
| **Waiting (parked)** | 대기 | channel, syscall, timer 등 외부 이벤트를 기다림. `gopark()` 호출로 진입 |

```
go func() → G 생성 → [Runnable] → M이 선택 → [Executing] → ch <- 블로킹 → [Waiting/parked]
                                                                                    ↓
                                                              수신자 나타남 → [Runnable] → [Executing]
```

**park vs sleep 차이**: `park`는 Go 런타임 수준의 상태 전환 (`gopark()` / `goready()`). OS sleep이 아님 — M은 다른 G를 계속 실행할 수 있음.

### Global Run Queue (GRQ) 상세

LRQ는 **각 P 전용** 큐고, GRQ는 **모든 P가 공유**하는 큐.

| | LRQ | GRQ |
|---|---|---|
| 소유 | P 1개 전용 | 전체 런타임 공유 |
| 크기 | **256개 고정** (순환 큐) | 제한 없음 |
| 접근 속도 | 빠름 (lock 없음) | 느림 (**mutex 보호**) |
| 우선순위 | 높음 (먼저 확인) | 낮음 |

**GRQ에 goroutine이 들어가는 경우:**
```
1. LRQ가 256개로 꽉 찼을 때 → 새 G는 GRQ로
2. LRQ가 꽉 찼을 때 LRQ의 절반을 GRQ로 이동시키고 새 G 삽입
```

**GRQ → LRQ로 꺼내오는 규칙:**
```
batch 크기 = (GRQ 크기 / P 개수) + 1
```
한 번에 조금씩 가져와 P들 간 공평하게 분배.

**1/61 규칙 (GRQ 기아 방지):**
```
P가 다음 G를 찾는 매 61번의 스케줄링 중 1번은
LRQ보다 GRQ를 먼저 확인
```
LRQ를 항상 우선하면 GRQ의 G가 영원히 실행 안 될 수 있음 → 이를 방지.

### P가 다음 G를 찾는 전체 순서

```
① 61번 중 1번: GRQ 먼저 확인  (기아 방지)
② LRQ에서 꺼냄                (가장 빠름, lock 없음)
③ GRQ에서 batch로 가져옴      (mutex lock 필요)
④ 다른 P의 LRQ에서 절반 훔침  (Work Stealing)
⑤ network poller 확인         (I/O 완료된 G)
```

### Work Stealing (작업 훔치기)

P의 LRQ가 비면 다른 P에서 goroutine을 가져옴:
```
P0: LRQ=[]   →  P1: LRQ=[G,G,G,G] 에서 절반 훔침  →  P0: LRQ=[G,G]
```

- GRQ에서도 가져올 수 있음
- CPU 코어를 놀리지 않고 최대한 활용하기 위한 메커니즘

### 시스템 콜 시 P 핸드오프

goroutine이 blocking syscall(파일 I/O 등)을 호출하면:
```
G0이 syscall 진입
→ M0이 syscall로 블로킹
→ P를 M1에 넘김 (Hand Off)
→ M1이 P로 다른 goroutine 계속 실행
→ syscall 완료 → G0이 Runnable 상태로 큐에 복귀
```

이 덕분에 syscall 동안에도 다른 goroutine들이 멈추지 않음.

### GOMAXPROCS

```go
runtime.GOMAXPROCS(4)  // P 개수 = 4 → 동시에 4개 M이 실행 가능
```

- 기본값: CPU 코어 수
- P 수 = 동시 실행 가능한 goroutine 수 (진짜 병렬)

> 출처: https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html | https://dev.to/aceld/understanding-the-golang-goroutine-scheduler-gpm-model-4l1g

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

**Q. Go의 G-P-M 스케줄러 구조를 설명해주세요.**
- G(Goroutine): ~2KB 경량 스레드, 수백만 개 생성 가능
- P(Processor): 논리 프로세서. GOMAXPROCS 개수. 각 P는 Local Run Queue 보유
- M(Machine): OS 스레드. P를 획득해야만 G 실행 가능
- Goroutine 상태 3가지: **Runnable**(LRQ 대기), **Executing**(CPU 실행 중), **Waiting/parked**(channel/syscall 대기)
- Work Stealing: P의 LRQ가 비면 다른 P에서 goroutine 훔침 → CPU 낭비 방지
- Syscall 시 P 핸드오프: blocking syscall 진입 시 P를 다른 M에 넘겨 나머지 goroutine 계속 실행

**⚠️ 자주 나오는 오개념**: "모든 goroutine이 park 상태로 대기한다" → Runnable(LRQ 대기)과 Waiting(park)은 다름. park는 외부 이벤트를 기다리는 상태.

---

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
