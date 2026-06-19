---
tags: [enliple, 학습, 온보딩, 로드맵]
related: [overview, architecture]
---

# 인라이플 — 공부해야 할 것들

## 온보딩 학습 순서

| 주차 | 주제 | 핵심 목표 |
|------|------|----------|
| 0주차 | 시스템 구조 파악 | 패키지 / 요청 흐름 / WAR 구조 → [[architecture]] |
| 1주차 | HttpServlet | `CommonServlet`, `AdBanner.doGet()` 읽기 |
| 2주차 | 3계층 캐시 | `CommonDaoPattern.retrieveData()` 완전 이해 |
| 3주차 | Redis 클러스터 | `JedisCluster`, 4개 클러스터 구조 |
| 4주차 | MongoDB | 사용자 성향 데이터 흐름, `ConsumerInclinationsMDao` |
| 5주차 | 커넥션풀 | DB/Redis 풀 + S2S HttpClient 풀 |
| 6주차 | 배포 | Jenkins + Maven 프로파일 |

---

## 주제별 공부 포인트

### HttpServlet

- `doGet()` / `doPost()` 흐름
- `HttpServletRequest` — 파라미터, 쿠키, 헤더
- `HttpServletResponse` — 응답 출력, 쿠키 쓰기
- `web.xml` 서블릿 URL 매핑 방식
- 쿠키로 사용자 식별 (AUID)

### EhCache

- JVM 로컬 캐시 동작 원리
- `CommonEHCacheManager` 템플릿 메서드 패턴
- TTL 설정 (`ehcache.xml`)
- `generateKey()` 오버라이드 필수

### Redis 클러스터

- `JedisCluster` 개념 및 사용법
- `JedisPoolConfig` — `maxTotal`, `maxIdle`
- `no reachable node in cluster` 에러 원인
- 4개 클러스터 분리 이유 (`FIRST` / `SECOND` / `THIRD` / `LPM`)

### MongoDB

- Document 기반 NoSQL vs RDB
- AUID 기반 사용자 성향 조회
- `MongoSocketReadTimeoutException` 처리
- Redis/EhCache로 MongoDB 부담 분산하는 이유 (`NoneDbDao` 패턴)

### 커넥션풀

| 개념 | 왜 중요한가 |
|------|------------|
| `maxTotal` / `maxIdle` | 스레드 수 > 커넥션 수면 대기 발생 → 응답 지연 |
| `maxConnPerRoute` | 특정 서버에 커넥션이 몰릴 때 병목 |
| `connectionRequestTimeout` | 풀이 꽉 찼을 때 대기시간 |
| `ConnectTimeout` vs `ReadTimeout` | 장애 대응의 핵심 — 둘 구분 필수 |
| `TCP KeepAlive` | 유휴 커넥션이 방화벽에 끊기는 문제 방지 |
| `TcpNoDelay` | Nagle 알고리즘 비활성 → 광고 응답 지연 최소화 |

### Maven + Jenkins 배포

- `pom.xml` 읽는 법 (의존성, 프로파일)
- 환경별 빌드: `mvn package -P server` / `-P dev` / `-P openrtb`
- Jenkins 빌드 로그에서 실패 원인 찾기
- 어떤 Job이 어떤 서버 그룹에 배포하는지 파악

---

## 검색 키워드

| 주제 | 검색어 |
|------|--------|
| Servlet | `Java HttpServlet doGet 예제` |
| EhCache | `EhCache 2.x 사용법` |
| Redis 클러스터 | `JedisCluster 예제`, `Redis Cluster 개념` |
| MongoDB | `MongoDB Java Driver Document 조회` |
| 커넥션풀 | `Commons Pool2 JedisPool 설정` |
| S2S 커넥션풀 | `Apache HttpClient PoolingHttpClientConnectionManager` |
| RestTemplate | `Spring RestTemplate 커넥션풀 설정` |
| Failover | `HttpClient ConnectTimeout ReadTimeout 차이` |
| Maven | `Maven profile 환경별 빌드` |

---

## 실습 로드맵 (선택)

직접 만들어보고 싶으면:

| Phase | 내용 |
|-------|------|
| 0 | WAR + Tomcat 기반 세팅 |
| 1 | HttpServlet — 파라미터 수신 + HTML 응답 |
| 2 | EhCache — JVM 메모리 캐시 |
| 3 | MySQL + Spring + iBatis |
| 4 | Redis + 3계층 캐시 (`retrieveData()`) |
| 5 | MongoDB — 사용자 성향 타겟팅 |
| 6 | ClickHouse — 노출 로그 적재 |
| 7 | 커넥션풀 튜닝 + 500ms 목표 |
| 8 | Jenkins CI/CD |
