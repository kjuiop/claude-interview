---
tags: [golang, go, mutex, sync, concurrency, channel]
related: [goroutine, channel, map]
---

# Golang — 동시성 패턴 (Mutex vs Channel)

→ [[home]] | [[topics/golang/goroutine]] | [[topics/golang/channel]] | [[topics/golang/map]]

---

## Mutex vs Channel

| 상황 | 권장 |
|------|------|
| 공유 상태 보호 (캐시, 카운터) | `sync.Mutex` |
| goroutine 간 데이터 전달 | `channel` |
| 소유권 이전 | `channel` |
| 결과 수집 | `channel` |

실무 패턴:
```go
// Mutex: 공유 상태 보호
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

// Channel: goroutine 간 통신 (이력서의 채팅 서버 개선 사례)
// Mutex 기반 broadcast → channel 기반 per-connection queue로 전환
// → Latency 13초에서 103ms로 개선
```

---

## 면접 질문

**Q. Mutex와 Channel을 어떻게 구분해서 사용하나요?**
- Mutex: 공유 상태 보호 (캐시, 카운터, 맵)
- Channel: goroutine 간 데이터 전달, 소유권 이전, 결과 수집
- 실무 경험: 채팅 서버에서 broadcast에 Mutex를 사용했다가 경합으로 Latency 13초 → channel 기반 per-connection queue로 전환 → 103ms

---

**Q. WebSocket 채팅 서버에서 Mutex 경합을 어떻게 해결했나요?** (이력서 심화)
- 문제: broadcast 경로에 Mutex Lock → 느린 커넥션 하나가 전체 전파 블로킹
- 분석: k6 부하 테스트로 재현, goroutine 스케줄링 증가 확인
- 해결: 각 커넥션에 독립적인 channel 기반 메시지 큐 부여 → lock-free broadcast
- 결과: Latency 13초 → 103ms (126배), 5,000명 동시 접속에서도 CPU spike 없음
- **답변 시작은 반드시 수치로**: "이 구조 전환으로 메시지 지연이 13초에서 103ms로 줄었습니다"

**꼬리 질문: goroutine 종료는 어떻게 보장했나요?**
- context.Done() 신호를 goroutine 내 select에서 수신
- 커넥션 종료 시 context cancel 호출 → goroutine 안전 종료

---

**Q. 트랜스코더 서버에서 ZooKeeper를 어떻게 활용했나요?**
- 문제: 70대 트랜스코더의 상태를 1분 주기 polling → 불필요한 네트워크/DB 부하
- 해결: ZooKeeper ephemeral node + Watch 이벤트 → goroutine이 Watch channel을 select로 대기
- 결과: polling 제거, 상태 변화 시에만 처리, 무중단 enable/disable 제어 가능
- 관련 개념: [[topics/zookeeper/concepts]]
