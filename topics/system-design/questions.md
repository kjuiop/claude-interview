---
tags: [system-design, architecture, real-time, interview-questions]
related: [redis, kafka, kubernetes, golang, distributed-systems]
---

# System Design — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/system-design/concepts]]

---

## 대용량 실시간 채팅 시스템 설계

### Q. 동시 시청자 10만 명이 참여하는 라이브 스트리밍 채팅 시스템을 처음부터 설계한다면 어떤 아키텍처를 선택하시겠나요?

**핵심 답변 포인트:**

1. **기술 선택 + 이유**
   - Go: goroutine 2KB, 대규모 WebSocket 커넥션 병렬 처리
   - Redis pub/sub: 서버 다중화 시 채팅방별 브로드캐스트, push 기반 낮은 지연
   - MongoDB: 채팅 메시지 메타데이터 유연성, 날짜/채팅방 인덱스로 조회
   - K8s: active connection 또는 CPU 기반 HPA 수평 확장

2. **Redis pub/sub 트레이드오프 명시**
   - 유실 허용 근거: 라이브 채팅은 실시간성 우선
   - 보완: 재연결 시 MongoDB에서 최근 N개 fetch
   - 유실 불허 시: Redis Streams(`XADD`/`XREAD`) 사용

3. **K8s WebSocket Graceful Shutdown**
   - `terminationGracePeriodSeconds` + `preStop hook`
   - readinessProbe 실패 → endpoints 제거 → 기존 커넥션 대기 → 자연 종료

**꼬리 질문 대비:**
- "Redis pub/sub 메시지 유실을 어떻게 처리하나요?" → 재연결 시 MongoDB fallback, 또는 Redis Streams
- "Pod 교체 시 WebSocket은 어떻게 처리하나요?" → terminationGracePeriodSeconds + preStop hook
- "10만 명을 단일 채팅방 vs 여러 채팅방으로 나눌 때 차이는?" → 단일 채팅방: 모든 서버가 동일 topic 구독, 브로드캐스트 부하 집중

**모범 답변 구조:**
기술 선택 + 이유 → Redis pub/sub(빠른 브로드캐스트, 유실 허용 + MongoDB fallback) → MongoDB(스키마 유연성 + 인덱스) → K8s(HPA + graceful shutdown)

---

## 10만 동시접속 채팅 서버를 설계한다면?

**난이도**: 심화

**핵심 키워드**: WebSocket 수평 확장, Redis pub/sub, Hot Partition, Consistent Hashing, Sequence Number, 재연결, Graceful Shutdown

**모범 답변 방향 (전체 흐름):**

1. **용량 계산 먼저**: 서버 1대 ≈ 5~8만 커넥션 안정 처리. 10만이면 3대(장애 대비). 메모리보다 CPU가 병목.

2. **핵심 아키텍처**:
   - WebSocket 서버 (Go): goroutine 2KB, 대규모 커넥션 병렬 처리
   - Redis pub/sub: 서버 간 브로드캐스트 (채팅방 = topic)
   - MongoDB: 메시지 영속 저장 + 재연결 시 히스토리 fetch
   - K8s: CPU/active connection 기반 HPA + Graceful Shutdown

3. **트레이드오프 명시**:
   - Redis pub/sub → 유실 허용, 재연결 시 MongoDB fallback
   - 히스토리/순서 보장 필요하면 Kafka로 전환

4. **심화 포인트 (꼬리 질문 대비)**:
   - Hot Partition → Consistent Hashing + 채팅방 자동 분할
   - 메시지 순서 → room_id 기반 Kafka 파티셔닝 또는 Sequence Number
   - 재연결 → lastSeq 기억 → MongoDB에서 미수신 메시지 보충 + 지수 백오프

**꼬리 질문 예시:**
- "인기 채팅방에 트래픽이 몰리면 어떻게 하나요?" → Consistent Hashing + 채팅방 분할
- "메시지 순서가 뒤집히는 문제는?" → Kafka room_id 파티셔닝 또는 Sequence Number + 클라이언트 재정렬
- "재연결 시 놓친 메시지는?" → lastSeq 저장 → 재연결 시 서버에 전달 → MongoDB에서 diff fetch
- "서버 대수를 얼마나 준비해야 하나요?" → 5만/서버 기준 × 2배 + 장애 여유 = 최소 3대, HPA로 자동 조절
- "Redis pub/sub vs Kafka 언제 어떻게 선택?" → 유실 허용 라이브 채팅 → Redis, 히스토리/손실 불허 → Kafka, 혼합 가능

**실제 사례:**
- LINE LIVE: Redis pub/sub + MySQL(배치), 채팅방 자동 분할로 라이브 급증 대응
- Slack: Kafka로 메시지 순서 보장 + 히스토리 제공

> 출처: https://engineering.linecorp.com/ko/blog/the-architecture-behind-chatting-on-line-live

---

## 작성 예정

- 대용량 파일 업로드 시스템 설계 질문
- API Rate Limiting 설계 질문
- 분산 스케줄러 설계 질문
