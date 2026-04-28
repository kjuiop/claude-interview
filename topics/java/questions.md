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

AOP(Aspect-Oriented Programming)는 OOP만으로는 분리하기 어려운 횡단 관심사(Cross-Cutting Concern)를 Aspect라는 별도 모듈로 추출해 핵심 비즈니스 로직에서 완전히 제거하는 프로그래밍 패러다임입니다. OOP는 객체 단위로 비즈니스 책임을 분리하는 데 탁월하지만, 로깅·트랜잭션·보안·캐시처럼 수십 개의 클래스에 걸쳐 반복적으로 등장하는 공통 관심사를 깔끔하게 분리하지 못하는 구조적 한계가 있습니다. 이런 공통 관심사를 횡단 관심사라고 부르는데, 예를 들어 트랜잭션 처리를 OOP만으로 구현하면 모든 서비스 메서드에 `begin/commit/rollback` 로직이 중복됩니다. 이 공통 코드를 변경하려면 해당 로직이 퍼진 모든 클래스를 일일이 수정해야 하고, 실수로 빠뜨리면 버그가 됩니다. AOP는 이 문제를 Aspect 분리로 해결합니다. 횡단 관심사를 Aspect라는 별도 클래스로 추출하면, 비즈니스 코드에는 순수 로직만 남고 공통 기능은 Aspect 한 곳에서 관리됩니다. 변경이 필요할 때는 Aspect 하나만 수정하면 모든 적용 지점에 반영되어 유지보수성이 크게 향상됩니다. 실무에서 가장 친숙한 예는 Spring의 `@Transactional`입니다. 이 어노테이션은 내부적으로 CGLIB 프록시를 생성해 메서드 호출을 가로채고 트랜잭션 begin/commit/rollback을 자동으로 처리합니다. 개발자는 비즈니스 로직에만 집중하고 트랜잭션 관리는 프레임워크에 위임할 수 있어 코드의 응집도와 단일 책임이 유지됩니다. OOP와 AOP는 상호 보완적인 관계로, OOP로 객체 책임을 분리하고 AOP로 횡단 관심사를 분리하는 조합이 Spring 애플리케이션의 기본 설계 원칙입니다.

**꼬리 질문 예시:**
- 횡단 관심사의 실제 예시를 들어주세요.
- Spring에서 AOP는 어떤 방식으로 구현되어 있나요?

**면접 세션 피드백 (2026-04-02 5회차)**:
- 잘한 점: `@Transactional` 예시로 "모든 구현체에서 begin/commit/rollback 직접 호출" 문제 → AOP로 wrapping 해결 설명 정확
- 보완: "횡단 관심사(Cross-Cutting Concern)" 용어 미사용. AOP = "어노테이션으로 묶은 것"이 아니라 "횡단 관심사 분리 패러다임"이 본질. OOP 한계와 연결 필요.

> 출처: https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html

---

## Aspect, Advice, Pointcut, JoinPoint를 각각 설명해주세요.

**난이도:** 기초

**핵심 키워드:** Aspect, Advice, Pointcut, JoinPoint, Weaving

**모범 답변 방향:**

네 가지 용어는 계층적으로 연결되며 각각 명확한 역할을 가집니다. JoinPoint는 AOP를 적용할 수 있는 모든 후보 지점입니다. AspectJ는 메서드 실행, 필드 접근, 생성자 호출 등 다양한 JoinPoint를 지원하지만, Spring AOP는 메서드 실행 시점만 JoinPoint로 지원합니다. Pointcut은 그 JoinPoint 중에서 실제로 Advice를 적용할 대상을 선별하는 표현식입니다. `execution(* com.example.service.*.*(..))` 같은 형태로 작성하며, JoinPoint 전체 집합에서 특정 조건을 만족하는 부분집합을 잘라낸다는 의미에서 "Point를 cut(잘라낸다)"로 이해하면 혼동을 줄일 수 있습니다. 표현식을 해석하면 `*`(모든 반환 타입) `com.example.service`(패키지) `*`(모든 클래스) `.`(의) `*`(모든 메서드) `(..)`(파라미터 개수 무관)입니다. Advice는 Pointcut으로 선별된 JoinPoint에서 실제로 실행할 공통 코드이며, `@Before`(메서드 실행 전), `@AfterReturning`(정상 반환 후), `@AfterThrowing`(예외 발생 후), `@After`(정상·예외 무관 항상), `@Around`(전후 모두 제어)로 실행 시점을 지정합니다. Aspect는 Advice와 Pointcut을 하나의 클래스로 묶은 모듈 단위입니다. "어디서(Pointcut) 무엇을(Advice) 실행할지"를 한 곳에 정의해 횡단 관심사를 캡슐화합니다. 마지막으로 Weaving은 Aspect를 타겟 Bean에 실제로 연결하는 과정이며, 시점에 따라 컴파일 타임 Weaving(AspectJ 컴파일러 필요), 클래스 로드 타임 Weaving(ClassLoader 조작), 런타임 Weaving(Spring AOP, CGLIB 프록시 생성) 세 가지 방식이 있습니다. Spring AOP는 런타임 프록시 방식을 사용해 별도 컴파일러 없이 바로 사용할 수 있지만, 그 대가로 메서드 실행 JoinPoint만 지원하고 self-invocation에서는 프록시가 작동하지 않는 제약이 있습니다.

**꼬리 질문 예시:**
- Pointcut 표현식 `execution(* com.example.service.*.*(..))` 을 해석해주세요.
- Weaving 시점 3가지를 비교해주세요.

**면접 세션 피드백 (2026-04-02 5회차)**:
- 보완: Pointcut/JoinPoint 혼동(자주 나오는 실수). Aspect 미언급.
- 암기 우선: JoinPoint(모든 후보 지점) → Pointcut(선별 표현식) → Advice(코드+시점) → Aspect(둘을 묶은 모듈)
- 암기 팁: "Join(합류)Point = 합류 가능한 모든 지점, Point(를)cut(잘라서) = 그 중 선택한 것"

