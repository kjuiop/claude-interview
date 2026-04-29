---
tags: [java, jvm, spring, backend]
related: [kotlin, distributed-systems, mysql]
---

# Java — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/java/questions]]

---

## @Transactional

Spring AOP Proxy 기반으로 동작. 외부에서 빈을 호출할 때만 proxy를 경유하므로 트랜잭션이 적용된다.

```java
// 외부 호출: proxy 경유 → 트랜잭션 적용 O
orderService.createOrder(order);

// 내부 호출: this.saveLog() 직접 호출 → proxy 우회 → 트랜잭션 적용 X
public void createOrder(Order order) {
    saveLog(order); // REQUIRES_NEW 무시됨
}
```

**self-invocation 해결책:**
1. **별도 클래스 분리 (권장)** — `OrderLogService`로 추출하면 외부 빈 호출이 되어 proxy 정상 작동
2. **self-injection** — `@Autowired OrderService self;` 후 `self.saveLog()` 호출. 순환 참조 위험 있음.

**checked exception rollback 주의:**
- 기본: `RuntimeException`(unchecked)만 rollback
- `IOException` 등 checked exception은 rollback 안 됨
- 해결: `@Transactional(rollbackFor = Exception.class)`

**propagation 주요 옵션:**

| 옵션 | 동작 |
|---|---|
| `REQUIRED` (기본) | 기존 트랜잭션 참여, 없으면 새로 생성 |
| `REQUIRES_NEW` | 항상 새 트랜잭션 생성, 기존 트랜잭션 일시 중단 |
| `SUPPORTS` | 기존 트랜잭션 있으면 참여, 없으면 비트랜잭션 |
| `NEVER` | 트랜잭션 없어야 함, 있으면 예외 |

---

## JPA

### N+1 문제

**정의:** 1번의 쿼리로 N개의 엔티티를 조회한 후, 각 엔티티의 연관 데이터를 로드하기 위해 N번의 추가 쿼리가 발생 → 총 **N+1번** 실행.

```java
List<Event> events = eventRepository.findAll(); // 쿼리 1번
for (Event e : events) {
    e.getAuthor().getName(); // 여기서 N번 추가 발생 (lazy loading)
}
// 이벤트 100개 → 총 101번 쿼리
```

**해결책 1: fetch join (JPQL/Querydsl)**
```java
// JPQL
@Query("SELECT e FROM Event e JOIN FETCH e.author")
List<Event> findAllWithAuthor();

// Querydsl
queryFactory.selectFrom(event)
    .join(event.author, author).fetchJoin()
    .fetch();
// → 1번 쿼리로 해결
```

**해결책 2: @EntityGraph**
```java
@EntityGraph(attributePaths = {"author"})
List<Event> findAll();
// → 어노테이션만으로 fetch join 효과, 단순 조회에 적합
```

**fetch join vs @EntityGraph:**

| | fetch join | @EntityGraph |
|---|---|---|
| 방식 | JPQL/Querydsl에 직접 작성 | 메서드 어노테이션 |
| 유연성 | 복잡한 조건/정렬 가능 | 단순 연관 로딩에 적합 |
| 가독성 | 쿼리 명시적 | 선언적, 간결 |
| 복잡 쿼리 | 적합 | 한계 있음 |

**⚠️ fetch join + pagination 주의:**
```java
// HHH90003004 경고 발생 — 메모리에서 페이징 처리 (전체 데이터 로드 후 잘라냄)
@Query("SELECT e FROM Event e JOIN FETCH e.author")
Page<Event> findAll(Pageable pageable); // 위험!
```
해결: `@BatchSize` 또는 쿼리 분리 (ID로 먼저 페이징 → 연관 데이터 별도 조회)

---

## Spring AOP — 핵심 개념

### AOP란?

**Aspect-Oriented Programming** — 여러 모듈에 반복 등장하는 **횡단 관심사(Cross-Cutting Concern)**를 분리해 모듈화하는 프로그래밍 패러다임.

OOP는 비즈니스 로직을 객체로 분리하지만, 로깅·트랜잭션·보안·캐시처럼 여러 객체에 걸친 공통 관심사는 분리하기 어렵다 → AOP로 해결.

**횡단 관심사 예시:**

| 관심사 | 설명 |
|---|---|
| 로깅 | 메서드 호출/응답/실행시간 기록 |
| 트랜잭션 | `@Transactional` 내부가 AOP로 동작 |
| 보안/인증 | 로그인 검증, 권한 체크 공통 처리 |
| 캐시 | `@Cacheable` |
| 성능 측정 | 실행 시간 모니터링 |

