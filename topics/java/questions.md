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

**⚠️ 면접 세션 오개념 (2026-05-04 7회차)**:
- "proceed()를 호출하지 않으면 AOP 함수가 실행되지 않는다" → **오답**
- 정답: `@Around` 어드바이스 코드는 정상 실행되지만 **원본 메서드만 실행되지 않는다** (반환값 null)
- 의도적 활용: 캐시 히트 시 proceed() 생략 후 캐시 값 반환 / 권한 없으면 proceed() 없이 예외 던지기

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

## JPA 영속성 컨텍스트 — 1차 캐시, 변경 감지, 쓰기 지연, flush vs commit, OSIV

**Q. JPA 영속성 컨텍스트의 1차 캐시, 변경 감지(Dirty Checking), 쓰기 지연(Write-Behind)이 각각 어떻게 동작하는지 설명해주세요. flush가 발생하는 시점과 commit과의 차이, OSIV 비활성화 시 LazyInitializationException 원인과 해결 방법도 설명해주세요.**

**난이도:** 기초

**핵심 키워드:** 1차 캐시(동일성 보장), 스냅샷 비교, 쓰기 지연 SQL 저장소, flush ≠ commit, OSIV, LazyInitializationException

**모범 답변 방향:**

- **1차 캐시**: 엔티티 조회 시 영속성 컨텍스트에 저장, 같은 트랜잭션 내 재조회 시 DB 쿼리 없이 캐시 반환. 동일한 식별자 → 동일한 객체 참조(동일성 보장)
- **변경 감지(Dirty Checking)**: 최초 조회 시 스냅샷 저장 → flush 시점에 현재 상태와 비교 → 변경된 필드 자동 UPDATE. 명시적 update 호출 불필요
- **쓰기 지연(Write-Behind)**: persist/변경감지로 생성된 SQL을 쓰기 지연 저장소에 모아뒀다가 flush 시점에 일괄 전송. JDBC 배치 활용 가능. **⚠️ 지연 로딩(Lazy Loading)과 완전히 다른 개념** — 쓰기 지연은 "쓰기 쿼리를 모아서 보내는 것", 지연 로딩은 "연관 엔티티를 접근 시점에 조회하는 것"
- **flush 발생 시점 3가지**: ① 트랜잭션 커밋 직전 ② JPQL 쿼리 실행 직전 ③ 명시적 flush() 호출
- **flush vs commit**: flush = SQL 전송(롤백 가능), commit = 트랜잭션 확정(롤백 불가). commit 내부에서 flush 먼저 실행 후 확정
- **OSIV**: Spring Boot 기본 활성화. ON → Controller까지 영속성 컨텍스트 유지(Lazy Loading 가능). OFF → Service 트랜잭션 종료와 함께 영속성 컨텍스트 닫힘 → Controller에서 Lazy 접근 시 LazyInitializationException. 해결: Service에서 fetch join/EntityGraph로 미리 로딩. 실무에서는 OSIV OFF 권장(DB 커넥션 풀 고갈 방지)

**세션 이력:**
- 2026-05-13 왓챠 1회차: 3/10 — 쓰기 지연을 지연 로딩으로 혼동, flush/commit 역전, OSIV 미답변

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

## 프록시, 동적 프록시, Spring Data JPA Repository의 관계를 연결해서 설명해주세요.

**난이도**: 중급

**핵심 키워드**: 동적 프록시, JDK Dynamic Proxy, InvocationHandler, CGLIB, JpaRepository 인터페이스, 메서드명 분석, JPQL 생성, 리플렉션

**모범 답변 (800자 이상)**:
> 프록시는 실제 객체 대신 앞에 서서 호출을 가로채는 대리 객체입니다. 클라이언트는 프록시를 실제 객체로 알고 호출하지만, 프록시는 그 호출을 가로채 공통 처리를 수행한 뒤 실제 객체에 위임합니다. 이 구조 덕분에 트랜잭션, 로깅, 캐시 같은 횡단 관심사를 비즈니스 로직에서 완전히 분리할 수 있습니다.
>
> 동적 프록시는 컴파일 시점이 아닌 런타임에 프록시 객체를 동적으로 생성하는 방식입니다. Java에는 두 가지 방식이 있습니다. JDK Dynamic Proxy는 `java.lang.reflect.Proxy`와 `InvocationHandler`를 사용해 인터페이스를 구현하는 프록시 클래스를 런타임에 생성합니다. 모든 메서드 호출이 `InvocationHandler.invoke()`로 위임되며, 내부적으로 리플렉션으로 실제 메서드를 실행합니다. 반드시 인터페이스가 있어야 한다는 제약이 있습니다. CGLIB는 대상 클래스를 상속한 서브클래스를 바이트코드 조작으로 생성하는 방식으로, 인터페이스 없이도 프록시를 만들 수 있습니다. Spring Boot 2.x부터는 인터페이스 유무와 무관하게 CGLIB를 기본으로 사용합니다.
>
> Spring Data JPA의 Repository가 이 동적 프록시의 가장 직관적인 사례입니다. 개발자는 `OrderRepository extends JpaRepository<Order, Long>`처럼 인터페이스만 선언하고 구현 클래스를 작성하지 않습니다. 그런데 실제로 `orderRepository.findById(1L)`을 호출하면 SQL이 실행됩니다. 이것이 가능한 이유는 Spring이 애플리케이션 시작 시 JDK Dynamic Proxy로 `OrderRepository` 인터페이스를 구현하는 프록시 객체를 런타임에 생성해 Bean으로 등록하기 때문입니다. `@Autowired`로 주입받는 `orderRepository`는 실제로 이 프록시 인스턴스입니다. 메서드를 호출하면 프록시 안의 `InvocationHandler`가 가로채 메서드 이름을 분석합니다. `findByUserId`이면 `WHERE user_id = ?` 조건의 JPQL을 생성하고, `save`이면 `persist` 또는 `merge`를 호출하는 방식으로 Hibernate에 위임합니다. 개발자가 인터페이스만 선언했는데 DB가 조회되는 "마법"의 실체는 동적 프록시와 리플렉션입니다.
>
> `@Transactional`도 같은 맥락입니다. Repository 메서드 호출 시 CGLIB 프록시가 먼저 트랜잭션을 시작하고, 내부에서 JDK Proxy 기반 Repository가 실제 SQL을 실행한 뒤, 다시 CGLIB 프록시가 커밋 또는 롤백을 처리합니다. 결국 Spring Boot 애플리케이션에서 Repository를 호출하는 단순한 한 줄 코드 뒤에는 CGLIB 프록시와 JDK 동적 프록시, 리플렉션, JPQL 파싱이 모두 맞물려 동작하고 있습니다.

**꼬리 질문 예시**:
- Spring Data JPA에서 인터페이스만 선언해도 동작하는 이유는?
- `findByUserId`처럼 메서드 이름으로 쿼리가 생성되는 원리는?
- Repository에 `@Transactional`을 붙이지 않아도 저장이 되는 이유는?

---

## Spring Boot의 핵심 동작 원리 — IoC/DI, 프록시, 싱글톤을 연결해서 설명해주세요.

**난이도**: 기초

**핵심 키워드**: IoC 컨테이너, DI 생성자 주입, CGLIB 프록시, 싱글톤 스코프, 무상태 설계, race condition, AOP

**모범 답변 (800자 이상)**:
> Spring Boot의 핵심 동작 원리는 IoC/DI, 프록시, 싱글톤 세 가지 축으로 설명할 수 있습니다.
>
> 먼저 IoC(Inversion of Control)는 객체의 생성과 의존성 연결에 대한 제어권을 개발자가 아닌 Spring 컨테이너가 가져간다는 개념입니다. 전통적인 방식에서는 개발자가 `new OrderService(new OrderRepository())`처럼 직접 객체를 생성하고 의존성을 연결했지만, Spring에서는 컨테이너가 애플리케이션 시작 시 Bean을 생성하고 필요한 의존성을 자동으로 주입해줍니다. 이것이 DI(Dependency Injection)입니다. 생성자 주입 방식이 권장되는 이유는 테스트 시 Mock 객체 주입이 쉽고, `final` 필드 선언으로 불변성을 보장할 수 있기 때문입니다. 실무에서 `@Service` 클래스에 생성자 하나만 선언해도 의존성이 주입되는 것이 이 원리로 동작합니다.
>
> 두 번째로 프록시입니다. Spring은 `@Transactional`, `@Cacheable` 같은 어노테이션이 붙은 Bean을 등록할 때 실제 객체 대신 CGLIB 프록시 객체를 생성해 주입합니다. 메서드를 호출하면 프록시가 먼저 가로채 트랜잭션 시작이나 캐시 확인 같은 공통 처리를 수행한 뒤 실제 메서드를 실행합니다. 이것이 AOP의 구현 방식이며, `@Transactional`이 같은 클래스 내 self-invocation에서 동작하지 않는 이유도 이 프록시 구조에서 비롯됩니다. 내부 호출은 프록시를 거치지 않고 실제 객체를 직접 호출하기 때문입니다. 실무에서 Saga 코레오그래피 패턴을 구현할 때 각 플로우 메서드마다 로깅 코드를 중복 작성하는 대신, `@Around`와 커스텀 어노테이션으로 AOP를 구성해 Aspect 한 곳에서 모든 단계의 로깅을 관리한 경험이 있습니다.
>
> 세 번째로 싱글톤입니다. Spring 컨테이너에 등록된 Bean은 기본적으로 싱글톤 스코프로 관리됩니다. 컨테이너당 인스턴스가 하나만 존재하고 모든 요청이 같은 객체를 공유합니다. 덕분에 객체 생성 비용이 줄고 메모리가 절약됩니다. 단, 싱글톤 Bean에 가변 상태를 저장하면 여러 스레드가 동시에 같은 인스턴스를 수정하는 race condition이 발생합니다. 예를 들어 `@Service` 클래스에 인스턴스 변수로 사용자 정보를 저장하면 요청 A의 데이터가 요청 B에 노출될 수 있습니다. 그래서 Bean은 무상태(stateless)로 설계하고 상태는 메서드 파라미터나 ThreadLocal로 관리해야 합니다.
>
> 이 세 가지는 독립된 개념이 아니라 서로 맞물려 동작합니다. IoC 컨테이너가 싱글톤 Bean을 생성하고, 필요한 경우 프록시로 감싸 AOP 기능을 제공하는 구조가 Spring Boot 애플리케이션의 근간입니다.