> 출처: https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html

---

## @Around, @Before, @After의 차이와 언제 각각 사용해야 하나요?

**난이도:** 중급

**핵심 키워드:** @Around, @Before, @AfterReturning, @AfterThrowing, ProceedingJoinPoint, proceed()

**모범 답변 방향:**

다섯 가지 Advice 타입은 실행 시점과 접근 가능한 정보에서 명확히 구분되며, 목적에 맞는 타입을 선택하는 것이 중요합니다. `@Before`는 메서드 실행 전에 동작하며, 실행 자체를 취소할 수는 없고 예외를 던지는 방식으로만 흐름을 중단할 수 있습니다. 파라미터 유효성 검사나 권한 확인처럼 사전 조건 검증에 적합합니다. `@AfterReturning`은 메서드가 정상적으로 반환한 후에만 실행되며 `returning` 속성으로 반환값에 접근할 수 있습니다. 예외가 발생하면 실행되지 않으므로 성공 시 로깅이나 후처리에 사용합니다. `@AfterThrowing`은 예외가 발생했을 때만 실행되며 `throwing` 속성으로 예외 객체에 접근할 수 있어 공통 예외 로깅이나 Slack 알림 발송 등에 활용합니다. `@After`는 정상 반환과 예외 발생 모두에서 항상 실행되어 Java의 `finally` 블록과 동일한 역할을 합니다. 리소스 해제나 락 반환처럼 결과와 무관하게 반드시 실행해야 하는 로직에 사용합니다. `@Around`는 가장 강력한 타입으로, `ProceedingJoinPoint.proceed()`를 직접 호출해 메서드 실행 여부와 타이밍을 완전히 제어할 수 있습니다. 반환값도 가로채거나 변경할 수 있어 메서드 실행 시간 측정, 캐싱, 재시도 로직처럼 전후 모두 제어가 필요한 경우에 사용합니다. 실무 원칙은 목적에 맞는 최소 권한의 타입을 선택하는 것입니다. `@Around`는 `proceed()`를 호출하지 않으면 대상 메서드 자체가 실행되지 않는 위험이 있어 꼭 필요한 경우에만 제한적으로 사용해야 합니다. Spring의 `@Transactional`도 내부적으로 `@Around`와 동일한 방식으로, `proceed()` 이전에 트랜잭션 `begin`을 실행하고 이후에 `commit` 또는 `rollback`을 처리합니다. 이 구조를 이해하면 `@Transactional`의 동작 원리와 한계(self-invocation, `final` 메서드)를 설명하는 데도 연결할 수 있습니다.

**꼬리 질문 예시:**
- `@Around`에서 `proceed()`를 호출하지 않으면 어떻게 되나요?
- `@AfterReturning`과 `@Around`에서 반환값을 바꾸는 방법의 차이는?

> 출처: https://www.swiftorial.com/tutorials/backend_framework/spring_framework/spring_aop/best_practices_for_spring_aop

---

## Spring AOP가 동작하지 않는 경우와 해결 방법은?

**난이도:** 중급

**핵심 키워드:** self-invocation, 프록시 우회, this 직접 호출, CGLIB, 별도 클래스 분리

**모범 답변 방향:**

Spring AOP가 동작하지 않는 케이스를 이해하려면 먼저 Spring AOP의 동작 원리를 알아야 합니다. Spring AOP는 CGLIB 프록시를 사용해 Bean을 감싸고, 외부에서 Bean의 메서드를 호출할 때 프록시가 가로채 Advice를 실행하는 방식입니다. 다시 말해 프록시를 경유하지 않는 상황이면 어느 케이스든 AOP가 동작하지 않습니다. 이 구조에서 세 가지 대표적인 동작 불능 케이스가 발생합니다. 첫째는 self-invocation, 즉 같은 클래스 내부에서 `this.메서드()`로 직접 호출하는 경우입니다. 외부에서 주입된 프록시 객체를 통한 호출이 아니라 실제 객체가 자기 자신의 메서드를 직접 호출하기 때문에 AOP 인터셉터가 개입할 기회가 없습니다. `@Transactional`이 self-invocation에서 트랜잭션이 시작되지 않는 이유도 완전히 동일한 원리입니다. 이 케이스는 개발 중에 실수하기 쉽고 테스트에서도 잡히지 않는 경우가 많아 실무에서 자주 문제가 됩니다. 권장 해결책은 공통 로직을 별도 Spring Bean 클래스로 분리해 외부 호출 구조로 만드는 것입니다. self-injection(`@Autowired`로 자기 자신 주입 후 프록시를 통해 호출)도 기술적으로는 가능하지만 순환 참조 위험과 가독성 저하로 권장하지 않습니다. 둘째는 `final` 메서드입니다. CGLIB는 대상 클래스를 상속해 메서드를 오버라이드하는 방식으로 프록시를 생성하는데, `final`로 선언된 메서드는 Java 언어 규칙상 오버라이드가 불가하므로 프록시가 가로챌 수 없어 Advice가 실행되지 않습니다. `final` 키워드를 제거하거나 인터페이스 기반 설계로 전환하는 것이 해결책입니다. 셋째는 `new`로 직접 생성한 객체입니다. Spring 컨테이너가 관리하지 않는 객체에는 프록시 자체가 생성되지 않으므로 AOP가 전혀 적용되지 않습니다. 반드시 `@Autowired`나 생성자 주입으로 Spring Bean을 주입받아 사용해야 합니다.

**꼬리 질문 예시:**
- `@Transactional`이 self-invocation에서 동작하지 않는 이유도 같은 원리인가요?
- `final` 클래스에 `@Transactional`이 적용되지 않는 이유는?

