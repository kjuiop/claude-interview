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

OOP는 비즈니스 로직을 객체 단위로 분리하는 데는 탁월하지만, 로깅·트랜잭션·보안·캐시처럼 여러 객체에 걸쳐 반복적으로 등장하는 공통 관심사를 분리하기 어렵습니다. 예를 들어 트랜잭션 처리를 OOP만으로 구현하면 모든 서비스 메서드에 `begin/commit/rollback` 로직이 중복됩니다. AOP는 이런 횡단 관심사(Cross-Cutting Concern)를 Aspect라는 별도 모듈로 분리해 핵심 비즈니스 로직에서 완전히 제거하는 프로그래밍 패러다임입니다. 결과적으로 비즈니스 코드는 순수하게 유지되고, 공통 기능은 Aspect 한 곳에서 관리되어 변경이 필요할 때 Aspect만 수정하면 됩니다. Spring의 `@Transactional`이 AOP 위에서 동작하는 대표적인 예입니다.

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

네 가지 용어는 계층적으로 연결됩니다. JoinPoint는 AOP를 적용할 수 있는 모든 후보 지점으로, Spring AOP에서는 메서드 실행 시점만 해당됩니다. Pointcut은 그 JoinPoint 중 실제로 Advice를 적용할 대상을 선별하는 표현식입니다. `execution(* com.example.service.*.*(..))` 같은 형태로 작성하며, JoinPoint의 부분집합이라고 이해하면 됩니다. Advice는 실제로 실행할 공통 코드로, `@Before`, `@After`, `@Around` 등으로 실행 시점을 지정합니다. Aspect는 이 Advice와 Pointcut을 묶은 모듈 단위로, "어디서(Pointcut) 무엇을(Advice) 실행할지"를 하나의 클래스에 정의한 것입니다. Weaving은 Aspect를 타겟 Bean에 연결하는 과정이며, Spring AOP는 런타임 프록시 방식을 사용합니다. 암기 팁으로는 "JoinPoint = 합류 가능한 모든 지점, Pointcut = 그 중 잘라서 선택한 것"으로 기억하면 혼동을 줄일 수 있습니다.

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

다섯 가지 Advice 타입은 실행 시점과 접근 가능한 정보에서 차이가 납니다. `@Before`는 메서드 실행 전에 동작하며, 실행 자체를 막을 수는 없습니다(예외 throw는 가능). `@AfterReturning`은 메서드가 정상적으로 반환한 후에만 실행되며 반환값에 접근할 수 있고, 예외 발생 시에는 실행되지 않습니다. `@AfterThrowing`은 예외가 발생했을 때만 실행되며 예외 객체에 접근할 수 있습니다. `@After`는 정상/예외 여부와 관계없이 항상 실행되어 Java의 `finally`와 유사합니다. `@Around`는 가장 강력한 타입으로 `ProceedingJoinPoint.proceed()` 호출로 메서드 실행 여부와 시점을 직접 제어할 수 있고 반환값도 변경할 수 있습니다. 실무 원칙은 필요한 최소 타입을 사용하는 것입니다. 단순 파라미터 로깅이면 `@Before`, 메서드 실행 시간 측정처럼 전후 모두 필요하면 `@Around`를 선택합니다. `@Around`는 `proceed()`를 호출하지 않으면 메서드 자체가 실행되지 않기 때문에 실수 가능성이 있어 불필요하게 남용하지 않는 것이 좋습니다.

**꼬리 질문 예시:**
- `@Around`에서 `proceed()`를 호출하지 않으면 어떻게 되나요?
- `@AfterReturning`과 `@Around`에서 반환값을 바꾸는 방법의 차이는?

> 출처: https://www.swiftorial.com/tutorials/backend_framework/spring_framework/spring_aop/best_practices_for_spring_aop

---

## Spring AOP가 동작하지 않는 경우와 해결 방법은?

**난이도:** 중급

**핵심 키워드:** self-invocation, 프록시 우회, this 직접 호출, CGLIB, 별도 클래스 분리

**모범 답변 방향:**

Spring AOP가 동작하지 않는 대표적인 세 가지 케이스가 있습니다. 첫째는 self-invocation입니다. Spring AOP는 CGLIB 프록시 기반으로 동작하는데, 같은 클래스 내에서 `this.메서드()`로 직접 호출하면 프록시를 경유하지 않아 AOP가 적용되지 않습니다. `@Transactional`이 self-invocation에서 동작하지 않는 이유도 동일한 원리입니다. 해결책으로는 별도 클래스로 분리해 외부 빈 호출로 만드는 방법이 권장됩니다. self-injection(`@Autowired` 자기 자신)도 가능하지만 순환 참조 위험이 있습니다. 둘째는 `final` 메서드입니다. CGLIB는 대상 클래스를 상속해 메서드를 오버라이드하는 방식인데, `final` 메서드는 오버라이드가 불가하므로 AOP를 적용할 수 없습니다. 셋째는 `new`로 직접 생성한 일반 객체입니다. Spring Container가 관리하는 Bean이 아니기 때문에 프록시가 생성되지 않아 AOP가 적용되지 않습니다. 반드시 Spring Bean으로 주입받아 사용해야 합니다.

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