**꼬리 질문 예시**:
- 싱글톤 Bean에 인스턴스 변수를 두면 어떤 문제가 생기나요?
- `@Transactional`이 같은 클래스 내부 호출에서 동작하지 않는 이유는?
- 생성자 주입이 필드 주입보다 권장되는 이유는?

---

## JPA 엔티티에 기본 생성자가 필요한 이유는 무엇인가요?

**난이도**: 기초

**핵심 키워드**: Reflection, Constructor.newInstance(), 기본 생성자, private 불가, JPA/Hibernate

**모범 답변 (600자 이상 말하기 형태)**:
> JPA 엔티티에 기본 생성자(no-arg constructor)가 필요한 이유는 JPA 구현체(Hibernate 등)가 엔티티 인스턴스를 생성할 때 리플렉션을 사용하기 때문입니다. Hibernate는 DB에서 조회한 ResultSet을 엔티티 객체로 변환할 때 `Constructor.newInstance()`로 파라미터 없이 인스턴스를 먼저 만든 후, 각 필드에 리플렉션(`field.setAccessible(true)` + `field.set(instance, value)`)으로 값을 채워 넣는 방식을 사용합니다. 개발자 입장에서는 명시적으로 기본 생성자를 선언하지 않아도 Java 컴파일러가 자동으로 추가해주지만, 파라미터를 받는 생성자를 하나라도 선언하면 Java는 기본 생성자를 더 이상 자동으로 추가하지 않습니다. 이 경우 Hibernate가 `Constructor.newInstance()`를 호출하면 `NoSuchMethodException`이 발생해 엔티티를 조회할 수 없게 됩니다. 따라서 파라미터 생성자를 선언한 엔티티에는 반드시 기본 생성자를 명시적으로 추가해야 합니다. 접근 제어자 관련해서는 `public` 또는 `protected` 기본 생성자가 필요합니다. `private`은 리플렉션에서 `setAccessible(true)` 없이는 접근이 불가한데, JPA 스펙 자체가 `private` 기본 생성자를 허용하지 않습니다. 실무에서는 Lombok의 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`를 사용해 의도치 않은 직접 생성을 막으면서 JPA 요건도 충족하는 패턴을 많이 씁니다. 이 구조는 리플렉션이 프레임워크 내부에서 어떻게 동작하는지, 그리고 왜 개발자가 JPA 엔티티 작성 시 특정 규약을 지켜야 하는지를 보여주는 대표적인 예입니다.

**꼬리 질문 예시**:
- Hibernate가 `private` 기본 생성자를 허용하지 않는 이유는?
- Lombok `@NoArgsConstructor(access = AccessLevel.PROTECTED)`를 쓰는 이유는?
- CGLIB으로 Lazy Loading 프록시를 생성할 때도 기본 생성자가 필요한가요?

> 출처: https://f-lab.ai/en/insight/understanding-java-reflection

---

## 작성 예정

- JVM GC 알고리즘 (G1GC, ZGC)
- Java Virtual Thread (Project Loom)
- 동시성: synchronized, Lock, atomic

---

## Java Reflection이란 무엇이고, 어떻게 동작하나요?

**난이도**: 기초

**핵심 키워드**: Class 객체, 힙 영역, 런타임 메타정보 조회, setAccessible, 성능 저하, 캡슐화 위반

**모범 답변 (600자 이상 말하기 형태)**:
> 리플렉션은 JVM이 클래스를 로드할 때 힙 영역에 생성하는 `Class<T>` 객체를 통해, 컴파일 시점에 알 수 없는 클래스의 필드·메서드·생성자·어노테이션 정보를 런타임에 동적으로 조회하고 조작할 수 있게 해주는 Java API입니다. JVM 클래스 로더가 `.class` 바이트코드를 로드하면서 클래스당 하나의 `Class` 객체를 힙에 생성하는데, 이것이 리플렉션의 진입점이 됩니다. `getDeclaredMethods()`, `getDeclaredField()` 등으로 메타정보를 꺼낼 수 있고, `setAccessible(true)`를 설정하면 `private` 멤버에도 접근 가능합니다. 실무에서는 프레임워크 레이어에서 주로 사용됩니다. Spring의 `@Autowired` 필드 주입은 리플렉션으로 `private` 필드에 직접 의존성을 주입하고, JPA는 기본 생성자로 인스턴스를 만든 뒤 리플렉션으로 `private` 필드에 DB 조회 값을 설정합니다. Jackson의 JSON 역직렬화도 리플렉션으로 어노테이션을 읽어 필드를 매핑합니다. 단점은 세 가지입니다. 첫째로 성능 저하입니다. JVM이 일반 메서드 호출에 적용하는 인라이닝·JIT 최적화를 리플렉션에는 적용하지 못해 호출 비용이 높습니다. Spring이 리플렉션으로 메서드 정보를 조회한 결과를 캐싱하는 이유가 여기 있습니다. 둘째로 컴파일 타임 타입 안전성이 없어, 잘못된 클래스명이나 메서드명은 런타임에 `ClassNotFoundException`이나 `NoSuchMethodException`으로 나타납니다. 셋째로 `setAccessible(true)`로 `private` 멤버에 접근할 수 있어 캡슐화 원칙이 깨질 위험이 있습니다. Java 9부터는 모듈 시스템이 도입되어 `module-info.java`에 `opens` 선언 없이 외부 모듈에서 리플렉션으로 접근하면 `InaccessibleObjectException`이 발생하도록 보안이 강화되었습니다.

**꼬리 질문 예시**:
- Spring의 `@Autowired` 필드 주입은 내부적으로 어떻게 동작하나요?
- 리플렉션의 성능 문제를 실무에서 어떻게 완화할 수 있나요?
- Java 9 모듈 시스템이 리플렉션에 미친 영향은?

> 출처: https://hudi.blog/java-reflection/
> 출처: https://f-lab.ai/en/insight/understanding-java-reflection

---

## JDK Dynamic Proxy와 CGLIB의 차이를 설명하고, Spring이 CGLIB를 기본으로 선택한 이유는?

**난이도**: 중급

**핵심 키워드**: InvocationHandler, Enhancer/MethodInterceptor, 인터페이스 필수, final 제약, proxyTargetClass, 바이트코드 조작

**모범 답변 (600자 이상 말하기 형태)**:
> JDK Dynamic Proxy와 CGLIB는 모두 런타임에 프록시 객체를 동적으로 생성하지만 메커니즘이 근본적으로 다릅니다. JDK Dynamic Proxy는 `java.lang.reflect.Proxy`와 `InvocationHandler`를 사용합니다. `Proxy.newProxyInstance()`에 ClassLoader, 구현할 인터페이스 목록, InvocationHandler를 넘기면 런타임에 해당 인터페이스를 구현하는 프록시 클래스 바이트코드가 생성됩니다. 이 프록시의 모든 메서드 호출은 `InvocationHandler.invoke(proxy, method, args)`로 위임되고, 내부에서 리플렉션(`method.invoke()`)으로 실제 메서드를 실행합니다. 핵심 제약은 대상 클래스가 반드시 인터페이스를 구현해야 한다는 점입니다. CGLIB는 ASM 바이트코드 라이브러리로 대상 클래스를 **상속**하는 서브클래스를 동적으로 생성합니다. `Enhancer.create()`로 프록시 인스턴스를 만들고, `MethodInterceptor.intercept()`로 메서드 호출을 가로챕니다. `proxy.invokeSuper()`로 부모 메서드를 직접 호출하기 때문에 리플렉션 오버헤드가 없어 JDK Proxy보다 런타임 성능이 유리합니다. 상속 기반이므로 `final` 클래스와 `final` 메서드는 오버라이드 불가하여 프록시 적용이 안 됩니다. Spring Boot 2.x 이전에는 인터페이스가 있으면 JDK Proxy, 없으면 CGLIB를 자동 선택했습니다. 2.x부터는 `spring.aop.proxy-target-class=true`가 기본값이 되어 항상 CGLIB를 사용합니다. 변경 이유는 실무에서 인터페이스 없이 `@Service`만 붙인 클래스가 많아 JDK Proxy 방식에서 AOP가 적용되지 않는 문제가 빈번했고, CGLIB가 더 예측 가능한 동작을 제공하기 때문입니다. 이 구조를 이해하면 `@Transactional`이 `final` 메서드에서 동작하지 않는 이유, self-invocation에서 AOP가 우회되는 이유도 자연스럽게 연결하여 설명할 수 있습니다.

**꼬리 질문 예시**:
- `final` 메서드에 `@Transactional`을 붙이면 왜 동작하지 않나요?
- JDK Dynamic Proxy에서 리플렉션을 사용하는 부분은 어디인가요?
- Spring Boot에서 CGLIB 대신 JDK Proxy로 전환하려면 어떻게 설정하나요?

**⚠️ 면접 세션 추가 포인트 (2026-05-04 7회차)**:
- Spring Boot 2.x CGLIB 기본 이유 추가: 인터페이스 없는 클래스뿐 아니라, 인터페이스가 있어도 `@Autowired MyServiceImpl service`처럼 **구체 타입으로 주입하면 JDK Proxy에서 ClassCastException** 발생. CGLIB는 구체 클래스 서브클래스라 안전.
- 핵심 키워드: `InvocationHandler`(JDK Proxy) vs `MethodInterceptor`(CGLIB)

> 출처: https://medium.com/@JanessaTech/java-dynamic-proxy-jdk-and-cglib-26dbdcab0bf0
> 출처: https://www.kapresoft.com/java/2023/12/28/java-proxy-vs-cglib.html

---

## Reflection과 Dynamic Proxy의 관계를 설명하고, Spring AOP에서 어떻게 함께 사용되나요?

**난이도**: 심화

**핵심 키워드**: Reflection은 Proxy의 구현 수단, InvocationHandler.invoke()의 method.invoke(), CGLIB는 리플렉션 최소화, 메타데이터 조회 vs 실행

**모범 답변 (600자 이상 말하기 형태)**:
> 리플렉션과 Dynamic Proxy는 서로 다른 계층에서 동작하며, Proxy가 리플렉션을 내부 구현 수단으로 활용하는 관계입니다. 리플렉션은 런타임에 클래스의 메타정보를 조회하고 메서드를 동적으로 호출하는 저수준 API이고, Dynamic Proxy는 리플렉션을 기반으로 "모든 메서드 호출을 한 곳에서 가로채는" 고수준 패턴입니다. JDK Dynamic Proxy에서 `InvocationHandler.invoke()`가 호출될 때 두 번째 파라미터로 `java.lang.reflect.Method` 객체가 전달됩니다. 이 `Method` 객체가 리플렉션 API의 일부이며, 핸들러 내에서 `method.invoke(target, args)`를 호출할 때 리플렉션으로 실제 메서드를 실행합니다. 즉 JDK Proxy의 메서드 가로채기 자체는 바이트코드 생성으로 구현되지만, 실제 대상 객체의 메서드 실행은 리플렉션을 사용합니다. Spring AOP에서 이 구조가 명확하게 나타납니다. CGLIB 프록시가 메서드를 가로채면 `TransactionInterceptor.invoke()`가 실행됩니다. 이 인터셉터는 리플렉션으로 메서드에 붙은 `@Transactional` 어노테이션과 속성을 읽어 전파 방식·격리 수준을 파악하고, 트랜잭션 매니저에게 트랜잭션 시작을 요청합니다. CGLIB 기반의 경우 실제 메서드 실행은 `proxy.invokeSuper()`로 부모 메서드를 직접 호출하기 때문에 리플렉션 없이 더 빠르게 동작합니다. 리플렉션은 주로 어노테이션 정보를 읽는 메타데이터 조회 단계에서 사용되고, 조회 결과는 캐싱됩니다. 정리하면, JDK Proxy는 리플렉션에 더 많이 의존하고 CGLIB는 바이트코드 조작으로 리플렉션 의존도를 줄인 것이 두 방식의 핵심 성능 차이입니다.

**꼬리 질문 예시**:
- Spring이 리플렉션으로 읽은 어노테이션 정보를 어떻게 캐싱하나요?
- CGLIB `proxy.invokeSuper()`와 JDK Proxy `method.invoke()`의 성능 차이는 왜 발생하나요?

> 출처: https://www.baeldung.com/java-dynamic-proxies
> 출처: https://medium.com/@AlexanderObregon/the-mechanics-behind-how-java-implements-dynamic-proxy-classes-at-runtime-a8dbc23844e2

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

---

## ConcurrentHashMap

### Q. `HashMap`이 멀티스레드 환경에서 unsafe한 이유와, `Collections.synchronizedMap()`과 `ConcurrentHashMap`의 잠금 방식 차이를 설명하고 각각의 선택 기준을 설명해주세요.

**난이도:** 기초

**핵심 키워드:** race condition, 전체 Map lock, 버킷(배열 슬롯) 단위 lock, CAS, 읽기 lock 없음, null 허용 여부

**모범 답변:**

`HashMap`이 멀티스레드 환경에서 unsafe한 이유는 내부 자료구조(배열 + 연결리스트/트리)에 어떠한 동기화 장치도 없기 때문입니다. 두 스레드가 동시에 같은 버킷에 쓰면 Node 연결이 깨지거나 값이 유실됩니다. Java 8 이전에는 무한 루프(Infinite Loop)가 발생하는 케이스도 있었습니다.

`Collections.synchronizedMap()`은 모든 메서드에 단일 뮤텍스(전체 Map lock)를 겁니다. 스레드 하나가 읽는 동안 다른 모든 스레드는 읽기·쓰기 모두 대기합니다. 읽기가 많은 환경에서 성능 병목이 됩니다.

`ConcurrentHashMap`은 Java 8 이후 **버킷(배열 슬롯) 단위**로만 lock을 겁니다. 쓰기 시 해당 버킷의 첫 번째 Node(헤드)에만 `synchronized`를 걸기 때문에 다른 버킷을 쓰는 스레드끼리는 충돌 없이 병렬 처리됩니다. 빈 버킷에 첫 element 삽입 시 CAS로 처리합니다. `get()` 읽기는 거의 lock 없이 동작합니다.

**선택 기준:**
- 읽기 많은 캐시·카운터 집계 → `ConcurrentHashMap` (기본 선택)
- null 키/값 허용 필요 → `synchronizedMap` (`ConcurrentHashMap`은 null 키/값 불허)
- 레거시 `HashMap` 빠른 thread-safe 교체 → `synchronizedMap`

**꼬리 질문 예시:**
- `ConcurrentHashMap`의 `size()` 메서드는 정확한 값을 보장하나요?
- `putIfAbsent()`가 필요한 경우는 어떤 상황인가요?

**면접 세션 피드백 (2026-04-29 3회차 → 5회차 복습)**:
- 3회차: 차이 전혀 모름 → 2/10
- 5회차: 전체 lock vs CAS·선택 기준 파악. "버킷=8비트" 오류 → 버킷=**배열 슬롯** 교정. 읽기 lock 없음 ✅. 6/10
- 반드시 암기: 버킷 = 배열 인덱스(슬롯), Java 8 이후 버킷 헤드 Node에 synchronized + 빈 버킷은 CAS
- 점수: 5/10 (@SendTo 역할 오류로 꼬리 질문 "모르겠습니다")

## Spring Batch

### Q. Spring Batch의 구조를 설명해주세요. Job, Step, ItemReader, ItemProcessor, ItemWriter의 역할과 chunk-size가 트랜잭션에 미치는 영향을 설명해주세요.

**난이도**: 기초

**핵심 키워드**: Job, Step, ItemReader, ItemProcessor, ItemWriter, Chunk-Oriented Processing, chunk-size, 트랜잭션, JobRepository, Restart

**모범 답변 (말하기 형태)**:
> Spring Batch는 대용량 데이터를 반복 처리하는 배치 작업 프레임워크입니다. Job이 가장 큰 단위이고, Job 안에 여러 Step이 순서대로 실행됩니다. 각 Step은 Chunk-Oriented Processing으로 동작합니다. ItemReader가 데이터를 chunk-size만큼 읽고 → ItemProcessor가 변환/필터링 → ItemWriter가 chunk-size 단위로 한 번에 씁니다.
>
> chunk-size는 트랜잭션 단위와 직결됩니다. chunk-size=100이면 100개 처리 후 커밋, 50번째 실패 시 100개 전체 롤백. 크면 성능 좋지만 재처리 범위 큼. 작으면 안전하지만 트랜잭션 오버헤드 증가.
>
> JobRepository가 실행 상태를 DB에 저장하므로 실패 시 성공한 Step부터 재시작(Restart) 가능합니다.

**구조 요약**:
```
Job
└── Step 1 (Chunk-Oriented)
│     ItemReader → ItemProcessor → ItemWriter
│     [chunk-size 단위로 트랜잭션 커밋]
└── Step 2
└── Step 3