**면접 세션 피드백 (2026-04-02 5회차)**:
- 현황: 전혀 몰랐음 → 신규 암기
- 핵심: Spring AOP = CGLIB 프록시 기반. self-invocation(`this.메서드()`) = 프록시 우회 = AOP 미적용
- 3가지 케이스: (1)self-invocation → 별도 클래스 분리, (2)final 메서드 → final 제거, (3)new 직접 생성 → Spring Bean 주입 사용

> 출처: https://f-lab.ai/en/insight/spring-interview-preparation-20250124

---

## @Transactional 주의사항

**Q. `@Transactional`의 동작 원리와 실무에서 주의할 점을 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** AOP Proxy, self-invocation, propagation, checked exception rollback

**모범 답변 방향:**

`@Transactional`의 핵심 동작 원리는 Spring AOP CGLIB 프록시입니다. Spring 컨테이너가 `@Transactional`이 붙은 Bean을 등록할 때, 실제 객체를 CGLIB로 상속한 프록시 객체로 교체합니다. 외부에서 해당 메서드를 호출하면 프록시가 먼저 가로채 트랜잭션을 시작(`setAutoCommit(false)`)하고, 메서드 실행 후 정상이면 `commit`, 예외 발생 시 `rollback`을 처리합니다. 실무에서 반드시 알아야 할 주의사항은 세 가지입니다. 첫째, self-invocation 문제입니다. 같은 클래스 내에서 `this.메서드()`로 직접 호출하면 프록시를 거치지 않아 트랜잭션이 시작되지 않습니다. 별도 클래스로 분리하는 것이 권장 해결책입니다. 둘째, checked exception의 롤백 기본 동작입니다. Spring은 기본적으로 `RuntimeException`(unchecked exception)에 대해서만 롤백하고, `IOException` 같은 checked exception은 롤백하지 않습니다. 예를 들어 결제 처리 중 PG사 API 호출에서 `IOException`이 발생했을 때 `rollbackFor`를 설정하지 않으면 결제 레코드가 DB에 커밋되어 실제 결제 실패와 데이터 불일치가 발생합니다. 이중 청구 같은 심각한 데이터 정합성 문제로 이어질 수 있으므로 외부 API를 호출하는 트랜잭션에는 반드시 `@Transactional(rollbackFor = Exception.class)`를 명시해야 합니다. 셋째, `REQUIRES_NEW`의 활용입니다. 메인 트랜잭션이 실패해 롤백되더라도 반드시 저장해야 하는 데이터(감사 로그, 실패 이력)가 있을 때 `REQUIRES_NEW`로 분리하면 완전히 독립된 트랜잭션에서 별도 커밋이 가능합니다. 단, 별도 DB 커넥션을 하나 더 사용한다는 점에서 커넥션 풀 고갈 위험을 인지하고 사용해야 합니다.

**꼬리 질문 예시:**
- 같은 클래스 내 `@Transactional` 메서드를 내부 호출하면 어떻게 되나요?
- checked exception과 unchecked exception의 rollback 기본 동작 차이는?
- `REQUIRES_NEW`는 언제 사용하나요? 주의점은?

**면접 세션 피드백 (2026-04-02 3회차 — checked exception rollback)**:
- 잘한 점: checked/unchecked 이유 정확(개발자 의도 vs 런타임 장애). `rollbackFor = Exception.class` 설정 정확. 꼬리 답변에서 "실패 상태 보존 → 재시도 효율" 실무 관점 강점.
- 보완:
  - 핵심 리스크 시나리오 먼저: "PG사 API IOException(checked) 발생 시 rollbackFor 없으면 결제 레코드가 DB에 '처리중'으로 커밋 → 실제 결제 실패와 DB 상태 불일치 → 이중 청구 위험"
  - 실패 로그 보존 패턴: `REQUIRES_NEW` 별도 트랜잭션으로 실패 기록 저장 → 메인 롤백과 무관하게 실패 이유 보존
  ```java
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void saveFailureLog(PaymentFailureLog log) { ... }
  ```

---

## @Transactional Propagation — REQUIRED, NESTED, REQUIRES_NEW 비교

**난이도:** 기초

**핵심 키워드:** REQUIRED join, NESTED 세이브포인트, REQUIRES_NEW 커넥션 2개, JPA NESTED 미지원

**REQUIRED (기본값)**:
- 기존 트랜잭션이 있으면 참여(join), 없으면 새로 생성
- serviceA() → serviceB() 호출 시 둘 다 REQUIRED면 **같은 트랜잭션**으로 묶임
- serviceB()가 실패하면 serviceA() 전체가 함께 롤백

**NESTED**:
- 기존 트랜잭션 안에 **세이브포인트(savepoint)** 를 찍고 중첩 실행
- 내부 실패 시 세이브포인트까지만 롤백, 외부 트랜잭션은 계속 진행
- REQUIRES_NEW와 달리 **같은 DB 커넥션** 사용 (오버헤드 낮음)
- **JPA 환경에서 미지원**: Hibernate가 세이브포인트를 공식 지원하지 않아 예외 발생 가능

**REQUIRES_NEW**:
- 기존 커넥션을 **일시 중단**하고 새 커넥션을 열어 완전히 독립된 트랜잭션 생성
- 두 트랜잭션이 완전히 독립적으로 커밋/롤백
- 실무 사례: **감사 로그** — 메인 로직 실패 → 롤백되어도 "누가 언제 시도했다"는 기록은 반드시 저장

**커넥션 수 비교**:
| propagation | 커넥션 수 | 특징 |
|---|---|---|
| REQUIRED | 1개 | join 또는 새 트랜잭션 시작 |
| NESTED | 1개 | 세이브포인트 추가 (같은 커넥션) |
| REQUIRES_NEW | 2개 | 기존 커넥션 중단 + 새 커넥션 |

