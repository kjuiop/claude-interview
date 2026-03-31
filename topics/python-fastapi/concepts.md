---
tags: [python, fastapi, asyncio, async, concurrency]
related: [golang]
---

# Python FastAPI — 핵심 개념

→ [[home]] | [[topics/python-fastapi/questions]]

---

## async def vs def

| | `def` | `async def` |
|---|---|---|
| 실행 방식 | 동기(blocking) | 비동기(non-blocking) |
| I/O 대기 중 | 스레드 점유 | 이벤트 루프에 제어권 반환 |
| 사용 시점 | CPU 작업, 빠른 연산 | DB 쿼리, HTTP 호출 등 I/O 대기 |
| FastAPI 처리 | ThreadPoolExecutor로 실행 | 이벤트 루프에서 직접 실행 |

**FastAPI의 동작 방식:**
- `async def` 엔드포인트 → 이벤트 루프에서 직접 실행 (블로킹 없음)
- 일반 `def` 엔드포인트 → FastAPI가 자동으로 `ThreadPoolExecutor`에서 실행 (이벤트 루프 블로킹 방지)

---

## asyncio 기반 비동기 처리

Python asyncio는 **싱글 스레드 이벤트 루프** 기반. Node.js 이벤트 루프와 동일한 원리.

**동작 원리:**
1. 이벤트 루프가 코루틴 실행
2. I/O 대기(`await`) 발생 시 → 이벤트 루프에 제어권 반환
3. 이벤트 루프가 다른 코루틴 실행
4. I/O 완료 시 → 원래 코루틴 재개

**장점:** 싱글 스레드이므로 공유 자원 동시성 문제 없음 (race condition 발생 안 함)

**단점:** CPU-bound 작업에 부적합 — 하나의 코루틴이 CPU를 오래 점유하면 이벤트 루프 전체 블로킹

**CPU-bound 해결책:** `ProcessPoolExecutor`로 별도 프로세스에서 실행
```python
from concurrent.futures import ProcessPoolExecutor
import asyncio

async def handle():
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy_task)
```

---

## asyncio vs goroutine 비교

| 항목 | Python asyncio | Go goroutine |
|---|---|---|
| 스레드 모델 | 싱글 스레드 이벤트 루프 | M:N (M goroutine : N OS thread) |
| 스케줄러 | 이벤트 루프 (협력적) | GMP 런타임 (선점적) |
| CPU-bound | 취약 (ProcessPoolExecutor 필요) | 강함 (멀티코어 활용) |
| I/O-bound | 강함 (이벤트 루프 최적화) | 강함 (goroutine 대기 중 다른 작업) |
| 메모리 | 코루틴 경량 | goroutine 초기 스택 2KB |
| 동시성 문제 | 싱글 스레드라 race condition 없음 | Mutex, Channel로 직접 제어 |

**Go GMP 모델:**
- **G** (Goroutine): 경량 스레드, 초기 스택 2KB
- **M** (OS Thread): 실제 CPU에서 실행되는 스레드
- **P** (Processor): GOMAXPROCS로 설정 (기본: CPU 코어 수), G를 M에 스케줄링

**선택 기준:**
- I/O 중심 API + 팀이 Python 익숙 → FastAPI/asyncio
- 고트래픽·CPU-bound 혼재·실시간 처리 → Go goroutine
