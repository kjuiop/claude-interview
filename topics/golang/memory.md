---
tags: [golang, go, memory, gc, escape-analysis, performance]
related: [goroutine, interface]
---

# Golang — 메모리 관리 & GC

→ [[home]] | [[topics/golang/goroutine]] | [[topics/golang/interface]]

---

## 스택 vs 힙

- 스택: 함수 내 지역 변수 (빠름, GC 불필요)
- 힙: 참조로 전달되거나 함수 범위를 벗어나는 변수 (GC 대상)

## Escape Analysis

컴파일러가 변수를 스택에 할당할지 힙에 할당할지 결정하는 분석.

**확인 방법:**
```bash
go build -gcflags="-m" ./...
# "escapes to heap" 메시지가 성능 최적화 포인트
```

**힙으로 escape되는 주요 케이스:**
- 포인터를 반환하는 경우
- 인터페이스 값으로 할당되는 경우
- goroutine 클로저에 캡처되는 경우
- 크기를 알 수 없는 경우 (동적 slice 등)

---

## Tricolor Marking 알고리즘

Go GC는 **Concurrent tri-color mark-and-sweep** 방식으로 동작한다.

### 색상 의미

| 색상 | 의미 |
|---|---|
| **White** | 아직 탐색하지 않은 객체. GC 후에도 White면 참조되지 않는 **수거 대상** |
| **Gray** | 탐색 시작했지만 자식 객체를 아직 다 확인하지 못한 상태 |
| **Black** | 자신과 모든 참조 객체 확인 완료. **살아있는 객체** |

### 알고리즘 진행 순서

1. 루트 객체(전역변수, 스택 변수)를 모두 **Gray**로 표시 ← STW (짧음)
2. Gray 객체를 꺼내 → 자식을 Gray로, 자신은 Black으로
3. Gray가 없을 때까지 반복 (애플리케이션과 **Concurrent**하게 진행)
4. 남은 White 객체 수거 ← STW (짧음)

### STW가 짧은 이유

- marking 단계를 애플리케이션 goroutine과 **동시에(Concurrent)** 실행
- STW는 **루트 스캔 시작**과 **최종 정리** 순간에만 발생
- Concurrent marking 중 포인터 변경 → **Write Barrier**로 추적

### GC 문제 확인 방법

```bash
# GC 빈도, STW 시간 실시간 로그 출력
GODEBUG=gctrace=1 ./your-service

# heap 프로파일로 allocation 집중 위치 확인
go tool pprof http://localhost:6060/debug/pprof/heap
```

- `gctrace=1`: `gc 1 @0.5s 2%: 0.1+1.2+0.3 ms clock, 힙 증감` 형태로 출력
- 먼저 측정(gctrace → pprof) → 원인 파악 → 최적화 순서로 접근

### GC pressure 최적화 패턴

```go
// sync.Pool로 구조체 재사용 (채팅 메시지 버퍼 등)
var msgPool = sync.Pool{
    New: func() any { return &Message{} },
}

func handleMessage() {
    msg := msgPool.Get().(*Message)
    defer msgPool.Put(msg)
    // 사용
}
```

- `sync.Pool`: heap 할당 횟수 감소 → GC pressure 완화
- 인터페이스 대신 **구체 타입** 유지 → heap escape 방지
- 슬라이스 미리 할당 재사용 → goroutine마다 새 slice 생성 방지

## GC 특징

- Concurrent tri-color mark-and-sweep
- STW pause 1ms 이하
- `GOGC` 환경변수로 GC 빈도 조정 (기본 100 = 힙 2배 되면 GC)

**튜닝 옵션:**
```bash
GOGC=200          # GC 빈도 줄이기 (메모리 더 쓰고 CPU 절약)
GOMEMLIMIT=4GiB   # 메모리 상한 설정 (Go 1.19+)
```

---

## 면접 질문

**Q. Escape Analysis란 무엇이고 왜 중요한가요?**

Escape Analysis는 Go 컴파일러가 변수를 스택에 할당할지 힙에 할당할지를 결정하는 정적 분석입니다. 스택 할당은 함수가 종료되면 자동으로 해제되어 GC 부담이 없고 매우 빠릅니다. 반면 힙 할당은 GC의 관리 대상이 되어 GC 실행 빈도와 STW 시간에 영향을 줍니다. 변수가 힙으로 escape되는 대표적인 케이스는 포인터를 반환하는 경우, 인터페이스 값으로 할당되는 경우, goroutine 클로저에 캡처되는 경우입니다. `go build -gcflags="-m" ./...` 명령어로 컴파일러가 "escapes to heap"이라고 출력하는 변수들을 확인할 수 있습니다. 성능이 중요한 hot path에서는 이 분석을 통해 불필요한 힙 할당을 줄이는 최적화가 가능합니다. 예를 들어 인터페이스 파라미터를 concrete type으로 바꾸거나, 포인터 반환 대신 값 반환으로 전환하는 것만으로도 GC 압박을 줄일 수 있습니다.

---

**Q. Go GC의 특징과 성능에 미치는 영향은?**

Go의 GC는 Concurrent tri-color mark-and-sweep 방식으로 동작합니다. 특징은 STW(Stop-The-World) pause가 1ms 이하로 매우 짧다는 점입니다. 힙 사용량이 이전 GC 이후 GOGC% 만큼 증가하면 GC가 트리거됩니다. 기본값 GOGC=100은 힙이 두 배가 되면 실행한다는 의미입니다. 대용량 트래픽 환경에서는 GC pause 자체보다 GC가 실행되는 빈도가 Latency spike의 원인이 될 수 있습니다. 튜닝 방법으로는 `GOGC=200`으로 올려 GC 빈도를 줄이되 메모리를 더 사용하거나, Go 1.19+에서 `GOMEMLIMIT`으로 메모리 상한을 설정해 OOM을 방지하는 방법이 있습니다. 실무에서는 `pprof`로 힙 프로파일링을 해서 어떤 부분에서 할당이 많이 일어나는지 먼저 파악하고 최적화합니다. GC 튜닝은 측정 없이 섣불리 하면 역효과가 날 수 있습니다.
