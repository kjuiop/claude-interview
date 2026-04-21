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

### Write Barrier 메커니즘

**Write Barrier가 필요한 이유:**
동시 마킹 중 goroutine이 **Black 객체에서 White 객체로 새 참조**를 만들 수 있음.
- Black은 이미 스캔 완료 → GC가 다시 확인하지 않음
- 결과: White 객체가 살아있는데 수거됨 (use-after-free 버그)

**Write Barrier 동작:**
참조가 변경될 때 White 객체를 **Gray로 재색칠** → 다시 스캔 큐에 넣음 → 실수로 수거 방지

```
goroutine: black.ref = white  // 참조 변경 발생
Write Barrier: white → Gray   // 즉시 Gray로 재색칠
GC: Gray 스캔 큐에 추가       // 다시 검사
```

**면접 세션 피드백 (2026-04-21 3회차)**:
- White 정의 오류 주의: "참조되는 자식이 없어 삭제 가능한 상태" → **White = 아직 GC에 발견되지 않은 미탐색 객체** (자식 유무와 무관)
- Write Barrier 답변 키워드: "Gray로 재색칠" 반드시 포함. "수거하지 못하게 한다"는 틀림.

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

Escape Analysis는 Go 컴파일러가 변수를 스택에 할당할지 힙에 할당할지를 컴파일 타임에 결정하는 정적 분석입니다. 스택 할당은 함수가 종료되면 프레임 전체가 자동으로 해제되어 GC 개입이 전혀 없고 할당·해제 비용이 사실상 0에 가깝습니다. 반면 힙 할당은 GC의 관리 대상이 되어 GC 실행 빈도가 높아지고 STW pause 시간에 영향을 줍니다. 변수가 힙으로 escape되는 대표적인 케이스는 네 가지입니다. 첫째, 함수 내 지역 변수의 포인터를 반환하는 경우로, 함수 스택이 사라진 뒤에도 참조가 살아있어야 하기 때문에 힙으로 이동합니다. 둘째, `interface{}` 값으로 할당되는 경우로, 컴파일러가 실제 타입 크기를 정적으로 알 수 없어 힙으로 빠집니다. 셋째, goroutine 클로저에 캡처되는 경우로, 클로저가 goroutine에서 별도로 실행되기 때문에 원본 함수의 스택과 수명이 달라지면서 힙으로 이동합니다. 넷째, 크기가 컴파일 타임에 결정되지 않는 동적 슬라이스나 맵도 힙에 할당됩니다. `go build -gcflags="-m" ./...` 명령어로 컴파일러가 어떤 변수에 대해 "escapes to heap"이라고 판단하는지 확인할 수 있습니다. 성능이 중요한 hot path에서는 이 분석을 통해 불필요한 힙 할당을 줄이는 최적화가 가능합니다. 예를 들어 인터페이스 파라미터를 concrete type으로 바꾸거나, 포인터 반환 대신 값 반환으로 전환하는 것만으로도 GC 압박을 줄일 수 있습니다. 또한 슬라이스를 make로 미리 용량을 지정해서 재할당 없이 재사용하는 패턴도 힙 escape를 줄이는 데 효과적입니다. 카테노이드에서 채팅 서버를 운영할 때 메시지 수신 핸들러의 hot path에서 `sync.Pool`로 Message 구조체를 재사용하고, 인터페이스 대신 concrete type을 사용함으로써 GC pressure를 낮춰 tail latency를 개선한 경험이 있습니다. Escape Analysis는 섣불리 최적화하기보다 pprof로 힙 할당이 집중된 곳을 먼저 측정하고, 실제 병목이 확인된 부분에만 적용하는 것이 효과적입니다. 최적화 결과는 반드시 benchmark test로 검증해야 합니다.

---

**Q. Go GC의 특징과 성능에 미치는 영향은?**

Go의 GC는 Concurrent tri-color mark-and-sweep 방식으로 동작합니다. 핵심 특징은 마킹 단계를 애플리케이션 goroutine과 동시에(Concurrent) 실행하기 때문에 STW(Stop-The-World) pause가 1ms 이하로 매우 짧다는 점입니다. STW는 루트 오브젝트 스캔 시작 시점과 최종 정리 시점에만 아주 짧게 발생합니다. 동시 마킹 중 goroutine이 포인터를 변경할 때 발생하는 일관성 문제는 Write Barrier가 해당 객체를 Gray로 재색칠해서 재스캔하도록 처리합니다. GC 트리거 조건은 힙 사용량이 이전 GC 완료 시점 대비 GOGC% 만큼 증가하면 실행됩니다. 기본값 GOGC=100은 힙이 마지막 GC 이후 두 배가 되면 실행한다는 의미입니다. 대용량 트래픽 환경에서는 GC pause 자체의 길이보다, GC가 얼마나 자주 실행되는지가 Latency spike의 주요 원인이 됩니다. GC가 실행되는 동안에는 마킹 작업이 CPU 자원을 소비하기 때문에, 초당 요청이 많은 서버에서는 GC 빈도가 높아질수록 응답 시간 변동성이 커집니다. 튜닝 방법은 크게 두 가지입니다. `GOGC=200`으로 올리면 힙이 3배가 될 때까지 GC를 미루므로 GC 빈도가 줄어들지만 메모리 사용량이 늘어납니다. Go 1.19+에서는 `GOMEMLIMIT`으로 메모리 절대 상한을 설정해서 할당량이 한계에 가까워지면 GC를 공격적으로 실행하게 함으로써 OOM 없이 메모리를 안정적으로 유지할 수 있습니다. 코드 레벨에서는 `sync.Pool`로 자주 생성·소멸되는 구조체를 재사용해 GC 대상 객체 수를 줄이는 것이 효과적입니다. 카테노이드에서 채팅 메시지 수신 핸들러에 sync.Pool을 적용하여 GC 빈도를 낮추고 평균 응답 시간의 안정성을 개선한 경험이 있습니다. 실무에서는 항상 `GODEBUG=gctrace=1`로 GC 빈도와 STW 시간을 먼저 측정하고, `pprof`의 힙 프로파일로 할당이 집중된 위치를 파악한 뒤에 최적화하는 순서를 지킵니다. GC 튜닝은 측정 없이 GOGC 값만 조정하면 메모리 증가나 OOM 같은 역효과가 생길 수 있어 신중하게 접근해야 합니다.
