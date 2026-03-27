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
- 컴파일러가 변수를 스택에 할당할지 힙에 할당할지 결정하는 분석
- 스택 할당: 빠름, GC 불필요. 힙 할당: 느림, GC 대상
- 확인: `go build -gcflags="-m" ./...` → "escapes to heap" 출력
- 최적화: 포인터 반환 최소화, 인터페이스 사용 줄이기 (인터페이스 값은 힙으로 이스케이프)

---

**Q. Go GC의 특징과 성능에 미치는 영향은?**
- Concurrent tri-color mark-and-sweep — STW pause 1ms 이하
- 힙이 GOGC% 증가하면 GC 트리거 (기본 100 = 2배 되면 실행)
- 튜닝: `GOGC=200` (GC 빈도 줄이기), `GOMEMLIMIT` (메모리 상한 설정, Go 1.19+)
- 대용량 트래픽에서 GC pause가 Latency spike 유발 가능 → pprof로 분석
