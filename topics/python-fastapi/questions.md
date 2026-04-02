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

---

## FastAPI Dependency Injection — Depends()

**Q. FastAPI의 `Depends()`를 활용한 DI 패턴을 설명하고, `yield`를 사용하는 이유를 설명해주세요.**

**난이도:** 기초

**핵심 키워드:** Depends(), yield, 요청 스코프, setup/teardown, DB 세션, 인증 미들웨어, @Autowired 비교

**모범 답변 방향:**

**Depends() 기본 개념:**
- 요청마다 실행되는 함수를 파라미터에 주입. 프레임워크가 호출 시점에 자동 실행.
- Spring `@Autowired`와 공통점: 프레임워크가 관리하는 컴포넌트를 주입
- 차이: `@Autowired`는 싱글톤·클래스 필드, `Depends()`는 요청 스코프·함수 파라미터

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
