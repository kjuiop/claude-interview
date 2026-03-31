---
tags: [java, jvm, spring, interview-questions]
related: [kotlin, distributed-systems]
---

# Java — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/java/concepts]]

---

## AOP란 무엇이고 OOP와 어떻게 다른가요?

**난이도:** 기초

**핵심 키워드:** 횡단 관심사, Cross-Cutting Concern, 모듈화

**모범 답변 방향:**
- OOP는 비즈니스 로직을 객체로 분리하지만, 로깅·트랜잭션·보안처럼 여러 객체에 걸친 공통 관심사는 분리하기 어렵다
- AOP는 이런 **횡단 관심사(Cross-Cutting Concern)** 를 별도 모듈(Aspect)로 분리해 핵심 로직에서 제거
- 결과: 비즈니스 코드가 순수해지고, 공통 기능은 한 곳에서 관리

**꼬리 질문 예시:**
- 횡단 관심사의 실제 예시를 들어주세요.
- Spring에서 AOP는 어떤 방식으로 구현되어 있나요?

> 출처: https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html

---

## Aspect, Advice, Pointcut, JoinPoint를 각각 설명해주세요.

**난이도:** 기초

**핵심 키워드:** Aspect, Advice, Pointcut, JoinPoint, Weaving

**모범 답변 방향:**
- **JoinPoint**: AOP를 적용할 수 있는 모든 지점. Spring AOP는 메서드 실행 시점만 지원
- **Pointcut**: JoinPoint 중 Advice를 적용할 대상을 선별하는 표현식
- **Advice**: 실제로 실행할 공통 코드. `@Before/@After/@Around` 등으로 시점 지정
- **Aspect**: Advice + Pointcut을 묶은 모듈 단위. "어디서(Pointcut) 무엇을(Advice)" 정의
- **Weaving**: Aspect를 타겟 객체에 연결하는 과정. Spring AOP는 런타임(프록시) 방식

**꼬리 질문 예시:**
- Pointcut 표현식 `execution(* com.example.service.*.*(..))` 을 해석해주세요.
- Weaving 시점 3가지를 비교해주세요.

> 출처: https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html

---

## @Around, @Before, @After의 차이와 언제 각각 사용해야 하나요?

**난이도:** 중급

**핵심 키워드:** @Around, @Before, @AfterReturning, @AfterThrowing, ProceedingJoinPoint, proceed()

**모범 답변 방향:**
- `@Before`: 메서드 실행 전. 실행 자체를 막을 수 없음 (예외 throw는 가능)
- `@AfterReturning`: 정상 반환 후. 반환값 접근 가능. 예외 시 미실행
- `@AfterThrowing`: 예외 발생 후. 예외 객체 접근 가능
- `@After`: 항상 실행 (finally와 유사). 정상/예외 무관
- `@Around`: 가장 강력. `proceed()` 호출로 실행 여부/시점 제어. 반환값 변경 가능
- 실무 원칙: **필요한 최소 타입 사용**. 단순 로깅이면 `@Before`, 시간 측정이면 `@Around`

**꼬리 질문 예시:**
- `@Around`에서 `proceed()`를 호출하지 않으면 어떻게 되나요?
- `@AfterReturning`과 `@Around`에서 반환값을 바꾸는 방법의 차이는?

> 출처: https://www.swiftorial.com/tutorials/backend_framework/spring_framework/spring_aop/best_practices_for_spring_aop

---

## Spring AOP가 동작하지 않는 경우와 해결 방법은?

**난이도:** 중급

**핵심 키워드:** self-invocation, 프록시 우회, this 직접 호출, CGLIB, 별도 클래스 분리

**모범 답변 방향:**
- Spring AOP는 프록시 기반 → **같은 클래스 내 `this.메서드()` 호출은 프록시를 우회** → AOP 미적용
- `final` 메서드에 CGLIB 적용 불가 (상속으로 오버라이드 불가)
- Spring Bean이 아닌 일반 객체(`new`로 생성)에는 AOP 미적용
- 해결책: 별도 클래스 분리(권장) → 외부 빈 호출로 프록시 경유, self-injection은 순환 참조 위험