**꼬리 질문 예시:**
- NESTED와 REQUIRES_NEW의 결정적 차이는? (커넥션 수, JPA 지원 여부)
- JPA 환경에서 "내부 실패해도 외부 트랜잭션 계속 진행"을 구현하려면? → REQUIRES_NEW + try-catch

**면접 세션 피드백 (2026-04-16 2회차 — NESTED 완전 모름)**:
- 보완: REQUIRED join 동작, NESTED 세이브포인트, JPA 미지원 이유 모두 암기 필요

**면접 세션 피드백 (2026-04-17 1회차)**:
- 잘한 점: REQUIRED join 개념 정확. NESTED 세이브포인트 → 부분 롤백, 외부 트랜잭션 계속 진행 정확.
- 보완: NESTED vs REQUIRES_NEW 커넥션 차이 미언급. JPA 미지원 이유 구체화 필요. 실무 경험(감사 로그 REQUIRES_NEW) 연결 없음.

**면접 세션 피드백 (2026-04-28 2회차)**:
- 잘한 점: REQUIRED/REQUIRES_NEW/NESTED 세 전파 속성 핵심 차이 명확. REQUIRES_NEW 커넥션 2개 + 독립성 트레이드오프 언급.
- 보완: JPA NESTED 미지원 원인 → JpaTransactionManager가 savepoint 미지원, 영속성 컨텍스트와 DB 상태 불일치 위험 때문. NESTED 실무 사례(배치 부분 실패 허용) 추가하면 완성도 향상.
- 점수: 7/10

---

## JPA N+1

**Q. JPA에서 N+1 문제가 무엇이고 어떻게 해결하나요?**

**난이도:** 중급

**핵심 키워드:** lazy loading, fetch join, @EntityGraph, N+1 = 1+N번, pagination 주의

**모범 답변 방향:**

N+1 문제는 1번의 목록 조회 쿼리 이후 각 엔티티의 연관 데이터를 로드하기 위해 N번의 추가 쿼리가 발생하는 현상입니다. JPA의 기본 전략인 Lazy Loading이 원인으로, 예를 들어 게시글 100개를 조회한 후 각 게시글의 `author` 필드에 접근하면 100번의 추가 SELECT가 발생해 총 101번의 쿼리가 실행됩니다. 개발 환경에서는 데이터가 적어 눈치채지 못하다가 운영에서 데이터가 쌓이면 갑자기 응답 시간이 폭발적으로 증가하는 전형적인 패턴입니다. 해결 방법은 세 가지이며 각각 적합한 상황이 다릅니다. 첫째로 Fetch Join은 JPQL에서 `JOIN FETCH`를 사용해 1번 쿼리로 연관 엔티티를 함께 로드합니다. Querydsl에서는 `.fetchJoin()`을 사용합니다. 단, OneToMany 관계에서 Fetch Join 결과에 중복 row가 생기므로 `DISTINCT`를 함께 사용해야 합니다. 둘째로 `@EntityGraph`는 Repository 메서드 위에 선언해 어노테이션만으로 Fetch Join 효과를 내는 방법입니다. JPQL 없이 단순 조회 메서드에 적합합니다. 셋째로 `@BatchSize`는 N+1을 IN 절 쿼리로 묶어 처리해 쿼리 수를 줄이는 방법입니다. Fetch Join 사용 시 반드시 알아야 할 주의사항은 Pagination과의 충돌입니다. Fetch Join에 `Pageable`을 함께 쓰면 Hibernate가 DB에 LIMIT을 적용하지 못하고 전체 데이터를 메모리에 올린 뒤 애플리케이션에서 페이징 처리합니다. 이때 Hibernate가 `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!` 경고를 출력합니다. 수백만 건 데이터라면 OutOfMemoryError로 서버가 다운될 수 있으므로 페이징이 필요한 경우 ID를 먼저 페이지네이션한 후 IN 절로 Fetch Join하는 2단계 조회나 `@BatchSize`를 사용해야 합니다.

**꼬리 질문 예시:**
- fetch join과 @EntityGraph의 차이는 언제 문제가 되나요?
- fetch join + Pageable을 같이 쓰면 어떤 문제가 생기나요? 어떻게 해결하나요?
- 즉시로딩(EAGER)으로 설정하면 N+1이 해결되나요?

**Fetch Join + Pagination 상세** (2026-04-01 세션 보완):
- Hibernate 경고: `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!`
- 동작: DB에 LIMIT/OFFSET 미적용 → **전체 데이터를 메모리에 올린 후 애플리케이션에서 페이징** → OOM 위험
- 해결 방법 2가지:
  ```java
  // 방법 1: ID 먼저 페이지네이션 후 fetch join
  List<Long> ids = queryFactory.select(post.id).from(post)
      .offset(pageable.getOffset()).limit(pageable.getPageSize()).fetch();
  List<Post> posts = queryFactory.selectFrom(post)
      .join(post.author).fetchJoin()
      .where(post.id.in(ids)).fetch();

  // 방법 2: @BatchSize — N+1을 IN 쿼리로 묶어 처리
  @BatchSize(size = 100)
  @OneToMany(mappedBy = "post")
  private List<Comment> comments;
  ```

**@EntityGraph 선언 위치 주의** (자주 나오는 오답):
- ❌ Entity 필드 위 → ✅ **Repository 메서드 위**에 선언
  ```java
  @EntityGraph(attributePaths = {"author"})
  List<Post> findAll();  // Repository 메서드에 선언
  ```
- Entity 클래스 위에 이름 등록하는 것은 `@NamedEntityGraph` (별도 개념)