### 핵심 용어

| 용어 | 정의 |
|---|---|
| **Aspect** | Advice + Pointcut의 결합체. "어디서(Pointcut) 무엇을 할지(Advice)"를 묶은 모듈 |
| **Advice** | Aspect의 실제 구현 코드. `@Before`, `@After`, `@Around` 등으로 실행 시점 지정 |
| **Pointcut** | Advice를 적용할 JoinPoint를 선별하는 **표현식**. JoinPoint의 부분집합 |
| **JoinPoint** | AOP를 적용할 수 있는 모든 지점. Spring AOP는 **항상 메서드 실행 시점**으로 제한 |
| **Weaving** | Aspect를 타겟 객체에 연결(적용)하는 과정 |
| **Target** | Advice가 적용되는 실제 Bean 객체 |

### Advice 타입 5가지

| 어노테이션 | 실행 시점 | 특징 |
|---|---|---|
| `@Before` | 메서드 실행 **전** | 메서드 실행 자체를 막을 수 없음 (예외 throw는 가능) |
| `@After` | 메서드 실행 **후 (항상)** | 정상/예외 관계없이 항상 실행 (finally와 유사) |
| `@AfterReturning` | 메서드 **정상 반환 후** | 반환값에 접근 가능, 예외 발생 시 미실행 |
| `@AfterThrowing` | 메서드 **예외 발생 후** | 발생한 예외 객체에 접근 가능 |
| `@Around` | 실행 **전후 모두** | 가장 강력. `ProceedingJoinPoint.proceed()`로 실행 제어. 반환값 변경 가능 |

> 실무 원칙: 필요한 최소 Advice 타입을 사용. 캐시 업데이트만 필요하면 `@Around` 대신 `@AfterReturning`.

### Weaving 시점 3가지

| 시점 | 설명 | 특징 |
|---|---|---|
| **컴파일 타임** | AspectJ 컴파일러가 소스 코드 컴파일 시 Aspect 적용 | 가장 빠름. 전용 컴파일러 필요 |
| **로드 타임** | JVM 클래스 로딩 시 바이트코드에 Aspect 삽입 | 특별한 ClassLoader 필요 |
| **런타임** | 실행 중 프록시 객체로 Aspect 적용 | **Spring AOP 기본 방식**. 메서드 레벨만 지원 |

### 실무 로깅 AOP 예시

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();  // 실제 메서드 실행
        long duration = System.currentTimeMillis() - start;
        log.info("{} 실행 시간: {}ms", joinPoint.getSignature(), duration);
        return result;
    }
}
```

### AspectJ vs Spring AOP

| | Spring AOP | AspectJ |
|---|---|---|
| Weaving 시점 | 런타임 (프록시) | 컴파일/로드타임 |
| 지원 JoinPoint | 메서드 실행만 | 필드 접근, 생성자 등 모두 |
| 성능 | 프록시 오버헤드 있음 | 더 빠름 |
| 설정 복잡도 | 간단 (Spring 내장) | 전용 컴파일러 필요 |
| 사용 대상 | 대부분의 Spring 프로젝트 | 극한 성능 / 메서드 외 JoinPoint 필요 시 |

> 출처: https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html
> https://f-lab.ai/en/insight/understanding-spring-aop

---

## Spring AOP 실제 구현 원리

`@Transactional`은 Spring AOP Proxy 위에서 동작한다. "실제로 구현한다면" 아래 구조를 이해해야 한다.

### 전체 호출 흐름

```
[클라이언트 호출]
       ↓
[CGLIB Proxy (Spring Boot 기본)]   ← 실제 Bean 대신 Proxy가 주입됨
       ↓
[TransactionInterceptor.invoke()]  ← AOP Advice
       ↓
[PlatformTransactionManager.getTransaction(TransactionDefinition)]
       ↓  트랜잭션 시작 (con.setAutoCommit(false))
[실제 Bean 메서드 실행]
       ↓
