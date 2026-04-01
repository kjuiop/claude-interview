---
tags: [kotlin, coroutines, jvm, interview-questions]
related: [java]
---

# Kotlin — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/kotlin/concepts]]

---

## Coroutine vs Java Thread, suspend fun, launch vs async

**난이도**: 중급

**핵심 키워드**: suspend fun, Continuation, CPS, Dispatcher, launch, async, Deferred, CoroutineScope, structured concurrency

**모범 답변 방향**:

**suspend fun 내부 동작:**
- 컴파일러가 **Continuation Passing Style(CPS)** 로 State Machine으로 변환
- 각 suspension 지점에서 현재 상태를 heap에 저장 → 스레드 해제 → 작업 완료 시 `resume()` 호출 → 저장된 상태에서 재개
- 스레드를 블로킹하지 않고 중단/재개 가능한 이유

**Coroutine vs Java Thread:**
- Java Thread: 1MB 스택, blocking 모델, I/O 대기 중 스레드 점유
- Kotlin Coroutine: Continuation(State Machine) 기반, Dispatcher 스레드 풀에 스케줄링
- Dispatcher 타입:
  - `Dispatchers.Default`: CPU-bound, CoreCount 스레드 풀
  - `Dispatchers.IO`: I/O-bound, 최대 64 스레드 풀 (확장 가능)
  - `Dispatchers.Main`: Android UI 스레드

**launch vs async:**
```kotlin
launch { }   // Job 반환 — 결과값 없음, fire-and-forget
async { }    // Deferred<T> 반환 — .await()로 결과 수신, 병렬 실행 후 값 합산 시 사용
```

**CoroutineScope:**
- Coroutine의 lifecycle 범위 정의 (structured concurrency)
- scope가 cancel되면 하위 모든 coroutine 자동 취소

**Go goroutine 비교:**
- 유사점: 둘 다 non-blocking, 경량 (goroutine 2KB)
- 차이: goroutine = GMP 런타임이 OS Thread 직접 관리 / Coroutine = Continuation을 Dispatcher 스레드 풀에 스케줄링
- **Kotlin Coroutine은 싱글스레드 기반이 아님** (Python asyncio·JS 이벤트 루프와 혼동 주의)

**꼬리 질문 예시:**
- "suspend fun을 일반 함수에서 호출할 수 있나요?"
- "GlobalScope 사용을 피해야 하는 이유는?"
- "launch와 async를 언제 선택하나요?"

**면접 세션 피드백 (2026-04-01 3회차)**:
- GMP 모델 설명 정확. non-blocking 특성 맞음.
- 오개념: "Coroutine = 싱글스레드" → Dispatcher에 따라 멀티스레드 동작
- suspend fun 모름 → Continuation/CPS 변환 구조 암기 필요
- launch(Job, 결과 없음) vs async(Deferred, .await() 필요) 구분 미인지

---

## 작성 예정

주요 주제:
- Kotlin Coroutine vs Java Thread vs Java Virtual Thread
- Null safety 처리 전략
- Data class vs POJO 차이
- Coroutine Flow vs RxJava
- inline 함수의 reified 타입 파라미터