**면접 세션 피드백 (2026-04-07 4회차)**:
- 잘한 점: N+1 정의 정확(101번 쿼리). BatchSize + ID 분리 두 가지 제시. 꼬리질문에서 OOM 위험으로 정확히 교정.
- 보완:
  - 초기 답변에서 "10개보다 적은 상품 조회" 오류 주의 → Hibernate는 틀린 결과가 아닌 전체 메모리 적재 후 페이징
  - Hibernate 경고 암기: `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!`
  - @EntityGraph 누락: Fetch Join과 동일 효과를 어노테이션으로 선언
  - 세 해결책 트레이드오프 비교: Fetch Join(쿼리 1, 페이징 불가) / ID 분리(쿼리 2, 페이징 가능) / BatchSize(IN 쿼리, 튜닝 필요)

**면접 세션 피드백 (2026-04-16 1회차)**:
- 잘한 점: Lazy Loading 원인 정확. fetch join → @EntityGraph → batch fetch 세 가지 모두 설명. batch fetch 내부 동작(ID 수집 → IN절 묶음 조회) 설명. @EntityGraph 선택 기준("단순하면 EntityGraph, 복잡하면 fetch join") 실무 관점 정확.
- 보완: **batch fetch 선택 이유 핵심 미언급** — "페이징이 필요할 때 fetch join 대신 batch fetch"가 핵심 답. HHH90003004 경고 아직 즉시 암기 안 됨. 2회 이상 보완으로 나온 항목이므로 다음 세션 전 반드시 암기.

**면접 세션 피드백 (2026-04-17 3회차)**:
- 잘한 점: 행 수 불일치 원인(Post+Comment Join 행 수 뒤틀림)과 Hibernate 메모리 전체 로드 동작 정확. ID 먼저 조회 → IN 절 패턴 실용적. @BatchSize 동작 원리(N+1을 IN 절로 묶음) 올바르게 설명.
- 보완:
  - **OOM 위험 미언급**: "메모리 부담이 크다" → "수백만 건 시 OutOfMemoryError로 서버 다운 가능"으로 명시
  - **HHH90003004 경고 여전히 미언급**: 3회차에도 나오지 않음 — 운영 리스크 키워드로 반드시 암기
  - **실무 연결 없음**: Spring Boot 프로젝트 경험과 연결 필요

**면접 세션 피드백 (2026-04-21 3회차)**:
- 잘한 점: N+1 원인·세 가지 해결책 전반 정확. Fetch Join + Pagination → Hibernate 메모리 페이징 → OOM 위험 언급 — 차별화 포인트.
- 보완:
  - **DISTINCT 미언급**: OneToMany Fetch Join 중복 row → `SELECT DISTINCT p FROM Post p JOIN FETCH p.comments`로 제거. "id 먼저 조회+IN절"은 @BatchSize 방식이며 중복 제거 답이 아님
  - **MultipleBagFetchException**: 두 개 이상의 컬렉션(List) 동시 Fetch Join 시 발생. 해결: 하나를 `Set`으로 변경 or 하나만 Fetch Join + 나머지 @BatchSize
  - 이력서 연결 없음 — 샵라이브 상품-옵션-카테고리 연관관계 조회 경험 연결 가능

---

## @Transactional 을 직접 구현한다면 어떻게 해야 하나요?

**난이도:** 심화

**핵심 키워드:** CGLIB, JDK Dynamic Proxy, TransactionInterceptor, PlatformTransactionManager, MethodInterceptor, Strategy Pattern

**모범 답변 방향:**

`@Transactional`을 직접 구현하려면 AOP 프록시 레이어와 트랜잭션 매니저 레이어 두 축으로 설계해야 합니다. Spring의 실제 구현을 역으로 따라가면 설계가 명확해집니다. Spring은 `@EnableTransactionManagement` 어노테이션이 등록되면 `@Transactional`이 붙은 Bean을 CGLIB 프록시로 감쌉니다. Spring Boot 2.x부터는 인터페이스 유무와 관계없이 기본값이 CGLIB입니다. CGLIB 프록시는 대상 클래스를 상속해 모든 메서드를 오버라이드하고, 메서드가 호출될 때 `TransactionInterceptor.invoke()`로 흐름을 가로챕니다. `TransactionInterceptor`는 Advice 역할을 하며, 실제 트랜잭션 처리는 `PlatformTransactionManager`에 위임합니다. 이 위임 구조가 Strategy Pattern으로, 런타임에 JDBC 환경이면 `DataSourceTransactionManager`, JPA 환경이면 `JpaTransactionManager` 구현체가 주입되어 동일한 인터페이스로 트랜잭션을 제어합니다. `DataSourceTransactionManager`는 내부적으로 `DataSource`에서 커넥션을 꺼내 `con.setAutoCommit(false)`로 트랜잭션을 시작하고, 메서드가 정상 완료되면 `con.commit()`, `RuntimeException`이 발생하면 `con.rollback()`을 호출합니다. 직접 구현한다면 `MethodInterceptor`를 구현해 `PlatformTransactionManager`를 주입받고, `getTransaction()` → `methodInvocation.proceed()` → 성공 시 `commit()` / 실패 시 `rollback()` 흐름으로 작성하면 Spring의 `@Transactional`과 동일한 동작을 만들 수 있습니다. 이 구조를 이해하면 `final` 메서드에 `@Transactional`이 동작하지 않는 이유(CGLIB가 오버라이드 불가), self-invocation에서 트랜잭션이 시작되지 않는 이유(프록시를 경유하지 않으므로 `TransactionInterceptor`가 개입 불가)도 자연스럽게 설명됩니다.

**꼬리 질문 예시:**
- CGLIB와 JDK Dynamic Proxy 중 어느 것이 Spring Boot 기본이고 왜 그렇게 바뀌었나요?
- `TransactionInterceptor`가 `PlatformTransactionManager`에 위임하는 패턴은 어떤 디자인 패턴인가요?
- `final` 메서드에 `@Transactional`을 붙이면 왜 동작하지 않나요?
- JDK Proxy는 인터페이스가 없으면 왜 사용 불가한가요?

