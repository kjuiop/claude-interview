---
tags: [system-design, architecture, real-time, backend]
related: [redis, kafka, kubernetes, golang, distributed-systems, networking]
---

# System Design — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/system-design/questions]]

---

## 라이브 스트리밍 채팅 아키텍처

### 기술 선택 기준 (동시 10만 명 기준)

| 컴포넌트 | 선택 | 이유 |
|---|---|---|
| 언어 | **Go** | goroutine 2KB, 병렬 WebSocket 처리 특화 |
| 서버간 브로드캐스트 | **[[topics/redis/concepts\|Redis]] pub/sub** | push 기반, 채팅방 = topic, 낮은 지연 |
| 메시지 영속 저장 | **[[topics/mongodb/concepts\|MongoDB]]** | 메타데이터 스키마 유연성, 날짜/채팅방 인덱스 |
| 오케스트레이션 | **[[topics/kubernetes/concepts\|Kubernetes]]** | active connection 기반 HPA, 수평 확장 |

### [[topics/redis/concepts\|Redis]] pub/sub vs Redis Streams

| | pub/sub | Streams |
|---|---|---|
| 메시지 영속성 | 없음 (구독자 부재 시 유실) | 있음 (log 구조로 저장) |
| 재처리 | 불가 | 가능 (XREAD + consumer group) |
| 사용 시나리오 | 실시간성 우선, 유실 허용 | 메시지 손실 불허 시 |

**유실 허용 시 보완 전략**: 클라이언트 재연결 시 MongoDB에서 최근 N개 메시지 fetch

### K8s WebSocket Graceful Shutdown

WebSocket 서버 Pod 교체 시 기존 커넥션 처리:

```yaml
terminationGracePeriodSeconds: 300  # WebSocket 평균 세션 길이 기준
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # endpoints 제거 시간 확보
```

**흐름:**
1. `kubectl rollout` 시작 → readinessProbe 실패
2. Service endpoints에서 Pod 제거 (신규 연결 차단)
3. 기존 WebSocket 커넥션 자연 종료 대기
4. `terminationGracePeriodSeconds` 초과 시 강제 종료

**실무 포인트**: 경매, 결제 등 민감한 WebSocket은 강제 종료 절대 불가 → grace period를 충분히 설정

### HPA Scale-out 기준

- **CPU 기준** (권장): 트래픽 부하와 직접 연동
- **active connection 기준**: Custom Metrics 필요하지만 채팅 서버에 직관적
- **Memory 지양**: memory leak 등으로 선형 증가 가능, 트래픽과 무관한 증가 판별 어려움
- k6 부하 테스트로 p95/p99 latency가 늘어나는 RPS 측정 후 임계치 설정

---

## 10만 동시접속 채팅 서버 심화 설계

### 서버 용량 계산

- WebSocket 연결 1개 ≈ 2~4KB 메모리 (헤더, 버퍼 포함)
- 10만 커넥션 ≈ 200~400MB → 메모리는 병목 아님
- **실제 병목: CPU** (브로드캐스트, pub/sub 오버헤드)
- 실무 기준: 서버 1대당 안정적 처리 용량 ≈ 5만~8만 커넥션
- 10만 명 → 최소 2대, 장애 대비 **3대 이상** 권장

OS 레벨 튜닝 (Linux):
```bash
# 파일 디스크립터 한도 증가 (기본 1024 → 100만)
ulimit -n 1000000
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.ip_local_port_range = 1024 65535
```

### 서버 간 브로드캐스트 옵션 비교

| | Redis pub/sub | Kafka | NATS |
|---|---|---|---|
| 레이턴시 | 1ms 이하 | 10~50ms | 1ms 이하 |
| 메시지 지속성 | 없음 | 있음 | JetStream으로 가능 |
| 구현 복잡도 | 낮음 | 높음 | 중간 |
| 적합한 케이스 | 라이브 채팅(유실 허용) | 일반 채팅(히스토리 필요) | 경량 고속 브로드캐스트 |

**실제 사례:**
- **LINE LIVE**: Redis pub/sub으로 100대+ 채팅 서버 간 브로드캐스트, MySQL은 배치로 영속 저장
- **Slack**: Kafka로 메시지 순서 보장 + 히스토리

### Hot Partition 문제와 해결

**문제**: 인기 채팅방(아이돌 라이브, 스포츠 중계)에 트래픽 집중 → 특정 서버/Redis 인스턴스 과부하

**해결 전략:**

1. **Consistent Hashing**: 채팅방 ID 해싱으로 여러 서버/Redis에 균등 분산
2. **채팅방 자동 분할**: 동시접속 임계치 초과 시 같은 방을 복수의 서브룸으로 분할 (LINE LIVE 방식)
3. **다중 Redis 샤딩**: 채팅방을 Redis 인스턴스별로 분산
   ```
   Redis1: room_id % 3 == 0
   Redis2: room_id % 3 == 1
   Redis3: room_id % 3 == 2
   ```

### 메시지 순서 보장

**Kafka 사용 시**: `room_id`를 key로 설정 → 같은 채팅방 메시지는 항상 같은 파티션 → 파티션 내 순서 보장

