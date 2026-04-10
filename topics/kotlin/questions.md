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

**모범 답변 (3분 이상 말하기 형태)**:

> Kotlin Coroutine은 Java Thread와 근본적으로 다른 방식으로 동시성을 처리합니다. 핵심은 `suspend fun`이 컴파일 단계에서 Continuation Passing Style, 즉 CPS 방식의 State Machine으로 변환된다는 점입니다. 각 suspension 지점, 즉 `suspend fun`을 호출하는 지점에서 현재 실행 상태를 heap에 저장하고 스레드를 해제합니다. 그리고 작업이 완료되면 `resume()`이 호출되어 저장된 상태에서 이어서 실행됩니다. 이 메커니즘 덕분에 스레드를 블로킹하지 않고도 중단과 재개가 가능합니다.
>
> Java Thread와 비교하면 차이가 명확합니다. Java Thread는 하나당 약 1MB의 스택 메모리를 차지하고, I/O 대기 중에도 스레드를 점유하는 blocking 모델입니다. 반면 Kotlin Coroutine은 Continuation 객체가 heap에 저장되고, Dispatcher가 관리하는 스레드 풀에 스케줄링됩니다. 수만 개의 Coroutine을 생성해도 스레드 수는 고정 풀 크기에 한정되기 때문에 메모리 효율이 훨씬 높습니다.
>
> Dispatcher 타입에 따라 동작 방식이 달라집니다. `Dispatchers.Default`는 CPU-bound 작업용으로 CPU 코어 수만큼의 스레드 풀을 사용합니다. 이 풀에서 하나라도 블로킹 작업이 실행되면 스레드 기아가 발생할 수 있기 때문에 블로킹 I/O를 섞어서는 안 됩니다. `Dispatchers.IO`는 I/O-bound 작업용으로 최대 `max(64, CPU 코어 수)`까지 스레드를 확장할 수 있고, `Dispatchers.Main`은 Android UI 스레드 1개에 바인딩됩니다.
>
> `launch`와 `async`의 선택 기준도 중요합니다. `launch`는 `Job`을 반환하고 결과값이 없는 fire-and-forget 방식입니다. `async`는 `Deferred<T>`를 반환하고 `.await()`로 결과를 수신할 수 있어 병렬 실행 후 결과를 합산해야 할 때 씁니다.
>
> CoroutineScope는 Coroutine의 lifecycle 범위를 정의하는 개념으로 Structured Concurrency의 핵심입니다. scope가 cancel되면 하위의 모든 Coroutine이 자동으로 취소됩니다. 이 때문에 `GlobalScope`는 애플리케이션 전체 수명을 따르기 때문에 메모리 누수나 예외 추적이 어려워 지양합니다.
>
> Go goroutine과 비교하면 유사점과 차이점이 모두 있습니다. 둘 다 non-blocking이고 경량이라는 점에서 비슷하지만, goroutine은 GMP 런타임이 OS Thread를 직접 관리하는 방식이고, Kotlin Coroutine은 Continuation을 Dispatcher 스레드 풀에 스케줄링합니다. 중요한 점은 Kotlin Coroutine은 싱글 스레드 기반이 아니라는 것입니다. Python asyncio나 JavaScript 이벤트 루프와 혼동하기 쉬운데, Dispatcher 설정에 따라 멀티 스레드로 동작합니다.

**꼬리 질문 예시:**
- "suspend fun을 일반 함수에서 호출할 수 있나요?"
- "GlobalScope 사용을 피해야 하는 이유는?"
- "launch와 async를 언제 선택하나요?"

**면접 세션 피드백 (2026-04-01 3회차)**:
- GMP 모델 설명 정확. non-blocking 특성 맞음.
- 오개념: "Coroutine = 싱글스레드" → Dispatcher에 따라 멀티스레드 동작
- suspend fun 모름 → Continuation/CPS 변환 구조 암기 필요
- launch(Job, 결과 없음) vs async(Deferred, .await() 필요) 구분 미인지

**면접 세션 피드백 (2026-04-02 1회차 — Dispatcher 심화)**:
- 잘한 점: Default(CPU코어수), IO(I/O용 대형 풀), Main(UI, 1개) 역할 구분 정확
- 보완:
  - IO 스레드 수 수정: "36개" → `max(64, CPU 코어 수)` 가 정확한 기본값
  - "이벤트 루프" 표현 지양: Kotlin Coroutine은 스레드 풀 기반 (Node.js/Python asyncio와 다름)
  - Unconfined: "스레드 없음"이 아니라 "시작 스레드에서 실행, 첫 suspend point 이후 재개 스레드에서 계속 → 예측 불가 → 비권장"
  - Default 블로킹 위험 구체화: 스레드 수 = CPU코어 수 → 하나라도 블로킹되면 스레드 기아 발생

---

## 작성 예정

주요 주제:
- Kotlin Coroutine vs Java Thread vs Java Virtual Thread
- Null safety 처리 전략
- Data class vs POJO 차이
- Coroutine Flow vs RxJava
- inline 함수의 reified 타입 파라미터