[정상 → commit() / RuntimeException → rollback()]
```

### JDK Dynamic Proxy vs CGLIB

| 항목 | JDK Dynamic Proxy | CGLIB |
|---|---|---|
| 생성 방식 | `java.lang.reflect.Proxy` | 대상 클래스를 **상속** |
| 조건 | 인터페이스 구현 필수 | 인터페이스 없어도 가능 |
| 가로채기 | `InvocationHandler.invoke()` | 메서드 오버라이드 |
| 제약 | 인터페이스 기반만 | `final` 클래스/메서드 불가 |
| Spring Boot 기본 | ❌ | ✅ (2.x부터 기본값) |

Spring Boot 2.x부터 `proxyTargetClass=true`가 기본 → 인터페이스 유무와 무관하게 CGLIB 적용.
`spring.aop.proxy-target-class=false`로 JDK Proxy로 전환 가능.

### 핵심 클래스 구조

```
@Transactional
    └── @EnableTransactionManagement → AOP Proxy 생성
            └── TransactionInterceptor  (MethodInterceptor 구현)
                    └── TransactionAspectSupport  (부모 클래스)
                            └── PlatformTransactionManager  (Strategy Pattern)
                                    ├── DataSourceTransactionManager  (JDBC)
                                    ├── JpaTransactionManager         (JPA)
                                    └── JtaTransactionManager         (분산 트랜잭션)
```

### PlatformTransactionManager 인터페이스 (직접 구현 시)

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition); // 트랜잭션 시작
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

### DataSourceTransactionManager 내부 동작 (JDBC 기준)

```java
// doBegin() — 트랜잭션 시작
Connection con = dataSource.getConnection();
con.setAutoCommit(false);  // ← 핵심: 자동 커밋 비활성화

// doCommit()
con.commit();

// doRollback()
con.rollback();
```

### @Transactional 속성 → TransactionDefinition 매핑

```java
@Transactional(
    propagation = Propagation.REQUIRED,    // 전파 방식 (기본)
    isolation = Isolation.READ_COMMITTED,  // 격리 수준
    timeout = 30,                           // 타임아웃(초)
    readOnly = false,                       // 읽기 전용 여부
    rollbackFor = Exception.class           // rollback 대상
)
```

> 출처: https://www.marcobehler.com/guides/spring-transaction-management-transactional-in-depth
> https://medium.com/@meet2sudhakar/spring-transaction-management-a-deep-dive-into-the-architecture-762b14a81f47

---

## 작성 예정

- JVM 구조 & GC
- 동시성 (Thread, ExecutorService, CompletableFuture)
- Java 17~21 주요 변경사항 (Record, Sealed Class, Virtual Thread)

---

## Java Reflection (리플렉션)

### 개념

Reflection은 JVM이 클래스를 로드할 때 생성하는 `Class` 객체를 통해, **컴파일 타임에 알 수 없는 클래스의 정보(필드, 메서드, 생성자, 어노테이션 등)를 런타임에 조회하고 조작하는 API**다.

클래스 파일(`.class`)은 컴파일 시 바이트코드로 변환되고, JVM 클래스 로더가 이를 로드하면서 **힙 영역에 `Class<T>` 타입 객체를 하나 생성**한다. 이 `Class` 객체가 리플렉션의 진입점이며, 여기서 `getDeclaredMethods()`, `getDeclaredFields()`, `getDeclaredConstructors()` 등을 통해 클래스 메타정보를 꺼낼 수 있다.

```
[소스 파일] → 컴파일 → [바이트코드 .class]
                                 ↓ 클래스 로더
              [힙: Class<Foo> 객체] ← 리플렉션 진입점
