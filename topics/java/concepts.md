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
