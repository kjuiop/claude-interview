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

### 💬 면접 답변 형태로 읽기

goroutine은 Go 런타임이 관리하는 경량 스레드입니다. OS 스레드가 보통 1MB의 고정 스택을 갖는 것에 비해, goroutine은 약 2KB에서 시작해 필요에 따라 동적으로 확장됩니다. 이 덕분에 수백만 개의 goroutine을 동시에 생성해도 메모리 부담이 크지 않습니다.

Go의 스케줄링 모델은 G-P-M이라는 세 가지 구성 요소로 이루어집니다. G는 Goroutine으로 실제 실행할 코드와 스택을 담은 단위입니다. P는 Processor로 GOMAXPROCS 개수만큼 존재하는 논리 프로세서이며, 각 P는 Local Run Queue를 보유해 lock 없이 goroutine을 빠르게 꺼낼 수 있습니다. M은 OS 스레드로, 반드시 P를 획득해야만 goroutine을 실행할 수 있습니다. 이 구조에서 N개의 goroutine이 M개의 OS 스레드에 매핑되는 M:N 스케줄링이 이루어집니다.

이 모델의 핵심 메커니즘이 두 가지 있습니다. 첫 번째는 Work Stealing으로, 한 P의 LRQ가 비면 다른 P의 LRQ에서 goroutine을 절반 훔쳐와 CPU 코어를 놀리지 않고 최대한 활용합니다. 두 번째는 syscall 시 P 핸드오프입니다. goroutine이 파일 I/O 같은 blocking syscall에 진입하면 해당 OS 스레드는 블로킹되지만, P는 즉시 다른 M에 넘겨져 나머지 goroutine들이 멈추지 않고 계속 실행됩니다. 이 두 메커니즘 덕분에 Go는 I/O 집약적인 서버 워크로드에서도 높은 처리량을 달성합니다.

goroutine 상태에 대한 흔한 오개념이 있습니다. "모든 goroutine이 park 상태로 대기한다"는 표현은 정확하지 않습니다. goroutine의 상태는 LRQ에서 실행을 기다리는 Runnable, 실제로 CPU를 사용하는 Executing, 그리고 channel이나 syscall 완료를 기다리는 Waiting(parked) 세 가지로 나뉩니다. park는 Waiting 상태에 해당하는 것으로, OS sleep이 아닌 Go 런타임 수준의 상태 전환입니다. park된 goroutine의 M은 다른 G를 계속 실행할 수 있습니다.

goroutine leak은 생성된 goroutine이 종료되지 않고 메모리를 점유하는 현상으로, 실무에서 특히 주의해야 합니다. 주요 원인은 아무도 읽지 않는 channel에 전송하거나, 종료 신호 없는 무한 루프, context 미전파입니다. 방지 방법은 goroutine을 생성할 때 반드시 종료 시점을 설계하는 것으로, `context.Context`를 전달하고 `case <-ctx.Done(): return` 패턴으로 취소 신호를 수신해 종료합니다. 탐지는 테스트 단계에서 `uber-go/goleak`, 운영에서는 `runtime.NumGoroutine()` 메트릭이 선형 증가하는 패턴을 Grafana로 모니터링합니다.

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

Go의 G-P-M 스케줄러는 수많은 goroutine을 적은 수의 OS 스레드에 효율적으로 매핑하기 위한 M:N 스케줄링 모델입니다. G는 Goroutine으로 약 2KB의 경량 스택을 가진 실행 단위입니다. OS 스레드가 보통 1MB 스택을 쓰는 것에 비하면 수백 배 작기 때문에 수백만 개도 생성할 수 있습니다. P는 Processor로, GOMAXPROCS 개수만큼 존재하는 논리 프로세서입니다. 각 P는 Local Run Queue를 보유하고 있어 goroutine을 lock 없이 빠르게 꺼낼 수 있습니다. M은 OS 스레드로, 반드시 P를 획득해야만 goroutine을 실행할 수 있습니다. 여기서 중요한 두 가지 메커니즘이 있습니다. 첫 번째는 Work Stealing입니다. P의 LRQ가 비면 다른 P의 LRQ에서 goroutine을 절반 훔쳐와서 CPU 코어를 최대한 활용합니다. 두 번째는 blocking syscall 시 P 핸드오프입니다. goroutine이 파일 I/O 같은 blocking syscall에 진입하면 그 OS 스레드는 블로킹되지만, P는 즉시 다른 M에 넘겨져서 나머지 goroutine들이 멈추지 않고 계속 실행됩니다. 한 가지 자주 나오는 오개념이 있는데, "모든 goroutine이 park 상태로 대기한다"는 표현은 정확하지 않습니다. goroutine 상태는 Runnable(LRQ 대기), Executing(CPU 실행 중), Waiting/parked(채널이나 syscall 대기) 세 가지이고, Runnable과 park는 엄연히 다릅니다.