```

### 주요 API

| API | 역할 |
|---|---|
| `Class.forName("com.example.Foo")` | 클래스 이름으로 `Class` 객체 로드 |
| `clazz.getDeclaredMethods()` | 해당 클래스 선언 메서드 목록 (상속 제외) |
| `clazz.getDeclaredField("name")` | 특정 필드 조회 |
| `field.setAccessible(true)` | `private` 필드/메서드 접근 허용 |
| `method.invoke(obj, args...)` | 메서드 동적 호출 |
| `constructor.newInstance(args...)` | 생성자로 객체 동적 생성 |

### 실무 활용처

| 활용처 | 설명 |
|---|---|
| **Spring DI** | `@Autowired`로 `private` 필드에도 의존성 주입 (field injection 방식) |
| **Spring AOP** | CGLIB/JDK Proxy가 메서드 메타정보를 리플렉션으로 획득 |
| **JPA** | 엔티티의 `private` 필드에 직접 값 설정 (기본 생성자 + 리플렉션) |
| **JSON 직렬화** | Jackson이 `@JsonProperty` 어노테이션을 리플렉션으로 읽어 필드 매핑 |
| **테스트 유틸** | `private` 메서드 테스트, 의존성 주입 목적 |

### 단점 및 주의사항

| 단점 | 설명 |
|---|---|
| **성능 저하** | 일반 메서드 호출보다 느림. JVM이 바이트코드 최적화(인라이닝 등)를 적용 못함 |
| **컴파일 타임 타입 안전성 없음** | 잘못된 클래스명·메서드명은 런타임에 `ClassNotFoundException`, `NoSuchMethodException` 발생 |
| **캡슐화 위반** | `setAccessible(true)`로 `private` 멤버에 접근 가능 → 의도치 않은 부작용 |
| **보안 제약** | Java 9+ 모듈 시스템에서 `opens` 선언 없이 리플렉션 접근 시 `InaccessibleObjectException` |

### 💬 면접 답변 형태로 읽기

리플렉션은 JVM이 클래스를 로드할 때 힙 영역에 생성하는 `Class` 객체를 통해 런타임에 클래스의 필드·메서드·생성자·어노테이션 정보를 동적으로 조회하고 조작할 수 있게 해주는 Java API입니다. 컴파일 시점에는 어떤 클래스가 사용될지 알 수 없는 프레임워크 레이어에서 특히 유용합니다. 예를 들어 Spring의 `@Autowired` 필드 주입은 `field.setAccessible(true)` 후 `field.set(bean, dependency)`로 `private` 필드에도 직접 의존성을 주입하고, JPA는 엔티티의 기본 생성자로 인스턴스를 만든 뒤 리플렉션으로 `private` 필드에 DB 조회 값을 설정합니다. Jackson의 JSON 역직렬화도 리플렉션으로 `@JsonProperty`를 읽어 필드를 매핑합니다. 단점은 세 가지입니다. 첫째로 성능 문제입니다. JVM이 일반 메서드 호출에 적용하는 인라이닝·JIT 최적화가 리플렉션에는 적용되지 않아 호출 비용이 높습니다. 그래서 Spring은 리플렉션으로 메서드 정보를 조회하되 결과를 캐싱하고, 실제 호출은 최대한 리플렉션을 피하는 방향으로 최적화합니다. 둘째로 컴파일 타임 타입 안전성이 없어 클래스명이나 메서드명 오타가 런타임에 `ClassNotFoundException`이나 `NoSuchMethodException`으로 나타납니다. 셋째로 `setAccessible(true)`로 `private` 멤버에 접근할 수 있어 캡슐화 원칙이 깨질 수 있습니다. Java 9부터는 모듈 시스템이 도입되어 `module-info.java`에 `opens` 선언 없이 외부 모듈에서 리플렉션으로 접근하면 `InaccessibleObjectException`이 발생하도록 제한이 강화되었습니다.

> 출처: https://hudi.blog/java-reflection/
> 출처: https://f-lab.ai/en/insight/understanding-java-reflection

---

## Java Dynamic Proxy — JDK Proxy vs CGLIB (독립 개념)

> Spring AOP 맥락의 프록시 비교는 [[topics/java/concepts#Spring AOP 실제 구현 원리]] 참고.
> 여기서는 두 프록시 메커니즘 자체를 독립적으로 설명한다.

### JDK Dynamic Proxy

`java.lang.reflect.Proxy`와 `InvocationHandler` 인터페이스로 구현되는 **인터페이스 기반 런타임 프록시**.

```
[클라이언트] → [Proxy 인스턴스 (인터페이스 구현체)]
                         ↓ 모든 메서드 호출
               [InvocationHandler.invoke(proxy, method, args)]
                         ↓ 공통 로직 처리 후
               [실제 대상 객체 메서드 호출]
```

**생성 방법:**
```java
MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
    target.getClass().getClassLoader(),   // ClassLoader
    new Class[]{MyInterface.class},        // 구현할 인터페이스 목록
    (p, method, args) -> {                 // InvocationHandler (람다)
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);  // 실제 호출 (리플렉션)
        System.out.println("After: " + method.getName());
        return result;
    }
);
```

**핵심 제약:** 대상 클래스가 반드시 **인터페이스를 구현**해야 한다. 인터페이스 없는 구체 클래스는 프록시 불가.

**내부 동작:** `Proxy.newProxyInstance()`가 런타임에 바이트코드를 생성해 지정된 인터페이스를 구현하는 프록시 클래스를 만들고, 모든 메서드 호출을 `InvocationHandler.invoke()`로 위임한다. 메서드 실행 시 내부적으로 리플렉션(`method.invoke()`)을 사용한다.

### CGLIB (Code Generation Library)

**바이트코드 조작으로 대상 클래스를 상속하는 서브클래스를 동적으로 생성**하는 프록시.

```
[원본 클래스 Foo]
       ↑ 상속
