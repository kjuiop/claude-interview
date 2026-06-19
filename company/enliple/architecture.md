---
tags: [enliple, 아키텍처, 요청흐름, 캐시, 서블릿]
related: [overview, study]
---

# 인라이플 — 시스템 아키텍처

## 광고 노출 end-to-end 흐름

```
[1] 광고 태그 JS (매체 페이지)
        │  GET /servlet/AdBanner?s={지면ID}&...
        ▼
[2] XssFilter.doFilter()   ← 모든 요청 통과 (/* 매핑)
        │  - 스레드 ID 생성 & ThreadLocal 세팅
        │  - auId/adId 유효성 방어 (early return 가능)
        │  - 서버 부하 시 디폴트 광고 직접 응답 후 return
        │  - chain.doFilter() → 서블릿 호출
        ▼
[3] CommonServlet.doGet()  ← 모든 광고 서블릿의 부모
        │  쿠키 파싱 / AUID 추출
        ▼
[4] CommonDaoPattern.retrieveData()  ← 3계층 캐시
        │
        ├─ EhCache (JVM 메모리, 수 ms)    ← hit → 즉시 반환
        │   miss ↓
        ├─ Redis Cluster (네트워크, 수~수십 ms)  ← hit → EhCache 저장 후 반환
        │   miss ↓
        └─ MongoDB / MySQL (수십~수백 ms) → EhCache + Redis 저장 후 반환
        ▼
[5] HTML 템플릿 조합 (ConfigServlet.xml에서 igb별 템플릿 선택)
        ▼
[6] iframe HTML 응답 → 브라우저 렌더링 → 광고 노출
        │
[finally] Kafka로 요청 로그 전송 + ThreadLocal 제거
```

## 3계층 캐시 구조 (500ms의 핵심)

```
요청
 ↓
[1] EhCache  — JVM 메모리, 수 ms
 ↓ miss
[2] Redis Cluster  — 네트워크, 수~수십 ms
 ↓ miss
[3] DB  — MongoDB / MySQL, 수십~수백 ms
 ↓
응답 + 상위 캐시에 저장
```

### Empty Key 패턴

DB 조회 결과가 null이면 매 요청마다 DB를 치지 않도록 EhCache에 "이 키는 DB에도 없음" 표시를 5분 TTL로 저장.

### SEARCH_TYPE

| 값 | 의미 | 상위 캐시 저장 |
|----|------|--------------|
| `CACHE` | EhCache hit | 없음 |
| `REDIS` | Redis hit | EhCache에만 저장 |
| `DB` | DB hit | EhCache + Redis 모두 저장 |

## Redis 클러스터 구성 (4개)

| 이름 | 용도 |
|------|------|
| FIRST | 기본 서비스 (`bNewRedis=false`) |
| SECOND | 주요 데이터 기본 (`bNewRedis=true, target=0`) |
| THIRD | 특정 기능 (`target=3`) |
| LPM | LPM(Landing Page Matching) 전용 (`target=18`) |

## S2S 커넥션풀 구조

광고 서버 → 추천 서버 등 내부 API 호출은 Apache HttpClient + RestTemplate 조합.

```
RestTemplateManager
 ├── restTempleteShort  ← 짧은 타임아웃 (광고 추천 등)
 └── restTempleteLong   ← 긴 타임아웃
```

Failover 로직: 서버 목록 shuffle → ConnectTimeoutException 시 다음 서버 재시도, ReadTimeoutException 시 재시도 없이 실패 처리.

## XssFilter가 실제로 하는 일

이름은 XSS 필터지만 실질적으로는 **요청 전처리 미들웨어**.

| 역할 | 내용 |
|------|------|
| 방어 | auId/adId 유효성, 필수 파라미터 체크 |
| 최후의 순간 | 서버 부하 시 디폴트 광고로 대체 응답 |
| ThreadLocal 관리 | 요청 컨텍스트 바인딩 (서블릿 내부 어디서든 꺼내 씀) |
| XSS 처리 | ASCII만 HTML escape, 한글은 그대로 |
| 정리 | 응답 후 Kafka 전송 + ThreadLocal 제거 (메모리 누수 방지) |

## 주요 클래스

| 클래스 | 역할 |
|--------|------|
| `CommonServlet` | 모든 광고 서블릿의 부모 |
| `AdBanner` | 배너 광고 처리 (`GET /servlet/AdBanner`) |
| `CommonDaoPattern` | 3계층 캐시 흐름 전체 |
| `CommonEHCacheManager` | EhCache 추상 클래스 (템플릿 메서드 패턴) |
| `CommonRedisManager` | Redis get/set |
| `RedisConnFactory` | 어떤 Redis 클러스터에 붙을지 결정 |
| `RestTemplateManager` | S2S RestTemplate 2종 관리 |
| `FailoverRestTemplate` | Failover + 클라이언트 사이드 로드밸런싱 |
| `ConfigServlet` | igb별 HTML 템플릿 메모리 보관 + ThreadMap |
| `XssFilter` | 요청 전처리 미들웨어 |