JobRepository: 실행 상태 DB 저장 → Restart 지원
```

**JobInstance vs JobExecution (2026-05-06 세션 추가)**:
- **JobInstance** = Job + JobParameters 조합으로 식별되는 논리적 실행 단위
  - 날짜 파라미터 `date=2026-05-06` → 오늘의 JobInstance
  - 내일 `date=2026-05-07`로 실행 → 새로운 JobInstance
- **JobExecution** = JobInstance의 실제 실행 시도 1회
  - 1개의 JobInstance가 FAILED → 재실행 → 같은 JobInstance에 새 JobExecution 추가
  - **JobInstance : JobExecution = 1:N 관계**
  - 이 구조 덕분에 Spring Batch는 실패한 배치를 재실행할 때 어느 Step부터 이어서 실행할지 추적 가능

**꼬리 질문 예시**:
- chunk-size를 크게 하면 어떤 장단점이 있나요?
- ItemProcessor에서 null을 반환하면 어떻게 되나요? → 해당 아이템을 ItemWriter에 전달하지 않고 스킵
- Spring Batch에서 실패한 배치를 이어서 재실행하려면 어떻게 하나요? → JobRepository의 실행 상태 기반 Restart
- JobInstance와 JobExecution의 차이는? → Job+JobParameters=JobInstance(논리 단위), 실행 시도 1회=JobExecution, 1:N 관계

---

## Spring Batch faultTolerant

### Q. Spring Batch에서 `faultTolerant()`를 사용할 때 `skip`과 `retry`를 각각 어떤 예외 상황에 적용하나요?

**난이도**: 중급

**핵심 키워드**: faultTolerant, skip, retry, skipLimit, retryLimit, DataIntegrityViolationException, TransientDataAccessException, chunk 트랜잭션 단위, 재처리 범위

**모범 답변 (말하기 형태)**:
> faultTolerant()를 사용할 때 skip과 retry는 예외의 성격으로 구분합니다. skip은 재시도해도 성공할 수 없는 영구적 오류에 사용합니다. DataIntegrityViolationException처럼 데이터 자체에 문제가 있으면 아무리 재시도해도 같은 오류가 발생합니다. `faultTolerant().skip(DataIntegrityViolationException.class).skipLimit(10)`처럼 설정하면 해당 아이템을 건너뛰고 다음을 처리합니다. skipLimit 초과 시 Step 전체 실패입니다.
>
> retry는 일시적 오류에 사용합니다. `retry(TransientDataAccessException.class).retryLimit(3)`처럼 설정하면 최대 3회 재시도하고 그래도 실패 시 실패 처리합니다.
>
> chunk-size 트레이드오프도 중요합니다. chunk-size=100이면 100개 단위로 커밋 → 커밋 횟수 감소로 성능 향상. 단 실패 시 100개 전체 롤백 → 단건 재처리로 전환 → **재처리 범위가 커진다**는 단점. 반대로 작으면 재처리 범위는 줄지만 트랜잭션 오버헤드 증가.

**꼬리 질문 예시**:
- `skip`과 `retry`를 같은 예외 클래스에 동시에 설정하면 어떻게 되나요? → retry가 먼저 적용되고, retryLimit 초과 후에 skip 적용
- skipLimit과 retryLimit 초과 시 각각 어떻게 처리되나요?

**면접 세션 피드백 (2026-05-02 1회차)**:
- 잘한 점: skip/retry 케이스 구분 정확, 코드 패턴 정확, chunk 트레이드오프 방향 파악
- 보완: "재처리 범위가 커진다" 표현 추가 필요. "DB 오버헤드"보다 더 명확한 표현.

---

---

## JPA N+1 문제

### Q. JPA에서 N+1 문제가 무엇인지 설명하고, Fetch Join / @EntityGraph / @BatchSize 세 가지 해결 방법의 차이와 적합한 상황을 설명해주세요. Fetch Join과 페이지네이션을 함께 사용할 때 발생하는 문제와 해결 방법도 설명해주세요.

**난이도**: 기초

**핵심 키워드:** N+1, lazy loading, Fetch Join, JOIN FETCH, MultipleBagFetchException, @EntityGraph, @NamedEntityGraph, @BatchSize, IN 절, Hibernate 메모리 페이징, OOM, 지연 조인, 커버링 인덱스

**N+1 정의:**
- 1번의 목록 조회 쿼리 후, 연관 엔티티 접근 시 각 엔티티마다 쿼리 발생
- 100개 Post → 100번 Author 조회 = 총 101번 쿼리

**해결 방법 비교:**

| 방법 | 방식 | 장점 | 단점/주의 |
|---|---|---|---|
| Fetch Join | JPQL `JOIN FETCH` 명시 | 복잡한 쿼리 직접 제어 | 컬렉션 2개+ → `MultipleBagFetchException` |
| @EntityGraph | 선언적 eager 로딩 | 쿼리 작성 불필요, 간단 | 복잡한 조건에서 예상치 못한 쿼리 |
| @BatchSize | IN 절 묶음 조회 | lazy loading 유지하며 N회 → N/size회 | 완전한 1회 조회 아님 |

**@EntityGraph 선언 위치 (두 가지):**
1. Repository 메서드 위: `@EntityGraph(attributePaths = {"author"})`
2. Entity 클래스에: `@NamedEntityGraph(name="...", attributeNodes=...)` 후 Repository에서 참조

**@BatchSize 정확한 효과:**
- "lazy loading 방지" ❌ → "N번을 batch_size로 나눈 횟수로 줄임" ✅
- 100건 + size=10 → 1+10번 쿼리
- `application.yml`: `hibernate.default_batch_fetch_size: 100`

**Fetch Join + Pagination 문제:**
- Hibernate가 전체 JOIN 결과를 메모리에 올린 뒤 페이징 → OOM 위험
- 경고 로그: `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!`

**지연 조인(Deferred Join) 해결:**
1. 페이지네이션이 적용된 ID 목록만 먼저 조회 (커버링 인덱스 활용)
2. 조회된 ID를 IN 절로 넘겨 연관 엔티티 함께 조회

**꼬리 질문 예시:**
- @BatchSize로 설정했을 때 N+1이 완전히 없어지나요?
- 컬렉션 2개를 Fetch Join하면 어떻게 되나요? → MultipleBagFetchException
- 지연 조인에서 커버링 인덱스를 왜 사용하나요?

**면접 세션 피드백 (2026-05-02 2회차)**:
- 잘한 점: N+1 정의 정확, 세 가지 해결책 모두 커버, Fetch Join + Pagination OOM 문제 파악, 지연 조인 + 커버링 인덱스까지 언급 (차별화 포인트)
- 보완: @BatchSize "lazy loading 방지"→ "N을 batch_size로 나눈 횟수로 줄임"으로 표현 교정. @EntityGraph 이중 선언 위치 암기.

---

## Spring @Component vs @Bean / 생성자 주입

### Q. @Component와 @Bean의 차이를 설명하고, 생성자 주입이 @Autowired 필드 주입보다 권장되는 이유를 final 불변성, 순환 참조 감지, 테스트 용이성 관점에서 설명해주세요.

**난이도**: 기초

**핵심 키워드:** @Component, 클래스 레벨, 컴포넌트 스캔, @Bean, 메서드 레벨, @Configuration, 외부 라이브러리, final 불변성, ApplicationContext 초기화, 순환 참조, Mock 주입

**@Component vs @Bean:**

| | @Component | @Bean |
|---|---|---|
| 선언 위치 | 클래스 레벨 | @Configuration 클래스의 메서드 레벨 |
| 등록 방식 | 컴포넌트 스캔 자동 감지 | 메서드 직접 호출로 인스턴스 반환 |
| 주 사용처 | 직접 작성한 클래스 | 외부 라이브러리, 초기화 로직 세밀 제어 필요 시 |

**생성자 주입 권장 이유:**
1. **final 불변성**: 필드를 final로 선언 가능 → 한 번 주입된 의존성 변경 불가, 컴파일러 보장
2. **순환 참조 감지**: **Spring ApplicationContext 초기화 시점**에 즉시 오류 발생 (컴파일/빌드 시점 아님)
3. **테스트 용이성**: Spring 컨텍스트 없이 `new` + Mock 직접 주입 가능

**꼬리 질문 예시:**
- 순환 참조 감지가 컴파일 시점인가요, 런타임인가요? → ApplicationContext 초기화 시점
- @Bean 없이 외부 라이브러리 클래스를 빈으로 등록할 수 있나요?

**면접 세션 피드백 (2026-05-02 3회차)**:
- 잘한 점: @Component/@Bean 사용 케이스 구분, final + Mock + 순환참조 세 가지 언급
- 보완: 순환 참조 감지 시점 = "ApplicationContext 초기화 시점" (컴파일/빌드 아님), @Bean은 메서드 레벨 + @Configuration 안에서 선언

---

## Spring Security JWT 인증

### Q. JWT 기반 인증을 Spring Security에 적용할 때 SecurityFilterChain 구성 방법, JWT 검증 필터 위치, Refresh Token 탈취 방지 전략을 설명해주세요.

**난이도**: 중급

**핵심 키워드:** SecurityFilterChain, csrf.disable(), STATELESS, addFilterBefore, OncePerRequestFilter, UsernamePasswordAuthenticationToken, SecurityContextHolder, HttpOnly Cookie, RTR(Refresh Token Rotation), Redis 무효화

**SecurityFilterChain 핵심 구성 3종 세트:**
```java
http
  .csrf().disable()
  .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
  .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