[CGLIB 생성 프록시 Foo$$EnhancerByCGLIB$$xxx]
       ↓ 메서드 오버라이드 → MethodInterceptor.intercept() 호출
```

**핵심 제약:** `final` 클래스나 `final` 메서드는 상속/오버라이드 불가 → 프록시 적용 불가.

**Enhancer + MethodInterceptor:**
```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Foo.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    System.out.println("Before: " + method.getName());
    Object result = proxy.invokeSuper(obj, args);  // 부모 클래스 메서드 직접 호출
    System.out.println("After: " + method.getName());
    return result;
});
Foo proxy = (Foo) enhancer.create();
```

`proxy.invokeSuper()`는 리플렉션 없이 직접 부모 클래스 메서드를 호출하므로 JDK Proxy보다 런타임 성능이 빠르다.

### 비교표

| 항목 | JDK Dynamic Proxy | CGLIB |
|---|---|---|
| 생성 방식 | 인터페이스 구현 (런타임 바이트코드 생성) | 대상 클래스 **상속** (바이트코드 조작) |
| 필요 조건 | **인터페이스 필수** | 인터페이스 불필요 |
| 핸들러 | `InvocationHandler.invoke()` | `MethodInterceptor.intercept()` |
| 메서드 호출 | 리플렉션 (`method.invoke()`) | `proxy.invokeSuper()` (직접 호출) |
| 성능 | 상대적으로 느림 | 상대적으로 빠름 |
| `final` 제약 | `final` 클래스/메서드 가능 (인터페이스 기반) | `final` 클래스/메서드 **불가** |
| Spring Boot 기본 | ❌ | ✅ (2.x부터 기본값) |
| 의존성 | JDK 내장 | 외부 라이브러리 (`spring-core`에 포함) |

### Spring이 CGLIB을 기본으로 선택한 이유

Spring Boot 1.x까지는 인터페이스가 있으면 JDK Proxy, 없으면 CGLIB를 자동 선택했다. 2.x부터 `proxyTargetClass=true`를 기본값으로 바꿔 항상 CGLIB를 사용하도록 변경되었다.

이유: 인터페이스 없이 `@Service`만 붙인 클래스가 많은 실무에서 JDK Proxy는 적용 불가 케이스가 빈번했고, CGLIB가 더 예측 가능한 동작을 제공하기 때문이다.

### 💬 면접 답변 형태로 읽기

JDK Dynamic Proxy와 CGLIB는 모두 런타임에 프록시 객체를 동적으로 생성하는 방식이지만, 생성 메커니즘이 근본적으로 다릅니다. JDK Dynamic Proxy는 `java.lang.reflect.Proxy`와 `InvocationHandler`를 사용해 인터페이스를 구현하는 프록시 클래스를 런타임에 생성합니다. 모든 메서드 호출은 `InvocationHandler.invoke()`로 위임되고, 내부적으로 리플렉션을 통해 실제 메서드를 실행합니다. 인터페이스 기반이므로 대상 클래스가 반드시 인터페이스를 구현해야 한다는 제약이 있습니다. CGLIB는 ASM 바이트코드 라이브러리를 사용해 대상 클래스를 상속하는 서브클래스를 동적으로 생성합니다. `MethodInterceptor.intercept()`로 메서드를 가로채고, `proxy.invokeSuper()`로 부모 메서드를 직접 호출하기 때문에 리플렉션 오버헤드가 없어 JDK Proxy보다 런타임 성능이 빠릅니다. 다만 상속 방식이므로 `final` 클래스나 `final` 메서드는 오버라이드할 수 없어 프록시 적용이 불가합니다. Spring Boot 2.x부터는 `proxyTargetClass=true`가 기본값이 되어 인터페이스 유무에 관계없이 CGLIB를 사용합니다. 실무에서 `@Service`, `@Component`만 붙인 클래스처럼 인터페이스가 없는 경우가 많아 JDK Proxy 기반 설정에서는 AOP가 적용되지 않는 문제가 빈번했기 때문입니다. 이 구조를 이해하면 `@Transactional`이 `final` 메서드에서 동작하지 않는 이유, self-invocation에서 AOP가 우회되는 이유도 자연스럽게 설명할 수 있습니다.

> 출처: https://medium.com/@JanessaTech/java-dynamic-proxy-jdk-and-cglib-26dbdcab0bf0
> 출처: https://www.kapresoft.com/java/2023/12/28/java-proxy-vs-cglib.html
> 출처: https://www.baeldung.com/java-dynamic-proxies
