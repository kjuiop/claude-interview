---
tags: [home, index]
---

# Interview Prep — Home

> 이 파일이 Knowledge Graph의 중앙 허브입니다.
> Obsidian에서 열면 모든 노트의 연결 관계를 그래프로 볼 수 있습니다.

---

## 라이브코딩 연습
- [[live-coding/2026-04-20]] — 채팅방 API (Gin, sync.Map, RWMutex)

---

## Harness Layer
- [[_harness/status]] — 현재 준비 수준, 취약 기술, 우선순위
- [[_harness/constraints]] — 아키텍처 제약, 파일 형식 규칙
- [[_harness/entropy]] — 채워야 할 gaps, 불일치 추적

## 지원자 프로필
- [[resume/profile]] — 기술 스택, 대표 성과, 경력

---

## 지원 공고
- [[jobs/whitecube-challengers/job]] — 화이트큐브 챌린저스 백엔드
- [[jobs/neptune-backend/job]] — 넵튠 솔루션개발실 백엔드
- [[jobs/wag-backend/job]] — 와그(WAUG) 백엔드 개발자 (Java/PHP, 서류통과)
- [[jobs/infobank-ixpert-ai-product-engineer/job]] — 인포뱅크 iXpert AI Product Engineer (Java/Spring/RabbitMQ/LLM)
- [[jobs/buzzni-ax-backend/job]] — 버즈니 에이플러스 AX 백엔드 엔지니어 (AI×이커머스 SaaS, LLM/RAG, 홈쇼핑모아)
- [[jobs/enliple-backend-java/job]] — 인라이플 백엔드 개발자 Java (AD Tech, 라이브방송/데이터파이프라인, ClickHouse/Kafka)
- [[jobs/watcha-backend-media-encoding/job]] — 왓챠 백엔드 개발자 미디어 인코딩 서비스 (Java/Kotlin, RabbitMQ, AWS, OVP)

---

## 기술 지식 베이스

### Backend — Go
- [[topics/golang/overview]] — 언어 특징, 버전 변경사항
- [[topics/golang/goroutine]] — Goroutine, 스케줄러, Leak
- [[topics/golang/channel]] — Channel, Select
- [[topics/golang/context]] — Context, 취소 전파
- [[topics/golang/concurrency]] — Mutex vs Channel, 채팅 서버 사례
- [[topics/golang/interface]] — 암시적 구현, Duck typing
- [[topics/golang/memory]] — GC, Escape Analysis
- [[topics/golang/error-handling]] — 에러 핸들링 패턴
- [[topics/golang/map]] — Map 내부 구조, nil map, concurrent map
- [[topics/golang/lint]] — golangci-lint
- [[topics/golang/clean-architecture]] — Clean Architecture, 레이어 구조, 프로젝트 구조
- [[topics/golang/hexagonal-architecture]] — Hexagonal (Ports & Adapters), Clean Architecture 비교
- [[topics/golang/ddd-modular-monolith]] — DDD Bounded Context, Modular Monolith, 도메인 간 Port/Adapter
- [[topics/golang/gin]] — Gin Middleware, HandlersChain, c.Next()/c.Abort(), 요청 흐름

### Backend — Java / Kotlin
- [[topics/java/concepts]] / [[topics/java/questions]]
- [[topics/kotlin/concepts]] / [[topics/kotlin/questions]]
- [[topics/python-fastapi/concepts]] / [[topics/python-fastapi/questions]]

### Database
- [[topics/mysql/concepts]] / [[topics/mysql/questions]]
- [[topics/postgresql/concepts]] / [[topics/postgresql/questions]]
- [[topics/redis/concepts]] / [[topics/redis/questions]]
- [[topics/mongodb/concepts]] / [[topics/mongodb/questions]]
- [[topics/elasticsearch/concepts]] / [[topics/elasticsearch/questions]]
- [[topics/clickhouse/concepts]] / [[topics/clickhouse/questions]] — OLAP, MergeTree, ORDER BY Primary Key, PARTITION BY

### Messaging & Coordination
- [[topics/kafka/concepts]] / [[topics/kafka/questions]]
- [[topics/rabbitmq/concepts]] / [[topics/rabbitmq/questions]]
- [[topics/zookeeper/concepts]] / [[topics/zookeeper/questions]]
- [[topics/aws/concepts]] / [[topics/aws/questions]] — SNS, SQS, Fan-out 패턴

### Infrastructure
- [[topics/kubernetes/concepts]] / [[topics/kubernetes/questions]]
- [[topics/networking/concepts]] / [[topics/networking/questions]] — HTTP/HTTPS, TLS, 핸드셰이크, HOL Blocking, HTTP/2 Multiplexing, HTTP/3 QUIC

### Architecture
- [[topics/distributed-systems/concepts]] / [[topics/distributed-systems/questions]]
- [[topics/system-design/concepts]] / [[topics/system-design/questions]]
- [[topics/feature-flag/concepts]] / [[topics/feature-flag/questions]] — 피처 플래그, 점진적 롤아웃, Flag Debt

### AI / ML
- [[topics/ai-stt/concepts]] / [[topics/ai-stt/questions]] — Whisper STT, Groq API, video-ai-stt 프로젝트

### Tools & Methodology
- [[topics/claude/concepts]] / [[topics/claude/questions]]
- [[topics/obsidian/concepts]] / [[topics/obsidian/questions]]
- [[topics/knowledge-systems/concepts]] / [[topics/knowledge-systems/questions]]
- [[topics/harness/concepts]] — Harness Engineering 참고 자료 (방법론, Obsidian+Claude 통합)

---

## 기술 관계 맵

```
[golang] ──사용── [goroutine] ──관련── [동시성]
                 [channel]            [분산시스템]
                 [gin]

[kubernetes] ──함께사용── [argocd]
                          [docker]
                          [istio]

[mysql] ──비교── [postgresql]
[kafka] ──비교── [rabbitmq]
[redis] ──관련── [분산락] ──관련── [zookeeper]
```