```

**JWT 필터 흐름 (OncePerRequestFilter):**
1. Authorization 헤더에서 Bearer 토큰 추출
2. 서명 유효성 + 만료 여부 검증
3. 페이로드에서 사용자 식별자 추출 → UserDetailsService로 로드
4. `UsernamePasswordAuthenticationToken` 생성 → `SecurityContextHolder.getContext().setAuthentication()` 직접 저장
5. `filterChain.doFilter()` 로 다음 필터로 이동

> `UsernamePasswordAuthenticationFilter`는 폼 로그인 처리용. JWT 흐름과 무관.

**RTR (Refresh Token Rotation):**
- Refresh Token 사용 시: 기존 토큰 Redis 삭제 + 새 토큰 발급
- 탈취된 토큰이 먼저 사용되면 → 정상 사용자 토큰 이미 무효화 → 재로그인 강제 → 탈취 감지
- Redis 구조: `userId → refreshToken`

**꼬리 질문 예시:**
- JWT 필터 이후에 UsernamePasswordAuthenticationFilter가 다시 비밀번호를 확인하나요? → 아님. JWT 필터가 SecurityContextHolder에 직접 설정
- HttpOnly Cookie 단독으로 탈취를 완전히 방지할 수 있나요? → 아님. RTR 조합 필요

**면접 세션 피드백 (2026-05-02 3회차)**:
- 잘한 점: OncePerRequestFilter JWT 검증, HttpOnly Cookie 방향 파악
- 보완: UsernamePasswordAuthenticationFilter는 JWT 흐름과 무관. JWT 필터가 SecurityContextHolder 직접 설정. RTR 패턴 암기 필수.

---

## @Scheduled 분산 환경 중복 실행 문제

**난이도:** 기초

**핵심 키워드:** JVM 레벨 스케줄러, ShedLock(lockAtMostFor), Redis 분산락(SETNX + TTL), Redisson tryLock, ZooKeeper ephemeral node, 세션 만료 자동 삭제

**모범 답변 방향:**

@Scheduled는 JVM 레벨 스케줄러이므로 N개 인스턴스 → N번 독립 실행.

**구현 전략 (우선순위 순):**

1. **ShedLock** — 가장 많이 쓰는 방식. 별도 인프라 불필요, 기존 DB 테이블 하나로 분산락 구현.
   ```java
   @SchedulerLock(name = "myTask", lockAtMostFor = "10m", lockAtLeastFor = "1m")
   ```
   - `lockAtMostFor`: 크래시 시 최대 락 유지 시간 (TTL 역할) → 자동 해제 보장
   - `shedlock` 테이블의 `lock_until` 컬럼으로 만료 관리

2. **Redis 분산락 (Redisson)** — Redis 인프라가 이미 있을 때.
   ```java
   RLock lock = redissonClient.getLock("myTask");
   lock.tryLock(0, 10, TimeUnit.MINUTES);
   ```
   - `SET lock_key value NX PX {TTL}` 원자적 점유 + TTL 자동 만료
   - Redisson watchdog: 작업 진행 중 TTL 자동 연장

3. **ZooKeeper ephemeral node** — ZooKeeper 인프라가 있을 때.
   - ephemeral node 생성 → 세션 끊기면 자동 삭제 → Watch 이벤트로 다른 인스턴스 감지

**꼬리 질문 예시:**
- 락을 획득한 서버가 crash되면 락이 어떻게 해제되나요? (ShedLock: lockAtMostFor 만료 / Redis: TTL 만료 자동 해제 / ZooKeeper: ephemeral node 세션 만료 자동 삭제)
- ShedLock의 lockAtMostFor와 lockAtLeastFor 차이는?
- Redis TTL을 너무 짧게 설정하면 어떤 문제가 생기나요?

**면접 세션 피드백 (2026-05-02 7회차)**:
- 잘한 점: 중복 실행 원인(JVM 독립 cron), Redis/ZooKeeper 두 전략, 단일 스레드 원자성, ZooKeeper 실무 경험 연결
- 보완: Redis = TTL 만료 자동 해제("키 삭제" 표현 부정확). ZooKeeper = ephemeral node 세션 만료 자동 삭제("다시 등록" 표현 부정확).

---

## Java GC

### Q. Java GC가 동작하는 과정을 Heap 영역(Eden, Survivor, Old Gen) 관점에서 설명하고, Minor GC와 Major GC의 차이, G1GC를 사용하는 이유를 SerialGC/ParallelGC와 비교해서 설명해주세요.

**Heap 구조와 GC 흐름:**
- Young Generation: Eden + Survivor 0 + Survivor 1
- 새 객체 → Eden 할당 → Minor GC 시 참조 확인 → 생존 객체 → Survivor 0 이동
- 반복 후 오래 살아남은 객체 → Old Generation 승격
- Old Generation 가득 참 → Major GC(Full GC) 발생 → 긴 STW

**GC 종류 비교:**

| GC | 스레드 | 특징 | 적합 환경 |
|---|---|---|---|
| SerialGC | 단일 | 처리량 낮음 | 소규모 앱 |
| ParallelGC | 멀티 | 처리량↑, STW 예측 어려움 | 배치 처리 |
| G1GC | 멀티 | STW 예측·제어 가능 | 대용량 Heap 서버 |

**G1GC 핵심 — 암기 필수:**
- Heap을 고정 Young/Old 대신 **동일 크기 Region**으로 분할
- 각 Region의 가비지 비율 추적 → **가비지 가장 많은 Region 우선 수집** = Garbage First
- Region 단위 증분 수집 → STW 시간 예측·제어 가능
- 대용량 Heap(4GB+)에서 Full GC 없이 안정적 응답 시간 유지

**면접 세션 피드백 (2026-05-03 2회차)**:
- 잘한 점: Heap 구조(Eden/Survivor/Old Gen), Minor GC 흐름, Major GC → STW 연결 정확
- 보완: G1GC = Region 분할 + Garbage First(가비지 많은 Region 우선) → STW 예측 가능. SerialGC/ParallelGC 비교 추가 필요.

**면접 세션 피드백 (2026-05-03 5회차 — 재도전)**: **9/10** ✅
- Heap 구조, Minor/Major GC, SerialGC/ParallelGC, G1GC Region + Garbage First 우선 수집 모두 완성
- 보완: G1GC 결론 "STW 시간 예측 가능 → 대용량 Heap 표준 선택" 한 문장 마무리 추가
- 2회차 5/10 → 5회차 9/10 대폭 개선 ✅

---

## @Async ThreadPoolTaskExecutor

**난이도:** 기초

**핵심 키워드:** SimpleAsyncTaskExecutor, corePoolSize, queueCapacity, maxPoolSize, RejectedExecutionException, self-invocation, AOP 프록시

**모범 답변 방향:**

@Async의 기본 executor인 SimpleAsyncTaskExecutor는 요청마다 새 스레드를 생성합니다. 스레드 풀 없이 요청마다 스레드를 만들기 때문에 트래픽이 몰리면 스레드가 무제한으로 늘어나 OOM이 발생할 수 있습니다. 실무에서는 반드시 ThreadPoolTaskExecutor를 직접 설정해야 합니다. 요청이 들어오면 먼저 corePoolSize까지 스레드를 생성합니다. corePool이 가득 차면 queueCapacity까지 큐에 대기시킵니다. 큐도 가득 차면 그때서야 maxPoolSize까지 스레드를 추가합니다. maxPool도 가득 차면 RejectedExecutionException이 발생합니다. 중요한 점은 maxPoolSize 확장 조건이 queueCapacity 포화 이후라는 것입니다. @Async 메서드를 같은 클래스 내에서 this.method()로 호출하면 비동기가 동작하지 않습니다. @Async는 Spring AOP 프록시를 통해 동작하는데, this 참조는 프록시를 우회하고 실제 객체를 직접 호출하기 때문입니다. 해결 방법은 비동기 메서드를 별도 Bean으로 분리하거나 ApplicationContext에서 self-reference를 주입받아 호출하는 것입니다.

**꼬리 질문 예시:**
- queueCapacity를 크게 잡으면 maxPoolSize가 잘 늘어나지 않는 이유는 무엇인가요?
- self-invocation 문제를 해결하는 방법은 무엇인가요?

**면접 세션 피드백 (2026-05-04 3회차)**: 0/10 — 처음 접하는 개념, 전체 보완 필요

---

## Spring Batch Chunk 처리

**난이도:** 기초

**핵심 키워드:** Job, Step, ItemReader, ItemProcessor, ItemWriter, Chunk, 단일 트랜잭션, faultTolerant, skip, retry, JobRepository

**모범 답변 방향:**

Spring Batch에서 Job은 배치 처리 전체 단위이고, Step은 Job을 논리적으로 분리한 실행 단계입니다. 각 Step은 ItemReader, ItemProcessor, ItemWriter로 구성된 Chunk 지향 처리로 구현합니다. Chunk 지향 처리의 핵심은 ItemReader가 chunk size만큼 데이터를 읽고, ItemProcessor가 가공하고, ItemWriter가 한 번에 쓰는 것이 하나의 트랜잭션으로 묶인다는 점입니다. 건당 insert 대신 chunk 단위 배치 insert로 DB 라운드트립을 최소화할 수 있습니다. chunk size가 크면 ItemReader가 그만큼 메모리에 올려두므로 OOM 위험이 있고, 실패 시 롤백 범위도 커집니다. Chunk 처리 중 오류가 발생하면 faultTolerant() 설정에 따라 skip 또는 retry로 처리합니다. skip/retry limit을 초과하면 해당 Step이 실패로 기록되고 JobRepository에 저장됩니다. 이후 재실행 시 성공한 Step은 건너뛰고 실패한 Step부터 재시작할 수 있습니다.

**꼬리 질문 예시:**
- Chunk size 결정 시 메모리 측면에서 고려할 점은?
- skip과 retry는 어떤 기준으로 구분하나요?

**면접 세션 피드백 (2026-05-04 3회차)**: 7/10
- 잘한 점: 전체 구조, skip/retry 구분, JobRepository 재시작
- 보완: Chunk 지향 처리의 read→process→write 단일 트랜잭션 흐름 명시 필요. Chunk size와 OOM 직접 연결 필요.

---

## Spring 멀티 모듈

### Q. Spring에서 멀티 모듈 프로젝트를 구성할 때 모듈을 어떤 기준으로 분리하는지 설명해주세요. core, api, batch, domain 모듈의 의존성 방향을 지키는 이유, Gradle implementation vs api 의존성 설정 차이도 설명해주세요.

**난이도**: 기초
**핵심 키워드**: 변경 빈도/재사용 범위/배포 단위, 의존성 단방향(상위→하위), 순환 참조 금지, implementation(내부 한정), api(전이 노출), 기본은 implementation

**모듈 구조**:
- `core`: 공통 유틸리티, 공통 예외, 공통 DTO — 모든 모듈에서 사용
- `domain`: 비즈니스 엔티티, JPA 레포지토리
- `api`: HTTP 요청 처리 (api → domain → core)
- `batch`: 스케줄러/배치 처리 (batch → domain → core)

**Gradle implementation vs api**:
- `implementation`: 내부에서만 사용, 이 모듈을 의존하는 상위 모듈에서 접근 불가 (전이 차단)
- `api`: 상위 모듈에서도 접근 가능 (전이 노출)
- 원칙: **기본 implementation, 의도적 노출 시에만 api**. api 남발 시 모듈 경계 흐려짐

**면접 세션 피드백 (2026-05-05 3회차)**:
- 잘한 점: 분리 기준, 의존성 방향, 순환 참조 방지 목적 명확
- 보완: implementation vs api는 라이브러리/모듈 구분이 아닌 전이 의존성 노출 여부

---

## Spring WebFlux

### Q. Spring WebFlux에서 event loop 모델이 동작하는 방식을 설명해주세요. Mono/Flux lazy evaluation 특성, blocking I/O 혼용 시 문제와 해결 방법도 설명해주세요.

**난이도**: 기초
**핵심 키워드**: Netty event loop, epoll/kqueue, CPU×2 스레드, I/O 완료 콜백 재개, Mono(0~1)/Flux(0~N), cold publisher, subscribe 전 실행 없음, operator chain, R2DBC, Schedulers.boundedElastic(), 코드 복잡성 트레이드오프

**event loop 동작**:
1. CPU 코어 × 2 개의 event loop 스레드
2. `epoll_wait()`로 수천 개 소켓 동시 감시 (I/O multiplexing)
3. I/O 준비된 소켓만 선택 → 처리 (스레드 블로킹 없음)
4. I/O 완료 이벤트 → **콜백으로 재개** ("잠시 기다리게 하고"가 아님!)

**lazy evaluation (cold publisher)**:
- subscribe() 전까지 아무 연산도 실행되지 않음
- operator chain은 실행 파이프라인 정의일 뿐, 실제 데이터 흐름은 subscribe 이후

**blocking I/O 혼용 문제**:
- JDBC → event loop 스레드 점유 → 해당 스레드가 담당한 모든 요청 블로킹
- 해결: R2DBC (reactive DB driver)로 교체, 또는 `Schedulers.boundedElastic()`으로 별도 스레드 풀 분리

**면접 세션 피드백 (2026-05-05 3회차)**:
- 잘한 점: epoll 기반 event loop, lazy evaluation, 코드 복잡성 트레이드오프
- 오개념: "잠시 기다리게 하고" → I/O 완료 콜백 재개로 수정 필요
- 보완: Mono/Flux 기준(0~1/0~N), R2DBC 필요성, cold publisher 개념

---

## Spring AOP — JDK Dynamic Proxy vs CGLIB

### Q. Spring AOP에서 JDK Dynamic Proxy와 CGLIB의 차이를 설명해주세요. Spring Boot의 기본 선택과 각각의 제약 조건도 함께 설명해주세요.

**난이도**: 기초~중급
**핵심 키워드**: JDK Dynamic Proxy(인터페이스 기반, 대리인 계약), CGLIB(상속 기반, 자식 위장), Spring Boot 기본 CGLIB, final 클래스/메서드 제약, 기본 생성자 필요, self-invocation 문제

**JDK Dynamic Proxy**:
- 조건: 대상 클래스가 반드시 인터페이스를 구현해야 함
- 동작: 인터페이스의 프록시 구현체를 런타임에 동적 생성 (대리인 계약)
- 메서드 호출 → 프록시 → InvocationHandler → 어드바이스 → 실제 메서드

**CGLIB**:
- 조건: 인터페이스 불필요, 클래스만 있으면 됨
- 동작: 바이트코드 조작으로 대상 클래스의 자식 클래스 생성 (자식 위장)
- 제약: `final` 클래스/메서드 프록시 불가 (자식이 override 못함), 기본 생성자 필요 (JPA `@Entity` 연결 이유)

**Spring Boot 기본 CGLIB**:
- Spring Boot 2.0부터 인터페이스 여부와 무관하게 CGLIB 기본 선택
- 이유: 인터페이스 없는 클래스에도 `@Transactional` 적용 가능하게, 개발자 혼란 감소

**self-invocation 문제**:
- 같은 Bean 내부에서 AOP 적용 메서드를 `this.method()`로 호출하면 프록시를 거치지 않음
- 해결: 해당 메서드를 별도 Bean으로 분리하거나 `AopContext.currentProxy()` 사용

**면접 세션 피드백 (2026-05-05)**:
- 핵심 암기: "JDK = 대리인 계약(인터페이스 필요), CGLIB = 자식 위장(상속, final 불가)"
- Spring Boot 기본값이 CGLIB인 이유와 self-invocation 주의사항 연결

---

## Java 버전별 주요 기능을 설명해주세요 (8 / 11 / 17 / 21 / 25)

**난이도**: 기초

**핵심 키워드**: Lambda, Stream API, java.time, HttpClient, Sealed Class, Record, Pattern Matching, Virtual Thread, Compact Object Headers, LTS 주기

**버전별 핵심 변화 요약**:

| 버전 | 출시 | LTS | 핵심 |
|---|---|---|---|
| Java 8 | 2014.03 | - | Lambda, Stream API, Optional, java.time, PermGen→Metaspace |
| Java 11 | 2018.09 | ✅ | var in lambda, HttpClient, Java EE 제거 |
| Java 17 | 2021.09 | ✅ | Sealed Class, Record, Pattern Matching 정식화. Spring Boot 3.x 최소 요건. |
| Java 21 | 2023.09 | ✅ | Virtual Thread(Project Loom) 정식, Structured Concurrency, Record Patterns |
| Java 25 | 2025.09 | ✅ | Compact Object Headers, Generational Shenandoah, Virtual Thread pinning 해결, Structured Concurrency 정식화 |

> 2026년 5월 기준 최신 LTS: **Java 25** (지원 기간 ~2033년)
> Java 21 지원 종료: 2026년 9월 — 신규 프로젝트는 Java 25 권장

**Virtual Thread vs Platform Thread**:
- Platform Thread: OS 스레드와 1:1 매핑. 생성 비용 높음, 스레드 수 수천 개 한계
- Virtual Thread: JVM이 관리하는 경량 스레드. OS 스레드 수십 개로 수백만 Virtual Thread 실행 가능
- I/O 대기 중 Virtual Thread는 OS 스레드를 반납하고 다른 Virtual Thread가 점유 → 스레드 자원 낭비 없음
- Java 21에서 pinning 문제 존재 → Java 25에서 해결 (JEP 491)

**모범 답변 (2513자)**:
> Java는 버전마다 개발 방식이 크게 바뀌었습니다. Java 8이 가장 큰 전환점인데, Lambda와 Stream API가 생기면서 코드 스타일 자체가 달라졌습니다. 예전에는 리스트에서 조건에 맞는 것만 뽑으려면 for문 돌리고 if문으로 걸러야 했는데, 이제는 .filter().map().collect() 한 줄로 표현할 수 있게 됐습니다. 함수형 프로그래밍이 들어온 거라고 보면 됩니다. Optional과 java.time도 이때 추가돼서 null 처리랑 날짜 처리가 훨씬 편해졌습니다. Java 11은 첫 번째 LTS 버전입니다. isBlank()나 strip() 같은 String 메서드가 추가됐고, HttpClient가 표준 라이브러리에 들어와서 외부 라이브러리 없이 HTTP 요청을 보낼 수 있게 됐습니다. Java 17은 Record랑 Sealed Class가 정식으로 들어온 버전입니다. Record는 DTO를 한 줄로 만들 수 있는 거고, Sealed Class는 상속할 수 있는 클래스를 permits로 딱 정해두는 겁니다. Spring Boot 3.x가 Java 17을 최소 요건으로 지정한 것도 이런 언어 개선 때문입니다. Java 21의 가장 큰 변화는 Virtual Thread입니다. 기존 Platform Thread는 OS 스레드와 1대1로 연결돼 있어서, I/O 기다리는 동안에도 OS 스레드를 계속 붙잡고 있었습니다. OS 스레드는 만들기도 비싸고 수천 개가 한계라서 트래픽이 몰리면 스레드 풀이 금방 찼습니다. Virtual Thread는 Carrier Thread라는 OS 스레드 위에 얹혀서 실행되는 경량 스레드입니다. I/O 대기가 생기면 park()가 호출되면서 Carrier Thread에서 내려오고(unmount), 다른 Virtual Thread가 그 자리를 씁니다. I/O가 끝나면 unpark()로 깨어나서 다시 Carrier Thread에 올라탑니다(mount). 이 덕분에 OS 스레드 몇 개만으로 Virtual Thread 수십만 개를 돌릴 수 있습니다. 주의할 점은 ThreadLocal 남용입니다. ThreadLocal은 스레드마다 독립적인 저장공간을 주는 개념입니다. 사용자 정보나 DB 커넥션을 스레드에 붙여놓고 메서드 파라미터 없이 어디서나 꺼내 쓰는 방식인데, Platform Thread 환경에서는 스레드가 수백 개라 ThreadLocal도 수백 개뿐이라 문제없습니다. 그런데 Virtual Thread는 수십만 개를 동시에 띄울 수 있는데, 각 Virtual Thread가 ThreadLocal을 들고 있으면 메모리가 폭발합니다. 또 ThreadLocal은 스레드가 살아있는 동안 GC가 수거를 못합니다. 스레드 풀처럼 재사용하는 구조라면 이전 요청 데이터가 남아서 다음 요청에 오염될 수도 있습니다. InheritableThreadLocal도 문제인데, 부모 Virtual Thread에서 자식 Virtual Thread를 만들 때 ThreadLocal 값을 전부 복사합니다. Virtual Thread를 대량으로 생성하는 환경에서 이 복사 비용이 쌓이면 성능에 영향을 줍니다. 대안이 Java 21에서 Preview로 들어온 ScopedValue입니다. ScopedValue는 특정 코드 블록 안에서만 살아있고, 블록이 끝나면 바로 GC 대상이 됩니다. ThreadLocal처럼 set()으로 값을 갈아끼우는 게 아니라 불변으로 딱 하나의 값을 바인딩합니다. 자식 Virtual Thread와 값을 공유할 때도 복사 없이 같은 참조를 바라보기 때문에 메모리 효율이 훨씬 좋습니다. Spring Boot 3.2부터는 spring.threads.virtual.enabled=true 하나로 활성화할 수 있습니다. 마지막으로 2026년 5월 기준 최신 LTS는 Java 25입니다. 세 가지가 핵심인데요. 첫 번째는 pinning 문제 해결입니다. Java 21에서 synchronized 블록 안에서 I/O가 생기면 Virtual Thread가 Carrier Thread에 고정돼서 unmount가 안 됐습니다. 그러면 그 Carrier Thread가 I/O 동안 묶여서 다른 Virtual Thread가 못 들어오게 됩니다. 레거시 코드에 synchronized가 많으면 Virtual Thread 써도 의미가 없었는데, Java 25에서 이걸 해결했습니다. 두 번째는 Compact Object Headers입니다. 자바 객체는 실제 데이터 외에 헤더가 붙는데, 이 헤더를 기존 12~16바이트에서 8바이트로 줄였습니다. 서버처럼 객체를 수백만 개 쓰는 환경에서는 힙이 줄고 GC가 덜 돌아서 성능이 올라갑니다. 세 번째는 Structured Concurrency 정식화입니다. 여러 Virtual Thread를 하나의 작업 단위로 묶어서, 하나가 실패하면 나머지가 자동으로 취소되는 구조입니다. 기존 CompletableFuture로 이걸 구현하려면 취소 로직을 직접 다 짜야 했는데 훨씬 간단해졌습니다. Java 21 지원이 2026년 9월에 끝나기 때문에 신규 프로젝트라면 Java 25를 쓰는 게 좋습니다.

**꼬리 질문 예시**:
- Virtual Thread를 사용할 때 주의해야 할 점은 무엇인가요? (ThreadLocal, synchronized pinning)
- Java 17이 Spring Boot 3.x의 최소 요건이 된 이유는 무엇인가요?
- Java 25의 Compact Object Headers는 어떤 성능 개선을 가져오나요?

**면접 세션 피드백 (2026-05-06 2회차 — 4/10)**:
- Java 8 Lambda/Stream ✅, Java 17 Sealed/Record ✅
- 취약: Java 11 핵심 기능(HttpClient, Java EE 제거) 미암기, Java 21 Virtual Thread 동작 원리 설명 부족
- Java 25 LTS 출시 — 기존 답변에 Java 25 포함 필요 (2026년 5월 기준 최신 LTS)
- 암기 포인트: "Java 21 지원 2026년 9월 종료 → 신규 프로젝트는 Java 25"

> 출처: https://openjdk.org/projects/jdk/25/
> 출처: https://www.oracle.com/java/technologies/java-se-support-roadmap.html
> 출처: https://www.baeldung.com/java-25-features

---

## 동기(Synchronous)와 비동기(Asynchronous)의 차이, CompletableFuture vs @Async, I/O bound vs CPU bound

**난이도**: 기초

**핵심 키워드**: 동기/비동기, @Async, ThreadPoolTaskExecutor, CompletableFuture, thenApply, thenCombine, I/O bound, CPU bound, 컨텍스트 스위칭

**모범 답변 (990자)**:
> 동기는 요청을 보낸 뒤 응답이 올 때까지 해당 스레드가 블로킹됩니다. 비동기는 작업을 별도 스레드에 위임하고 현재 스레드는 바로 다음 작업으로 넘어갑니다. Java에서 비동기를 구현하는 방법은 크게 두 가지입니다. 첫 번째는 Spring @Async입니다. AOP 기반 어노테이션으로, 메서드를 호출하면 Spring이 내부 TaskExecutor에서 스레드를 꺼내 실행합니다. 실무에서는 기본 SimpleAsyncTaskExecutor 대신 ThreadPoolTaskExecutor를 Bean으로 등록해서 스레드풀 크기를 직접 조정합니다. 반환값이 필요하면 CompletableFuture로 반환하고, 필요 없으면 void로 fire-and-forget 방식으로 씁니다. 두 번째는 CompletableFuture입니다. Java 8에 추가된 비동기 API로, 여러 비동기 작업을 체이닝하거나 결합할 수 있습니다. thenApply로 결과 변환, thenCombine으로 두 Future 결과를 합치는 등 파이프라인 구성이 가능합니다. @Async는 Spring 빈에만 적용 가능하고 CompletableFuture는 순수 Java 어디서나 쓸 수 있다는 차이가 있습니다. I/O 바운드 작업에서 @Async가 유리한 이유는 caller 스레드 해방 때문입니다. DB 조회나 외부 API 호출처럼 오래 걸리는 작업을 별도 스레드에 위임하면, caller 스레드는 I/O 완료를 기다리지 않고 다른 요청을 받을 수 있습니다. 단, 작업 스레드는 여전히 I/O 대기 중 블로킹됩니다. 총 스레드 수를 실제로 줄이려면 WebFlux(NIO) 또는 Virtual Thread가 필요합니다. CPU 바운드 작업에서 비동기가 불리한 이유는 CPU는 대기 시간이 없기 때문입니다. 이미지 압축, 암호화 연산은 CPU를 계속 점유합니다. 비동기로 별도 스레드에 넘겨도 그 스레드가 CPU 코어를 써야 합니다. 코어 수 이상으로 스레드를 늘리면 컨텍스트 스위칭 오버헤드만 추가됩니다. CPU 바운드는 스레드 수를 코어 수에 맞추는 것이 핵심이고, 비동기로 스레드를 많이 만든다고 성능이 올라가지 않습니다.

**꼬리 질문 예시**:
- @Async를 같은 클래스 내에서 this.method()로 호출하면 왜 비동기가 동작하지 않나요?
- CompletableFuture.allOf()와 thenCombine()의 차이는 무엇인가요?
- CPU 바운드 작업에서 적절한 스레드 수는 어떻게 정하나요?

> 출처: 2026-05-06 5회차 세션 피드백

**면접 세션 피드백 (2026-05-07 1회차)**:
- 6/10 (이전 2/10 → 개선 ✅) — caller 스레드 해방 표현, thenApply/thenCombine 키워드 추가 연습 필요

**면접 세션 피드백 (2026-05-11 1회차)**:
- 3/10 — 반환값 유무 구분 부분 ✅, I/O vs CPU bound 이유 ❌, thenApply/thenCompose/thenCombine/allOf 전체 ❌. **핵심 암기**:
  - I/O bound → @Async + 큰 ThreadPoolTaskExecutor (스레드 대기 많음)
  - CPU bound → CompletableFuture + ForkJoinPool.commonPool() (코어 수 제한)
  - thenApply = 동기 변환 (Stream.map)
  - thenCompose = 비동기 체이닝 (Stream.flatMap), CompletableFuture<CompletableFuture<T>> 중첩 방지
  - thenCombine = 두 Future 결과 합치기
  - allOf = N개 모두 완료 대기, 결과는 .join()으로 개별 추출

**면접 세션 피드백 (2026-05-12 1회차)**:
- 7/10 (이전 3/10 → 개선 ✅) — I/O·CPU bound 스레드 풀 전략 ✅, thenCombine/allOf+join() ✅, thenApply/thenCompose 꼬리에서 map/flatMap 교정 ✅
- 남은 보완: 선택 기준 첫 문장을 "스레드 풀 전략"으로 시작, thenApply "동기 변환" 키워드 명시

---

## 싱글톤 패턴 thread-safe 구현 (synchronized → DCL+volatile → enum)

**난이도**: 기초

**핵심 키워드**: synchronized 성능 낭비, double-checked locking, volatile(명령어 재배열 방지), enum(직렬화·리플렉션 공격 방어), Spring Bean 무상태 설계

**모범 답변 (885자)**:
> 싱글톤 패턴은 애플리케이션 전체에서 인스턴스를 단 하나만 생성하고 그 인스턴스를 공유해서 사용하는 디자인 패턴입니다. 가장 직관적인 구현은 synchronized 키워드를 getInstance() 메서드 전체에 붙이는 방식인데, 이 경우 인스턴스가 이미 생성된 이후에도 매번 synchronized 구문을 거쳐야 해서 불필요한 성능 낭비가 생깁니다. 이를 개선한 것이 Double-Checked Locking입니다. 인스턴스가 null인지 먼저 체크하고, null일 때만 synchronized 블록에 진입해서 다시 한 번 null 체크 후 생성합니다. 여기서 volatile 키워드가 반드시 필요합니다. volatile이 없으면 JVM이 명령어 순서를 바꿀 수 있어서, 객체가 완전히 초기화되기 전에 참조가 반환될 수 있습니다. volatile은 메인 메모리에서 직접 읽도록 강제하고, 명령어 재배열을 방지해서 이 문제를 막습니다. 가장 안전한 구현은 enum입니다. JVM이 enum 상수를 단 하나만 생성하도록 보장하기 때문에 synchronized나 volatile 없이도 thread-safe합니다. 또한 Java 직렬화/역직렬화 시 새 인스턴스가 생성되는 문제와 리플렉션으로 private 생성자를 강제 호출하는 공격에도 안전합니다. synchronized나 DCL 방식은 이 두 가지 공격에 취약합니다. Spring Bean이 싱글톤으로 관리될 때는 상태값을 가지지 않도록 설계해야 합니다. 싱글톤 Bean은 애플리케이션 전체에서 하나의 인스턴스를 여러 요청이 공유하기 때문에, 만약 인스턴스 변수로 상태를 저장하면 요청 A가 변경한 상태가 요청 B에 영향을 줄 수 있습니다. 상태가 필요하다면 request 스코프 Bean이나 ThreadLocal을 활용해야 합니다.

**꼬리 질문 예시**:
- double-checked locking에서 volatile 키워드를 붙여야 하는 이유는 무엇인가요?
- enum 싱글톤이 직렬화·리플렉션 공격에 안전한 이유는 무엇인가요?
- Spring Bean 싱글톤에서 상태값을 가지면 어떤 문제가 생기나요?

**면접 세션 피드백 (2026-05-07 1회차)**:
- 7/10 — synchronized→DCL→enum 발전 흐름 정확. volatile 메인메모리 접근 맞음. 보완: volatile의 명령어 재배열 방지 역할, enum의 직렬화·리플렉션 공격 방어 추가 필요

**면접 세션 피드백 (2026-05-11 1회차)**:
- 6/10 — 3가지 방법 커버 ✅, synchronized 성능 문제 ✅, volatile+DCL 패턴 ✅, enum 간결성 ✅. 취약: requestCount++ 비원자 연산 메커니즘(read-modify-write 3단계) 미언급, 해결책(AtomicInteger·ThreadLocal) 미언급. **핵심 암기**: requestCount++는 비원자 연산 → race condition → AtomicInteger 또는 ThreadLocal로 해결.

> 출처: 2026-05-07 1회차, 2026-05-11 1회차 세션

---

## Vert.x

> 관련 개념: [[topics/java/concepts#Vert.x]]
> 출처: 2026-05-09 샵라이브 코드 기반 정리

---

### Q1. Node.js와 Vert.x는 둘 다 Event Loop 기반이라고 하는데, 내부적으로 어떻게 non-blocking I/O를 처리하는지 설명해주세요. Vert.x를 실무에서 사용한 경험이 있다면 어떤 구조로 설계했는지도 함께 설명해주세요.

**핵심 키워드**: libuv(Netty), epoll/kqueue, Verticle 스레드 고정, Event Bus(Anycast/Broadcast), ZooKeeper 레지스트리 + raw TCP, setWorker 블로킹 분리, 인스턴스 수 튜닝, Kotlin Coroutines / Virtual Thread

**모범 답변 방향**:
1. Vert.x = Node.js와 동일한 Event Loop 철학, 구현체는 Netty
2. 네트워크 I/O → epoll/kqueue, 파일 I/O → Netty 내부 스레드풀
3. Node.js 차이: Event Loop 스레드가 CPU 코어 × 2 → 멀티코어 기본 활용
4. Verticle = 하나의 Event Loop 스레드에 고정 → 싱글 스레드 보장, Mutex 불필요
5. Event Bus: Anycast(Request-Reply) vs Broadcast(Pub-Sub) 구분
6. 서버 간 브로드캐스트: ZooKeeper(구독 레지스트리) + Netty raw TCP(실제 전송)
7. 블로킹 코드: setWorker(true)로 Worker Pool 자동 분리, 인스턴스 수 워크로드별 튜닝
8. 발전 방향: Kotlin Coroutines(현재 성숙), Virtual Thread(Vert.x 5.x)

**꼬리 질문 예시**:
- Vert.x Event Loop와 Go GMP 스케줄러의 차이는 무엇인가요?
- 서버 간 브로드캐스트에서 ZooKeeper가 실제로 하는 역할은 무엇인가요?
- Vert.x에서 블로킹 코드를 Event Loop에서 실행하면 어떤 문제가 생기나요?
- Kotlin Coroutines와 Virtual Thread 중 어떤 상황에서 각각을 선택하시겠어요?

**Go와의 핵심 차이 (면접 연결 포인트)**:
- Go: Work Stealing + 런타임 자동 파킹 → 개발자가 블로킹 신경 안 써도 됨
- Vert.x: Verticle 스레드 고정(Work Stealing 없음) → 블로킹 코드는 개발자가 직접 분리
- 공통: 결국 epoll → OS 비동기 I/O로 귀결

---

## Java 동시성 제어

### Q. Java에서 멀티스레드 환경의 동시성 문제를 해결하는 방법들을 설명하고, ReentrantLock을 synchronized 대신 선택하는 기준을 설명해주세요.

**난이도:** 기초

**핵심 키워드:** synchronized 자동 락, volatile 가시성 + 명령어 재배치 방지, ReentrantLock tryLock(timeout), lockInterruptibly, ConcurrentHashMap 버킷 단위 CAS + 읽기 무락, synchronizedMap 전체 락

**모범 답변 방향:**

1. **synchronized**: 모니터 락 자동 획득/해제. 예외 시에도 자동 해제. 단, 타임아웃/인터럽트 처리 불가, 락 실패 시 무한 대기.
2. **volatile**: 메인 메모리 직접 읽기/쓰기 강제 + 명령어 재배치 방지 → 가시성 해결. 복합 연산(i++) 원자성은 미보장 → AtomicInteger 필요.
3. **ReentrantLock 선택 기준**: 무한 대기를 피해야 할 때 → `tryLock(timeout, TimeUnit)`으로 획득 실패 시 포기 후 별도 처리. 대기 중 인터럽트 허용이 필요할 때 → `lockInterruptibly()`.
   - 실무 예: 라이브 방송 재고 차감에서 타임아웃 후 "재고 소진" 응답 즉시 반환.
4. **ConcurrentHashMap vs synchronizedMap**:
   - ConcurrentHashMap: 버킷 단위 CAS + 읽기 무락 → 동시 읽기 처리량 높음.
   - synchronizedMap: 전체 맵 락 → 읽기도 직렬화. 읽기 많은 캐시성 자료구조에 부적합.

**꼬리 질문 예시:**
- ReentrantLock을 synchronized 대신 선택하는 구체적인 상황은 무엇인가요?
- volatile이 복합 연산의 원자성을 보장하지 못하는 이유는 무엇인가요?

**면접 세션 피드백 (2026-05-10 2회차):**
- synchronized/volatile/ConcurrentHashMap 핵심 구조는 정확히 설명
- ReentrantLock 선택 기준(tryLock timeout + lockInterruptibly) 미암기 — 재출제 필요 (5/10)
- 보완 필수: "무한 대기를 피해야 하는 경우 → ReentrantLock, 타임아웃 후 별도 처리 필요 → tryLock()" 한 문장 암기

---

## Virtual Thread (가상 스레드) — Java 21+

### Q. Java의 Virtual Thread가 무엇이고, 기존 플랫폼 스레드와 어떻게 다른지 설명해주세요. Spring Boot에서의 적용 방법과 WebFlux 대비 장점도 포함해주세요.

**난이도:** 중급

**핵심 키워드:** Virtual Thread, Carrier Thread, M:N 매핑, park/unpark, [[topics/java/concepts|JVM]] 스케줄링, Spring Boot 3.2+, CPU-bound pinning, Java 25 synchronized pinning 해결

**모범 답변 방향:**

Virtual Thread는 Java 21에서 정식 도입된 JVM 관리 경량 스레드입니다. 기존 플랫폼 스레드는 OS 스레드와 1:1로 매핑되어 생성 비용이 높고(~1MB 스택) 수천 개 이상 생성하기 어렵습니다. 반면 Virtual Thread는 JVM이 관리하는 경량 스레드로, 소수의 Carrier Thread(플랫폼 스레드) 위에 M:N 매핑으로 스케줄링됩니다. 수십만 개를 생성해도 메모리 부담이 적습니다.

핵심 동작 원리는 park/unpark 메커니즘입니다. Virtual Thread가 I/O 블로킹(DB 쿼리, HTTP 호출 등)을 만나면 Carrier Thread에서 unmount(분리)되고, Carrier Thread는 즉시 다른 Virtual Thread를 mount해서 실행합니다. I/O가 완료되면 Virtual Thread가 다시 사용 가능한 Carrier Thread에 mount되어 실행을 재개합니다. 이 과정이 JVM 레벨에서 자동으로 이루어지므로 개발자는 동기식 코드를 그대로 작성하면서 비동기 수준의 동시성을 얻습니다.

**Spring Boot 적용:** Spring Boot 3.2+에서는 `spring.threads.virtual.enabled=true` 설정 한 줄로 Tomcat의 요청 처리 스레드를 Virtual Thread로 전환할 수 있습니다.

**vs [[topics/system-design/questions#Spring WebFlux vs Spring MVC|WebFlux]]:** WebFlux는 Mono/Flux와 리액티브 연산자로 비동기 파이프라인을 구성해야 하므로 학습 곡선이 높고 디버깅이 어렵습니다. Virtual Thread는 동기식 코드 스타일(순차적 코드)을 유지하면서 동일한 non-blocking 동시성을 달성합니다. 콜백/Flux 체이닝 없이 기존 Spring MVC + JPA 코드를 그대로 활용할 수 있다는 것이 핵심 장점입니다.

**vs @Async:** `@Async`는 별도 스레드 풀(ThreadPoolTaskExecutor)에서 실행되므로 풀 크기가 동시 처리량의 상한이 됩니다. Virtual Thread는 풀 크기 제한 없이 필요한 만큼 생성되므로 스레드 풀 고갈 문제가 원천적으로 해소됩니다.

**주의사항 — CPU-bound 작업에 부적합:**
- CPU-bound 작업(암호화, 대량 연산 등)은 I/O 대기가 없어 park가 발생하지 않음
- Virtual Thread가 Carrier Thread를 계속 점유(pin)하므로 다른 Virtual Thread가 실행되지 못함
- CPU-bound 작업은 기존 플랫폼 스레드 풀에서 처리하는 것이 적합

**synchronized pinning 문제와 Java 25 해결:**
- Java 21~24: `synchronized` 블록 안에서 I/O 블로킹이 발생하면 Carrier Thread에서 unmount되지 못하고 pin됨
- 해결책: `ReentrantLock`으로 교체하면 pin 없이 정상 unmount
- **Java 25:** synchronized 블록에서도 pinning이 발생하지 않도록 JVM 레벨에서 해결됨

**꼬리 질문 예시:**
- "Virtual Thread에서 synchronized를 사용하면 어떤 문제가 생기나요?" → Carrier Thread pinning. Java 25에서 해결.
- "CPU-bound 작업에 Virtual Thread를 쓰면 왜 비효율적인가요?" → park가 발생하지 않아 Carrier Thread를 독점.
- "WebFlux를 이미 쓰고 있는 서비스에서 Virtual Thread로 전환할 이유가 있나요?" → R2DBC/리액티브 파이프라인이 이미 구축됐으면 전환 비용 대비 이점 적음. 신규 프로젝트라면 Virtual Thread가 코드 복잡도 면에서 유리.

**면접 세션 피드백 (2026-05-13 1회차):**
- M:N 매핑, park/unpark unmount 메커니즘, Spring Boot 3.2+ 설정 정확
- vs WebFlux 비교에서 "동기식 코드 스타일로 동일한 동시성" 핵심 포인트 명확
- CPU-bound pinning 주의사항, Java 25 synchronized 해결 언급
- 보완: @Async 대비 스레드 풀 제한 해소 포인트 추가 연습 필요

---