> 출처: https://www.marcobehler.com/guides/spring-transaction-management-transactional-in-depth
> https://medium.com/@meet2sudhakar/spring-transaction-management-a-deep-dive-into-the-architecture-762b14a81f47

---

## Spring IoC/DI — @Component vs @Bean, Bean 스코프, 생명주기 콜백

**난이도**: 기초

**핵심 키워드**: IoC, DI, @Component, @Bean, @Configuration, Singleton, @PostConstruct, @PreDestroy

**모범 답변 방향**:

Spring IoC(Inversion of Control) 컨테이너는 객체의 생성, 의존성 주입, 생명주기 관리를 프레임워크가 담당하는 구조입니다. 제어의 역전이란 개발자가 `new`로 직접 객체를 생성하고 의존성을 연결하던 제어권을 컨테이너에 넘긴다는 의미입니다. DI(Dependency Injection)는 IoC를 구현하는 구체적인 방법으로, 컨테이너가 생성자·필드·세터를 통해 필요한 의존 객체를 자동으로 주입해줍니다. 생성자 주입이 권장되는 이유는 테스트 시 Mock 주입이 쉽고, `final` 필드로 선언해 불변성을 보장할 수 있기 때문입니다. `@Component`와 `@Bean`은 Bean 등록 방식의 차이입니다. `@Component`는 내가 작성한 클래스에 직접 붙여 컴포넌트 스캔 시 자동으로 Bean으로 등록하는 방식입니다. `@Service`, `@Repository`, `@Controller`는 모두 내부에 `@Component`를 포함하는 메타 어노테이션으로, 계층 역할을 명시적으로 구분하기 위한 의미론적 차이가 있습니다. `@Bean`은 `@Configuration` 클래스의 메서드에 붙여 외부 라이브러리 클래스나 복잡한 초기화 설정이 필요한 객체를 수동으로 등록할 때 사용합니다. `DataSource`, `ObjectMapper`, `RestTemplate` 같이 소스 코드를 수정할 수 없는 라이브러리 클래스는 `@Component`를 붙일 수 없으므로 반드시 `@Bean`으로 등록해야 합니다. Bean의 기본 스코프는 Singleton으로, 컨테이너당 인스턴스 하나만 생성해 모든 요청이 같은 객체를 공유합니다. 이 때문에 Singleton Bean에 가변 상태를 저장하면 여러 스레드가 동시에 같은 인스턴스를 수정해 race condition이 발생합니다. Prototype 스코프는 요청마다 새 인스턴스를, Request 스코프는 HTTP 요청마다 새 인스턴스를 생성합니다. 생명주기 콜백으로는 `@PostConstruct`(의존성 주입 완료 직후 실행, DB 연결 초기화나 캐시 워밍업에 적합)와 `@PreDestroy`(컨테이너 소멸 직전 실행, 커넥션 해제나 스레드 풀 종료에 사용)가 있습니다.

**꼬리 질문 예시**:
- "@Component와 @Bean을 헷갈리면 어떤 문제가 생기나요?"
- "Singleton Bean에 상태를 저장하면 왜 위험한가요?" → 여러 요청이 같은 인스턴스를 공유해 race condition 발생

**면접 세션 피드백 (2026-04-17 4회차)**:
- 잘한 점: @PostConstruct/@PreDestroy 타이밍 정확
- 보완: IoC 컨테이너 역할·@Component vs @Bean 차이·Singleton 스코프 모두 모름 — 인포뱅크 필수 스택 기초 질문이므로 반드시 암기

---

## 작성 예정

- JVM GC 알고리즘 (G1GC, ZGC)
- Java Virtual Thread (Project Loom)
- 동시성: synchronized, Lock, atomic

---

## Netty의 동작 원리를 설명하고, Event Loop가 무엇인지 설명해주세요.

**난이도**: 심화

**핵심 키워드**: NIO, Reactor 패턴, EventLoopGroup, Channel, ChannelPipeline, Boss/Worker, Non-Blocking I/O

**모범 답변 방향**:

Netty는 Java 기반의 비동기 이벤트 기반 네트워크 프레임워크입니다. 기존 Java BIO 방식은 연결당 스레드 1개를 할당하기 때문에 연결이 많아질수록 스레드 수가 폭발적으로 증가하는 문제가 있었습니다. Netty는 NIO(Non-Blocking I/O) 기반의 Reactor 패턴으로 소수의 스레드로 대량 연결을 처리합니다. gRPC, Kafka, Elasticsearch, Spring WebFlux 내부 HTTP 서버(reactor-netty)가 모두 Netty를 기반으로 동작합니다. Reactor 패턴은 I/O 이벤트가 발생할 때까지 대기하다가 이벤트가 발생하면 등록된 핸들러에 디스패치하는 방식으로, Event Loop 스레드가 블로킹되지 않는 것이 핵심입니다.

**Netty 핵심 구조**:
```
[Boss EventLoopGroup]     — Accept 전용 (연결 수락)
       │
[Worker EventLoopGroup]   — I/O 처리 (읽기/쓰기), CPU 코어 수 × 2 스레드
       │
[Channel]                 — 연결된 소켓을 추상화
       │
[ChannelPipeline]         — ChannelHandler 체인 (인코딩, 디코딩, 비즈니스 로직)
```

**Event Loop 동작 원리**:
1. Selector(NIO)로 I/O 이벤트 감지 (select/epoll)
2. 이벤트 발생 시 해당 Channel의 ChannelPipeline 통해 Handler 호출
3. 완료 후 다시 Selector 대기 → **스레드가 블로킹 없이 순환**
4. 모든 I/O 작업은 비동기 → `ChannelFuture`로 완료 콜백 처리

