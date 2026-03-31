---
tags: [python, fastapi, asyncio, interview]
related: [golang]
---

# Python FastAPI — 면접 질문

→ [[home]] | [[topics/python-fastapi/concepts]]

---

## async def vs def

**Q. FastAPI에서 `async def`와 `def`의 차이는 무엇인가요?**

**난이도:** 기초

**핵심 키워드:** 이벤트 루프, blocking, coroutine, ThreadPoolExecutor

**모범 답변 방향:**
- `def`: 동기 함수, FastAPI가 ThreadPoolExecutor에서 실행하여 이벤트 루프 블로킹 방지
- `async def`: 코루틴, 이벤트 루프에서 직접 실행, `await`로 I/O 대기 중 제어권 반환
- DB 쿼리·HTTP 호출 등 I/O가 있으면 `async def` + 비동기 드라이버 사용

**꼬리 질문 예시:**
- `async def` 안에서 동기 함수를 직접 호출하면 어떻게 되나요? (이벤트 루프 블로킹)
- 비동기 DB 드라이버(asyncpg, motor)를 사용해야 하는 이유는?

---

## asyncio vs goroutine

**Q. Python asyncio와 Go goroutine 기반 동시성 처리 방식의 차이를 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** 싱글 스레드 이벤트 루프, GMP 모델, M:N 스케줄링, CPU-bound vs I/O-bound

**모범 답변 방향:**
- asyncio: 싱글 스레드 이벤트 루프 (Node.js와 동일 원리), I/O-bound에 최적, CPU-bound는 ProcessPoolExecutor 필요
- goroutine: GMP 모델 M:N 스케줄링, CPU-bound·I/O-bound 모두 효율적, 초기 스택 2KB
- 선택 기준: 워크로드 특성(I/O vs CPU) + 팀 역량

**GMP 용어 순서 주의:**
- G(Goroutine) : M(OS Thread) : P(Processor)
- "P가 G를 M에 스케줄링한다"
- M:N = M개의 goroutine을 N개의 OS thread에 매핑

**꼬리 질문 예시:**
- asyncio에서 CPU-heavy 작업을 처리하려면 어떻게 해야 하나요?
- Go의 GOMAXPROCS는 무엇이고 어떻게 설정하나요?
- goroutine이 OS 스레드보다 경량인 이유는?
