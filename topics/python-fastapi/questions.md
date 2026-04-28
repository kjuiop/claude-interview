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

FastAPI에서 `def`와 `async def`의 차이는 실행 방식과 이벤트 루프 차단 여부에 있습니다. `async def`로 선언한 엔드포인트는 코루틴으로, FastAPI가 이벤트 루프에서 직접 실행하며 `await` 지점에서 I/O 대기 중에 이벤트 루프에 제어권을 반환해 다른 요청을 처리할 수 있습니다. 반면 일반 `def`로 선언한 함수는 동기 함수인데, 이를 이벤트 루프에서 직접 실행하면 함수가 완료될 때까지 이벤트 루프 전체가 블로킹됩니다. FastAPI는 이를 방지하기 위해 일반 `def` 엔드포인트를 자동으로 `ThreadPoolExecutor`에서 실행합니다. 실무 선택 기준으로는 DB 쿼리, HTTP 호출처럼 I/O 대기가 있는 작업에는 `async def`와 비동기 드라이버(asyncpg, motor, httpx)를 함께 사용해야 성능 이점을 얻을 수 있습니다. 주의할 점은 `async def` 안에서 동기 함수를 직접 호출하면 이벤트 루프가 블로킹되므로, 반드시 `asyncio.run_in_executor`로 별도 스레드에서 실행해야 합니다.

**꼬리 질문 예시:**
- `async def` 안에서 동기 함수를 직접 호출하면 어떻게 되나요? (이벤트 루프 블로킹)
- 비동기 DB 드라이버(asyncpg, motor)를 사용해야 하는 이유는?

---

## asyncio vs goroutine

**Q. Python asyncio와 Go goroutine 기반 동시성 처리 방식의 차이를 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** 싱글 스레드 이벤트 루프, GMP 모델, M:N 스케줄링, CPU-bound vs I/O-bound

**모범 답변 방향:**

Python asyncio와 Go goroutine은 동시성 처리 방식이 근본적으로 다릅니다. asyncio는 싱글 스레드 이벤트 루프 기반으로 Node.js와 동일한 원리입니다. 코루틴이 `await` 지점에서 이벤트 루프에 제어권을 반환하고, 이벤트 루프가 다음 코루틴을 실행하는 협력적 멀티태스킹 방식입니다. 싱글 스레드이기 때문에 공유 자원의 race condition이 발생하지 않는 장점이 있지만, CPU-bound 작업이 이벤트 루프를 독점하면 전체가 블로킹되어 `ProcessPoolExecutor`로 별도 프로세스에서 실행해야 합니다. Go goroutine은 GMP 모델로 M개의 goroutine을 N개의 OS 스레드에 매핑하는 M:N 스케줄링을 사용합니다. P(Processor)가 G(Goroutine)를 M(OS Thread)에 스케줄링하며 선점적으로 동작합니다. goroutine의 초기 스택은 2KB로 매우 경량이라 수십만 개를 동시에 실행해도 메모리 부담이 적습니다. CPU-bound와 I/O-bound 모두 효율적으로 처리할 수 있어 고트래픽 환경에서 강점을 보입니다. 기술 선택 기준으로는 I/O 중심 API 서버에 팀이 Python을 잘 안다면 FastAPI/asyncio, 고트래픽·CPU-bound 혼재·실시간 처리가 요구되면 Go goroutine이 적합합니다.

**GMP 용어 순서 주의:**
- G(Goroutine) : M(OS Thread) : P(Processor)
- "P가 G를 M에 스케줄링한다"
- M:N = M개의 goroutine을 N개의 OS thread에 매핑

**꼬리 질문 예시:**
- asyncio에서 CPU-heavy 작업을 처리하려면 어떻게 해야 하나요?
- Go의 GOMAXPROCS는 무엇이고 어떻게 설정하나요?
- goroutine이 OS 스레드보다 경량인 이유는?

---

## FastAPI Dependency Injection — Depends()

**Q. FastAPI의 `Depends()`를 활용한 DI 패턴을 설명하고, `yield`를 사용하는 이유를 설명해주세요.**

**난이도:** 기초

**핵심 키워드:** Depends(), yield, 요청 스코프, setup/teardown, DB 세션, 인증 미들웨어, @Autowired 비교

**모범 답변 방향:**

FastAPI의 `Depends()`는 요청마다 실행되는 함수를 라우터 파라미터에 주입하는 DI 메커니즘입니다. 프레임워크가 요청이 들어올 때마다 자동으로 해당 함수를 호출하고 결과를 주입합니다. Spring의 `@Autowired`와 비교하면, 둘 다 프레임워크가 관리하는 컴포넌트를 주입한다는 공통점이 있지만 스코프와 주입 위치가 다릅니다. `@Autowired`는 기본적으로 싱글톤이고 클래스 필드나 생성자에 주입되는 반면, `Depends()`는 기본적으로 요청(Request) 스코프여서 요청마다 새로 실행되고 함수 파라미터로 주입됩니다.

**Depends() 기본 개념 (핵심 구분):**

**yield 패턴 — DB 세션 표준 관용구:**
```python
def get_db():
    db = SessionLocal()
    try:
        yield db        # 요청 전: setup, 라우터에 db 전달
    finally:
        db.close()      # 요청 완료 후: teardown (성공/실패 무관)

@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```
> `return` 대신 `yield`: yield 이전 = setup, yield 이후 finally = cleanup. 요청이 성공하든 예외가 발생하든 `db.close()` 보장.

**인증 처리 예시:**
```python
def get_current_user(token: str = Header(...)):
    user = verify_token(token)
    if not user:
        raise HTTPException(status_code=401)
    return user

@app.get("/me")
async def me(user: User = Depends(get_current_user)):
    return user
```

**Spring @Autowired vs FastAPI Depends() 비교:**

| | Spring `@Autowired` | FastAPI `Depends()` |
|---|---|---|
| 스코프 | 기본 싱글톤 | 기본 요청(Request)마다 실행 |
| 주입 위치 | 클래스 필드/생성자 | 함수 파라미터 |
| cleanup | Bean 소멸자 | `yield` 이후 finally |
| 중첩 | Bean 간 의존 그래프 | Depends() 중첩 체이닝 가능 |

**면접 세션 피드백 (2026-04-02 4회차)**:
- 잘한 점: DI 핵심 개념("프레임워크가 관리하는 컴포넌트 주입, 자원 재사용") 정확. `Depends()`와 `@Autowired` 공통점을 추론으로 연결.
- 보완:
  - `yield` 패턴 신규 암기: `yield` 이전 = setup, 이후 finally = teardown. 요청 완료 후 `db.close()` 보장.
  - `@Autowired`(싱글톤, 클래스 필드) vs `Depends()`(요청 스코프, 함수 파라미터) 차이 명시 필요
  - 구체적 예시(`get_db`, `get_current_user`) 암기하여 면접에서 코드 수준으로 설명할 것

**면접 세션 피드백 (2026-04-28 4회차)**:
- 잘한 점: yield setup/teardown 구분, Go defer/Java finally 비유 정확. Spring 싱글톤 생성/재사용 동작 정확.
- 보완: **요청 스코프 격리 메커니즘 미언급** — FastAPI는 요청마다 Depends() 제너레이터를 새로 실행해 독립 인스턴스를 생성함. 이것이 Spring 싱글톤과의 핵심 차이. "매 요청마다 새 인스턴스 생성 → 요청 스코프" 표현 암기 필요.
- 점수: 5/10 (꼬리 질문 "모르겠습니다")