**Event Loop 사용 시 주의**:
- **Event Loop 스레드를 절대 블로킹하지 말 것** — DB 쿼리, HTTP 호출 등 블로킹 작업은 별도 스레드 풀로 오프로드
- 블로킹하면 해당 Loop가 처리하는 모든 Channel이 멈춤

**Spring MVC vs WebFlux 비교**:
| | Spring MVC | Spring WebFlux |
|---|---|---|
| 서버 | Tomcat (BIO/NIO) | Netty (NIO) |
| 스레드 모델 | 요청당 스레드 | Event Loop (소수 스레드) |
| 적합 케이스 | CPU Bound, 단순 CRUD | I/O Bound, 대량 동시 연결 |

**꼬리 질문 예시**:
- Netty에서 CPU Bound 작업(이미지 리사이징 등)을 처리할 때 어떻게 설계해야 하나요?
- ChannelPipeline의 Inbound/Outbound Handler 차이와 처리 순서는?
- Spring WebFlux에서 `block()` 호출이 위험한 이유는?

> 출처: https://engineering.linecorp.com/ko/blog/do-not-block-the-event-loop-part3/
> 출처: https://mark-kim.blog/netty_deepdive_1/

---

## Spring Security JWT 인증

**세션 피드백 (2026-04-21)**: Filter 흐름과 B2B 설계 OK. OncePerRequestFilter 클래스명 오답(JWT Tokenizer로 답변).

**Q. Spring Security에서 JWT 기반 인증/인가 흐름을 설명하고, B2B API에서 고객사별 API Key 인증을 추가로 설계한다면 어떻게 구성하겠나요?**

**핵심 키워드**: `OncePerRequestFilter`, `UsernamePasswordAuthenticationFilter`, `SecurityContextHolder`, `UsernamePasswordAuthenticationToken`, `addFilterBefore`, Filter Chain 순서

**Filter 구현 핵심:**
```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) {
        // 1. Authorization 헤더에서 Bearer 토큰 추출
        // 2. 서명 검증 + 만료 시간 체크
        // 3. 클레임에서 userId 추출 → 사용자 조회
        // 4. SecurityContext에 Authentication 주입
        SecurityContextHolder.getContext().setAuthentication(
            new UsernamePasswordAuthenticationToken(userDetails, null, authorities)
        );
        filterChain.doFilter(request, response);
    }
}
```

**Filter 등록:**
```java
http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
```

**B2B API Key 설계:**
- JWT Filter와 별도 `ApiKeyFilter extends OncePerRequestFilter` 추가 (앞에 배치)
- X-API-Key 헤더 검증 → DB에서 고객사 정보 조회
- 고객사별 허용 기능은 Role DB 테이블로 관리
- AOP `@Around`에서 SecurityContext의 고객사 정보로 권한 체크 → 없으면 403

**꼬리 질문:**
- JWT Filter를 구현할 때 상속받는 클래스는? (`OncePerRequestFilter`)
- `UsernamePasswordAuthenticationFilter` 앞에 배치하는 이유는?

---

## @Async 비동기 처리 패턴과 @Transactional 관계

**난이도**: 기초

**핵심 키워드**: @Async, ThreadLocal, 트랜잭션 미전파, 스레드 풀 고갈, TaskRejectedException, Outbox 패턴, Kafka

**모범 답변 방향**:

`@Async`는 Spring에서 메서드를 비동기로 실행하는 어노테이션으로, 해당 메서드는 별도의 스레드 풀에서 실행됩니다. 가장 큰 장점은 저장이나 알림 전송처럼 시간이 걸리는 I/O 작업을 caller 스레드의 응답 흐름에서 분리할 수 있다는 것입니다. 예를 들어 사용자 행동 로그를 저장하는 작업에 `@Async`를 적용하면 로그 저장 시간이 API 응답 지연에 영향을 주지 않아 caller 스레드의 레이턴시를 낮출 수 있습니다. 그러나 단점도 분명합니다. 비동기 스레드에서 저장 실패가 발생해도 caller는 이미 응답을 완료한 상태이기 때문에 유실된 데이터를 복구할 수단이 없습니다. 또한 스레드 풀이 가득 찼을 때 `corePoolSize`와 `queueCapacity` 설정에 따라 `TaskRejectedException`이 발생하거나, `CallerRunsPolicy`로 caller 스레드가 직접 처리하게 되어 응답이 블로킹됩니다. 데이터 유실이 허용되지 않는 경우에는 두 가지 전환 전략이 있습니다. 첫째는 Kafka를 통한 비동기 처리입니다. 메시지를 Kafka에 발행하고 컨슈머가 저장하는 방식으로, Kafka의 내구성 덕분에 데이터 유실 없이 비동기 처리가 가능하지만 Kafka 인프라가 필요합니다. 둘째는 Outbox 패턴입니다. Kafka가 없는 환경에서 메인 트랜잭션 안에 Outbox 테이블에 이벤트를 함께 기록하고 별도 워커가 폴링해서 처리하는 방식으로, 트랜잭션의 원자성 덕분에 메인 비즈니스 로직의 성공·실패와 이벤트 기록이 항상 일치해 유실이 방지됩니다. `@Transactional`과의 관계도 중요합니다. Spring 트랜잭션 컨텍스트는 `ThreadLocal`에 저장되는데, `@Async`는 새 스레드를 생성하므로 caller의 `ThreadLocal`을 공유하지 않습니다. 따라서 `@Async` 메서드에는 caller의 트랜잭션이 전파되지 않으며, `@Transactional`을 함께 선언하면 완전히 새로운 독립 트랜잭션이 시작됩니다.