`@Transactional`은 Spring AOP Proxy 위에서 동작합니다. 외부에서 빈을 호출할 때는 CGLIB 프록시를 경유해 트랜잭션이 시작되지만, 같은 클래스 내에서 `this.메서드()`로 직접 호출하면 프록시를 우회하므로 트랜잭션이 적용되지 않습니다. 이 경우 별도 클래스로 분리해 외부 빈 호출로 만드는 것이 권장 해결책입니다. 또 한 가지 중요한 주의사항은 checked exception의 롤백 동작입니다. Spring은 기본적으로 `RuntimeException`(unchecked)에 대해서만 롤백하며, `IOException` 같은 checked exception은 기본적으로 롤백하지 않습니다. 예를 들어 PG사 API 호출 중 `IOException`이 발생해도 `rollbackFor`를 설정하지 않으면 결제 레코드가 DB에 커밋되어 실제 결제 실패와 불일치가 생깁니다. `@Transactional(rollbackFor = Exception.class)`로 해결할 수 있습니다. `REQUIRES_NEW`는 현재 트랜잭션과 무관한 새 트랜잭션을 생성해 실패 로그나 감사 기록처럼 메인 트랜잭션이 롤백되어도 반드시 저장해야 하는 경우에 활용합니다.

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

---

## JPA N+1

**Q. JPA에서 N+1 문제가 무엇이고 어떻게 해결하나요?**

**난이도:** 중급

**핵심 키워드:** lazy loading, fetch join, @EntityGraph, N+1 = 1+N번, pagination 주의

**모범 답변 방향:**

N+1 문제는 1번의 목록 조회 쿼리 이후 각 엔티티의 연관 데이터를 로드하기 위해 N번의 추가 쿼리가 발생하는 현상입니다. 이벤트 100개를 조회하면 각 이벤트의 author를 lazy loading으로 접근하는 반복문에서 100번 추가 쿼리가 발생해 총 101번이 실행됩니다. 해결 방법은 크게 세 가지입니다. 첫째로 fetch join은 JPQL에서 `JOIN FETCH`를 사용해 1번 쿼리로 연관 엔티티를 함께 로드하는 방법이며, Querydsl에서도 `.fetchJoin()`으로 동일하게 처리합니다. 둘째로 `@EntityGraph`는 Repository 메서드 위에 선언해 어노테이션만으로 fetch join 효과를 내는 방법으로, 단순 조회에 적합합니다. 셋째로 `@BatchSize`는 N+1을 IN 쿼리로 묶어 처리하는 방법입니다. fetch join 사용 시 가장 중요한 주의사항은 pagination과의 충돌입니다. fetch join에 `Pageable`을 함께 쓰면 Hibernate가 DB에 LIMIT을 적용하지 못하고 전체 데이터를 메모리에 올린 뒤 애플리케이션에서 페이징 처리합니다(HHH90003004 경고). OOM 위험이 있으므로 ID를 먼저 페이지네이션한 후 fetch join으로 2단계 조회하거나 `@BatchSize`를 사용하는 것이 권장됩니다.

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

---

## @Transactional 을 직접 구현한다면 어떻게 해야 하나요?

**난이도:** 심화

**핵심 키워드:** CGLIB, JDK Dynamic Proxy, TransactionInterceptor, PlatformTransactionManager, MethodInterceptor, Strategy Pattern

**모범 답변 방향:**

`@Transactional`을 직접 구현한다면 AOP Proxy와 트랜잭션 매니저 두 축으로 설계해야 합니다. Spring은 `@EnableTransactionManagement`를 통해 대상 Bean을 CGLIB Proxy로 교체합니다. Spring Boot 2.x부터는 기본값이 CGLIB로, 대상 클래스를 상속해 메서드를 오버라이드하는 방식으로 메서드 호출을 가로챕니다. 가로챈 시점에 `TransactionInterceptor.invoke()`가 실행되고, 이 인터셉터는 실제 트랜잭션 처리를 `PlatformTransactionManager`에 위임합니다. 이것이 Strategy Pattern으로, JDBC면 `DataSourceTransactionManager`, JPA면 `JpaTransactionManager` 구현체가 주입됩니다. `DataSourceTransactionManager`는 내부적으로 `con.setAutoCommit(false)`로 트랜잭션을 시작하고, 정상 완료 시 `con.commit()`, `RuntimeException` 발생 시 `con.rollback()`을 호출합니다. 직접 구현한다면 `MethodInterceptor`를 구현해 `PlatformTransactionManager`를 주입받고, `getTransaction()` → 메서드 실행 → `commit/rollback` 흐름으로 작성하면 됩니다.

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
Spring IoC 컨테이너는 객체의 생성과 의존성 주입을 프레임워크가 담당하는 구조입니다. 개발자는 `new`로 직접 객체를 생성하지 않고 선언만 하면 컨테이너가 주입해줍니다. `@Component`는 내가 작성한 클래스에 붙여 컴포넌트 스캔 대상으로 등록합니다(`@Service`, `@Repository`, `@Controller`가 모두 내부적으로 `@Component`). `@Bean`은 `@Configuration` 클래스의 메서드에 붙여 외부 라이브러리 클래스나 복잡한 설정이 필요한 객체를 등록할 때 사용합니다(`DataSource`, `ObjectMapper` 등). Bean의 기본 스코프는 **Singleton** — 컨테이너당 인스턴스 하나, 모든 요청이 같은 객체 공유. 생명주기 콜백: `@PostConstruct`(의존성 주입 완료 직후 실행, 초기화), `@PreDestroy`(소멸 직전 실행, 리소스 해제).

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