**⚠️ 자주 나오는 오개념**: "모든 goroutine이 park 상태로 대기한다" → Runnable(LRQ 대기)과 Waiting(park)은 다름. park는 외부 이벤트를 기다리는 상태.

---

**Q. goroutine과 OS 스레드의 차이는?**

goroutine과 OS 스레드의 근본적인 차이는 스케줄링 주체와 비용입니다. OS 스레드는 커널이 직접 스케줄링하며 스택 크기가 약 1MB로 고정되어 있고, 컨텍스트 스위칭 시 커널 모드 전환이 발생해 비용이 큽니다. 반면 goroutine은 Go 런타임이 유저 스페이스에서 스케줄링하며, 초기 스택이 약 2KB에서 시작해 필요에 따라 동적으로 확장됩니다. 이 덕분에 수백만 개의 goroutine을 동시에 띄워도 메모리 부담이 적고, 컨텍스트 스위칭도 런타임 수준에서 처리되어 훨씬 가볍습니다. Go의 G-P-M 모델은 N개의 goroutine을 M개의 OS 스레드에 매핑해서 이 둘의 장점을 결합한 구조입니다.

---

**Q. goroutine leak이 발생하는 상황과 방지 방법은?**

goroutine leak은 생성된 goroutine이 종료되지 않고 메모리를 계속 점유하는 현상입니다. 발생 원인은 크게 세 가지입니다. 첫째, 아무도 읽지 않는 channel에 전송하는 경우입니다. unbuffered channel에 데이터를 보내는 goroutine은 수신자가 나타날 때까지 영원히 블로킹됩니다. 둘째, 종료 신호 없는 무한 루프입니다. `for {}` 로 반복하는 goroutine에 멈추라는 신호를 주지 않으면 프로그램이 살아있는 한 계속 동작합니다. 셋째, context 미전파입니다. 부모 context가 취소됐는데 자식 goroutine이 해당 context를 받지 않으면 취소 신호가 전달되지 않습니다. 방지 방법은 goroutine을 생성할 때 반드시 종료 시점을 설계하는 것입니다. 구체적으로는 `context.Context` 를 goroutine에 전달하고, 내부 select 루프에서 `case <-ctx.Done(): return` 패턴으로 취소 신호를 수신해 종료합니다.

```go
func worker(ctx context.Context, ch <-chan Message) {
    for {
        select {
        case msg := <-ch:
            process(msg)
        case <-ctx.Done():
            return  // 취소 신호 수신 → 종료
        }
    }
}
```

탐지 측면에서는 테스트 단계에서 `uber-go/goleak` 라이브러리를 활용해 `defer goleak.VerifyNone(t)` 를 붙이면, 테스트 종료 후에도 살아있는 goroutine이 있으면 자동으로 실패 처리됩니다. 운영 환경에서는 `runtime.NumGoroutine()` 을 Prometheus 메트릭으로 노출하고, Grafana 대시보드에서 goroutine 수가 선형적으로 증가하는 패턴이 보이면 leak을 의심합니다.

**꼬리 질문: goroutine leak을 테스트에서 검증하는 방법과, 운영 중 의심할 수 있는 징후는?**
- 테스트: `uber-go/goleak` — `defer goleak.VerifyNone(t)` 로 테스트 종료 시 잔존 goroutine 감지
- 운영 징후: Grafana에서 메모리 사용량 선형 증가 + goroutine 수(`go_goroutines`) 지속 증가
- 추가 징후: GC가 자주 돌아도 메모리가 회수되지 않는 패턴, 응답 지연 점진적 증가

---

## 참고 링크
- [Go Concurrency Patterns — Rob Pike](https://go.dev/talks/2012/concurrency.slide)
- [uber-go/goleak](https://github.com/uber-go/goleak)