**꼬리 질문 예시**:
- `@Async` 스레드 풀이 가득 찼을 때 기본 동작은? → `TaskRejectedException` 또는 `CallerRunsPolicy` (설정에 따라 다름)
- caller에 트랜잭션이 있을 때 `@Async` 메서드에서 같은 트랜잭션을 쓰려면? → 불가. 별도 트랜잭션이 필요하거나 동기 방식으로 변경해야 함.

**면접 세션 피드백 (2026-04-27 1회차)**:
- 잘한 점: @Async 장단점 + Kafka 전환 방향 정확. @Transactional 미전파 이유(스레드 분리) 꼬리에서 정확 답변.
- 보완: 스레드 풀 고갈(`corePoolSize`, `queueCapacity`) 미언급. Outbox 패턴 미언급. ThreadLocal 메커니즘 구체화 필요.
- 점수: 7/10

---

## WebSocket과 STOMP — @MessageMapping / @SendTo / SimpMessagingTemplate

**난이도**: 기초

**핵심 키워드**: WebSocket 양방향 커넥션, STOMP 메시지 프로토콜, 목적지 기반 라우팅, @MessageMapping, @SendTo(브로드캐스트), @SendToUser(1:1), SimpMessagingTemplate(서버 능동 push)

**모범 답변 방향**:

WebSocket은 HTTP 핸드셰이크를 통해 한 번 연결이 수립되면 클라이언트와 서버가 양방향으로 자유롭게 메시지를 주고받을 수 있는 프로토콜입니다. HTTP와 달리 연결이 유지되기 때문에 서버가 클라이언트에게 먼저 데이터를 push할 수 있어 실시간 채팅, 알림, 라이브 스트리밍 상태 공유 같은 기능에 적합합니다. 그러나 WebSocket 자체는 단순히 양방향 커넥션만 제공할 뿐, 메시지 포맷이나 라우팅, 구독 관리 체계는 없습니다. 개발자가 직접 메시지 포맷을 정의하고 어떤 클라이언트에게 보낼지 판단하는 로직을 구현해야 합니다. STOMP(Simple Text Oriented Messaging Protocol)는 이 한계를 보완하는 WebSocket 위의 메시지 프로토콜입니다. CONNECT, SEND, SUBSCRIBE, UNSUBSCRIBE 같은 명령 체계와 destination 기반 라우팅, 헤더 기반 인증을 표준으로 정의하기 때문에 Spring과 함께 쓰면 채팅방 같은 구독 구조를 간결하게 구현할 수 있습니다. Spring에서 제공하는 세 가지 어노테이션의 역할 구분이 중요합니다. `@MessageMapping("/send")`는 클라이언트가 `/app/send`로 보낸 메시지를 핸들러 메서드에 매핑하는 역할로, HTTP의 `@RequestMapping`과 유사합니다. `@SendTo("/topic/room-1")`는 메서드의 반환값을 해당 토픽을 구독한 모든 클라이언트에게 브로드캐스트합니다. 여기서 자주 혼동하는 포인트가 있는데, `@SendTo`는 1:1 전송이 아니라 구독자 전체에게 보내는 브로드캐스트입니다. 특정 사용자 1명에게만 전송하려면 `@SendToUser`를 사용해야 합니다. `SimpMessagingTemplate`은 컨트롤러 메서드의 요청-응답 흐름 밖에서 서버가 능동적으로 메시지를 push해야 할 때 사용합니다. 예를 들어 스케줄러가 주기적으로 실시간 가격을 push하거나, 이벤트 핸들러에서 특정 이벤트 발생 시 관련 채팅방 구독자에게 알림을 보내는 경우가 여기에 해당합니다. 샵라이브에서 라이브 스트리밍 중 상품 정보 업데이트를 실시간으로 시청자에게 전달할 때 이와 유사한 서버 push 구조가 필요했습니다.

**WebSocket vs STOMP 차이**:
- WebSocket: 양방향 커넥션만 제공. 메시지 포맷/라우팅/구독 관리 없음. 개발자가 직접 구현 필요.
- STOMP: WebSocket 위의 메시지 프로토콜. CONNECT/SEND/SUBSCRIBE/UNSUBSCRIBE 명령 체계. 목적지(destination) 기반 라우팅. 헤더 기반 인증 지원.

**Spring 어노테이션 역할 정리**:
| | 역할 | 방향 |
|---|---|---|
| `@MessageMapping("/send")` | `/app/send`로 오는 클라이언트 SEND 메시지를 핸들러에 매핑 | 클라이언트 → 서버 |
| `@SendTo("/topic/room-1")` | 처리 결과를 해당 토픽 구독자 전체에게 브로드캐스트 | 서버 → 다수 클라이언트 |
| `@SendToUser` | 특정 사용자(1:1)에게 전송 | 서버 → 1명 |
| `SimpMessagingTemplate` | 컨트롤러 외부(서비스/스케줄러)에서 서버가 능동적으로 push | 서버 → 클라이언트 |

**자주 틀리는 포인트**:
- `@SendTo` ≠ 1:1 전송 → 브로드캐스트(구독자 전체)
- 1:1 전송 = `@SendToUser`
- `SimpMessagingTemplate` = 요청-응답 흐름 밖에서 서버가 직접 메시지 push (스케줄러, 이벤트 핸들러 등)

**면접 세션 피드백 (2026-04-28 3회차)**:
- 잘한 점: WebSocket 라우팅 부재 vs STOMP 목적지 기반 라우팅 차이 정확. 채팅방 ID 기반 구독 개념 올바름.
- 보완: @SendTo를 1:1로 오해 → 실제는 브로드캐스트. @SendToUser가 1:1. SimpMessagingTemplate 역할(서버 능동 push, 컨트롤러 외부) 미설명.
- 점수: 5/10 (@SendTo 역할 오류로 꼬리 질문 "모르겠습니다")