**꼬리 질문 예시:**
- `@Transactional`이 self-invocation에서 동작하지 않는 이유도 같은 원리인가요?
- `final` 클래스에 `@Transactional`이 적용되지 않는 이유는?

> 출처: https://f-lab.ai/en/insight/spring-interview-preparation-20250124

---

## @Transactional 주의사항

**Q. `@Transactional`의 동작 원리와 실무에서 주의할 점을 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** AOP Proxy, self-invocation, propagation, checked exception rollback

**모범 답변 방향:**
- AOP Proxy 기반 → 외부 호출만 proxy 경유, 내부 호출은 `this` 직접 호출로 우회
- self-invocation 해결: 별도 클래스 분리(권장) > self-injection
- checked exception은 기본 rollback 안 됨 → `rollbackFor = Exception.class`
- `REQUIRES_NEW`: 새 트랜잭션 생성, 로그/감사 기록에 활용

**꼬리 질문 예시:**
- 같은 클래스 내 `@Transactional` 메서드를 내부 호출하면 어떻게 되나요?
- checked exception과 unchecked exception의 rollback 기본 동작 차이는?
- `REQUIRES_NEW`는 언제 사용하나요? 주의점은?

---

## JPA N+1

**Q. JPA에서 N+1 문제가 무엇이고 어떻게 해결하나요?**

**난이도:** 중급

**핵심 키워드:** lazy loading, fetch join, @EntityGraph, N+1 = 1+N번, pagination 주의

**모범 답변 방향:**
- N+1 = 1(목록 조회) + N(각 연관 엔티티 조회) = **N+1번** 쿼리 (100개면 101번)
- 발생 조건: lazy loading 연관 엔티티를 반복문에서 접근
- 해결: fetch join(`JOIN FETCH`) → 1번 쿼리로 해결
- @EntityGraph: 어노테이션 방식, 단순 조회에 적합
- 주의: fetch join + pagination 동시 사용 시 메모리 페이징(HHH90003004 경고)

**꼬리 질문 예시:**
- fetch join과 @EntityGraph의 차이는 언제 문제가 되나요?
- fetch join + Pageable을 같이 쓰면 어떤 문제가 생기나요? 어떻게 해결하나요?
- 즉시로딩(EAGER)으로 설정하면 N+1이 해결되나요?

---

## @Transactional 을 직접 구현한다면 어떻게 해야 하나요?

**난이도:** 심화

**핵심 키워드:** CGLIB, JDK Dynamic Proxy, TransactionInterceptor, PlatformTransactionManager, MethodInterceptor, Strategy Pattern

**모범 답변 방향:**
- Spring은 `@EnableTransactionManagement`로 대상 Bean을 AOP Proxy로 교체
- Spring Boot 2.x 기본은 CGLIB — 클래스를 상속하여 메서드를 오버라이드하는 방식으로 가로챔
- 가로챈 시점에 `TransactionInterceptor.invoke()` 실행 → `PlatformTransactionManager`에 실제 처리 위임 (Strategy Pattern)
- `PlatformTransactionManager`는 JDBC면 `con.setAutoCommit(false)` → `commit()` or `rollback()`
- 직접 구현 시: `MethodInterceptor` 구현 → `PlatformTransactionManager` 주입 → `getTransaction()` → 메서드 실행 → `commit/rollback`

**꼬리 질문 예시:**
- CGLIB와 JDK Dynamic Proxy 중 어느 것이 Spring Boot 기본이고 왜 그렇게 바뀌었나요?
- `TransactionInterceptor`가 `PlatformTransactionManager`에 위임하는 패턴은 어떤 디자인 패턴인가요?
- `final` 메서드에 `@Transactional`을 붙이면 왜 동작하지 않나요?
- JDK Proxy는 인터페이스가 없으면 왜 사용 불가한가요?

> 출처: https://www.marcobehler.com/guides/spring-transaction-management-transactional-in-depth
> https://medium.com/@meet2sudhakar/spring-transaction-management-a-deep-dive-into-the-architecture-762b14a81f47

---

## 작성 예정

- JVM GC 알고리즘 (G1GC, ZGC)
- Spring Bean 스코프와 생명주기
- Java Virtual Thread (Project Loom)
- 동시성: synchronized, Lock, atomic
