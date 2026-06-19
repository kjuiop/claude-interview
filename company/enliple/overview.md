---
tags: [enliple, 매체플랫폼, 광고서버, 현황]
related: [architecture, study]
---

# 인라이플 — 매체플랫폼팀 현황

## 팀 핵심 목표

> 매체 페이지에서 광고 요청이 들어오면 **500ms 안에 광고를 응답**

## 기술 스택

| 기술 | 역할 |
|------|------|
| HttpServlet (직접 등록) | 광고 노출 핵심 처리 — `web.xml`에 100개 가까이 매핑 |
| Spring DispatcherServlet | 관리 기능 (`/elp/*`) |
| Jersey (JAX-RS) | API 엔드포인트 (`/api/*`, `/script/*`) |
| EhCache | JVM 로컬 캐시 (1계층) |
| Redis Cluster (4개) | 분산 캐시 (2계층) |
| MongoDB | 사용자 성향/행동 데이터 |
| MySQL | 광고 메타데이터 |
| Kafka | 로그/통계 비동기 처리 |
| Elasticsearch | 광고 데이터 검색 |
| Maven + Jenkins | 빌드/배포 |
| Tomcat (외부) | WAR 배포 |

## 애플리케이션 구조

단일 WAR 모놀리식. Maven 멀티모듈 아님.

- 루트 `pom.xml` 하나, 빌드 산출물 하나 (`platform.war`)
- 기능 분리는 Maven 모듈 대신 **Java 패키지**로 처리 (`adgather`, `openrtb`, `platform` 등)
- 환경별 분리는 Maven 프로파일로 **설정 파일만** 교체 (코드는 동일)

```
platform.war
├── com/adgather/   ← 광고 수집/노출 코어
├── com/platform/   ← 광고 플랫폼 메인 (캠페인/소재/매체/RTB 등)
├── com/openrtb/    ← RTB(Real-Time Bidding)
├── com/adsave/     ← 광고 저장
└── com/common/     ← 공통 유틸
```

## 환경 구성

| 환경 | Maven 프로파일 |
|------|--------------|
| 개발 | `-P dev` |
| 운영 | `-P server` |
| OpenRTB | `-P openrtb` |
| 배치 | `-P batch` |

## 진입점 (URL 패턴)

| 방식 | URL | 설명 |
|------|-----|------|
| HttpServlet | `/servlet/*`, `/rtb/*`, `/video/*` 등 | 광고 처리 핵심 |
| Spring DispatcherServlet | `/elp/*` | 관리 기능 |
| Jersey (JAX-RS) | `/api/*`, `/script/*` | API |

주요 URL:

| URL | 역할 |
|-----|------|
| `/servlet/*` | 광고 노출 (배너, 모바일, 키워드, 네이티브 등) |
| `/rtb/*` | OpenRTB 요청 (파트너사별 전용 서블릿) |
| `/video/*` | VAST 비디오 광고 |
| `/asp/*` | 광고 저장 프록시 |