**Redis pub/sub 사용 시**: 애플리케이션 레벨에서 Sequence Number 부여
```go
// 메시지 발행 시 atomic 증가 시퀀스 번호 부여
seq := redisClient.Incr(ctx, "room:"+roomID+":seq").Val()
msg := Message{Seq: seq, Content: content}
redisClient.Publish(ctx, "room:"+roomID, marshal(msg))
```
- 클라이언트가 seq 기준으로 재정렬
- 갭 발생 시 누락된 메시지를 DB에서 fetch

### 재연결 시 미수신 메시지 처리

**패턴**: 클라이언트가 마지막 수신 seq를 기억 → 재연결 시 서버에 전달 → 그 이후 메시지를 DB/Redis에서 보충

```
클라이언트: { type: "RECONNECT", lastSeq: 1523, roomId: "room_001" }
서버: MongoDB/Redis에서 seq > 1523인 메시지 최대 500개 fetch → 전송
이후: 실시간 pub/sub 구독 재개
```

**버퍼 전략**:
- 인메모리 circular buffer (최근 500~1000개) → 단기 재연결용
- MongoDB/Redis Streams → 장기 히스토리 조회용
- TTL: 인메모리는 5분, DB는 90일

**재연결 백오프 (Thundering Herd 방지)**:
```
1차 재연결: 500ms
2차: 1000ms + random jitter
3차: 2000ms + random jitter
...최대 30초
```

### Sticky Session vs Stateless 선택

| | Sticky Session | Stateless (Redis 공유) |
|---|---|---|
| 규모 | 소규모(~1만) | 중~대규모 |
| 서버 장애 시 | 해당 서버 클라이언트 전부 재연결 | 다른 서버로 자동 이동 |
| 배포 시 | Rolling 중 재연결 강요 | 자연스러운 재연결 |
| 구현 복잡도 | 낮음 | 높음 |

**실무 권장**: 대규모에서는 Redis에 세션 상태 저장 + 어느 서버에서든 처리 가능한 Stateless 설계

> 출처: https://engineering.linecorp.com/ko/blog/the-architecture-behind-chatting-on-line-live

---

## Vert.x vs Go — 채팅 서버 동시성 모델 비교

| | **Go** | **Vert.x (Java)** |
|---|---|---|
| 동시성 단위 | Goroutine (2KB, M:N 스케줄링) | Verticle (Event Loop, 단일 스레드) |
| 블로킹 모델 | 동기 코드 그대로 작성 가능 | Non-blocking 강제 — blocking 시 Event Loop 블로킹 |
| WebSocket 처리 | goroutine 1개 per connection | Event Loop에서 콜백/Future 체인 |
| JVM 오버헤드 | 없음 | JVM 기동 시간 + GC 압박 |
| 코드 가독성 | 동기 스타일 자연스러움 | Reactive 체인 — 복잡도 높음 |
| 메모리 | 2KB per goroutine | JVM 최소 수백 MB + 스레드 오버헤드 |

**Vert.x의 핵심 위험**: DB/Redis 호출 등 모든 I/O를 non-blocking API로 써야 함. 팀이 이 규칙을 지키지 않으면 Event Loop 블로킹 → 전체 서버 응답 불가.

**Go의 장점**: goroutine이 블로킹되면 스케줄러가 다른 goroutine에 CPU 자동 양도 → 동기 스타일로 써도 안전.

**결론**: 10만 동시접속 채팅에서는 Go 선호. Vert.x는 팀 전체가 Reactive 패턴에 익숙해야 하는 운영 리스크 존재.

---

## 투표/경매 — 메시지 유실 불허 케이스

채팅(유실 허용)과 투표/경매(유실 불허)를 같은 실시간 시스템에서 처리하는 **이중 채널 전략**.

### 채널 분리 아키텍처

```
클라이언트
  ├── 채팅 메시지       → Redis pub/sub          → 빠른 브로드캐스트 (유실 허용)
  └── 투표/경매 이벤트  → Kafka / Redis Streams  → 전달 보장 (유실 불허)
```

### 투표/경매 전달 보장 3가지 핵심

**1. 멱등 처리 (Idempotent)**
- `event_id` (UUID)를 DB에 저장 → 중복 수신 시 무시
```json
{ "event_id": "uuid-xxx", "user_id": 123, "choice": "A" }
```

**2. Kafka exactly-once (투표/경매)**
- `enable.idempotence=true` + `transactional.id` 설정
- Consumer: `isolation.level=read_committed`
- DB 쓰기 + offset commit → Outbox 패턴으로 원자적 처리

**3. ACK 확인 + 재전송**
- 서버가 처리 완료 후 클라이언트에 명시적 ACK
- 클라이언트는 ACK 미수신 시 타임아웃 후 재전송 (멱등키로 중복 방지)

### 실시간 결과 브로드캐스트 분리

```
투표 이벤트 → Kafka → Consumer 집계 → 결과를 Redis pub/sub으로 브로드캐스트
```

- **"내 투표 접수됐나?"** → Kafka (반드시 보장)
- **"현재 투표 현황"** → Redis pub/sub (유실 허용 — 다음 집계로 덮임)

→ 보장이 필요한 **이벤트 처리**와 빠른 **상태 전파**를 레이어로 분리하는 것이 핵심.

---

## 작성 예정

- API 설계 패턴 (REST vs gRPC vs GraphQL)
- 대용량 파일 업로드 시스템 설계
- 분산 스케줄러 설계
- Rate Limiting 전략
