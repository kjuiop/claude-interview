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

### 💬 면접 답변 형태로 읽기

Mutex와 Channel은 목적이 근본적으로 다릅니다. Mutex는 여러 goroutine이 동일한 메모리 영역에 동시에 접근하는 것을 막기 위한 상호 배제 도구입니다. 캐시, 카운터, 맵처럼 공유 상태를 읽고 쓰는 작업을 보호할 때 적합합니다. 반면 Channel은 goroutine 간에 데이터를 전달하거나 소유권을 이전할 때 사용합니다. Go의 철학인 "공유 메모리로 통신하지 말고, 통신으로 메모리를 공유하라(Do not communicate by sharing memory; instead, share memory by communicating)"는 원칙이 이 구분을 잘 설명합니다.

실무에서는 이 두 가지가 혼용되는 경우도 있습니다. 예를 들어 WebSocket 채팅 서버를 개발할 때 broadcast 경로에 Mutex를 사용했다가 문제를 겪었습니다. 메시지를 보낼 때 모든 커넥션을 순회하며 Mutex Lock을 잡고 write하는 구조였는데, 느린 커넥션 하나가 write 중에 블로킹되면 Lock을 쥔 채로 대기하기 때문에 나머지 모든 커넥션에 대한 메시지 전파까지 함께 멈추는 현상이 발생했습니다. k6로 부하 테스트를 재현해보니 goroutine 스케줄링 대기가 폭증하는 것이 확인됐습니다.

해결 방법은 각 커넥션에 독립적인 channel 기반 메시지 큐를 부여하고, broadcast는 각 채널에 메시지를 넣기만 하는 lock-free 구조로 전환하는 것이었습니다. 각 커넥션은 전담 write goroutine이 채널에서 메시지를 꺼내 WebSocket write를 처리합니다. 이렇게 하면 느린 커넥션이 자기 채널에서 느리게 처리되더라도 다른 커넥션에 전혀 영향을 주지 않습니다. 결과적으로 Latency가 13초에서 103ms로 줄었고, goroutine 종료는 context.Done() 신호를 select에서 수신하는 방식으로 안전하게 처리했습니다. 이 경험을 통해 공유 상태 보호에는 Mutex, 데이터 흐름과 소유권 이전에는 Channel이라는 구분이 더욱 명확해졌습니다.

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

Mutex와 Channel은 목적이 다릅니다. Mutex는 여러 goroutine이 동일한 메모리 영역에 동시에 접근하는 것을 막기 위한 상호 배제 도구입니다. 캐시, 카운터, 맵처럼 공유 상태를 보호할 때 적합합니다. 반면 Channel은 goroutine 간에 데이터를 전달하거나 소유권을 이전할 때 사용합니다. Go의 철학인 "공유 메모리로 통신하지 말고, 통신으로 메모리를 공유하라"는 원칙이 이 구분을 잘 설명합니다. 실제로 WebSocket 채팅 서버에서 broadcast 경로에 Mutex를 사용했다가 느린 커넥션 하나가 Mutex Lock을 쥔 채로 write를 지연시키면서 전체 메시지 전파가 블로킹되는 문제를 겪었습니다. 이를 각 커넥션에 독립적인 channel 기반 메시지 큐를 부여하는 구조로 전환했더니 Latency가 13초에서 103ms로 줄었습니다. 이 경험을 통해 공유 상태 보호에는 Mutex, 데이터 흐름과 소유권 이전에는 Channel이라는 구분이 더욱 명확해졌습니다.

---

**Q. WebSocket 채팅 서버에서 Mutex 경합을 어떻게 해결했나요?** (이력서 심화)

이 구조 전환으로 메시지 지연이 13초에서 103ms로 줄었습니다. 문제의 원인은 broadcast 경로에 단일 Mutex가 있었다는 점입니다. 메시지를 보낼 때 모든 커넥션을 순회하며 Mutex Lock을 잡고 write하는 구조였는데, 느린 커넥션 하나가 write 중에 블로킹되면 Lock을 쥔 채로 대기하기 때문에 나머지 모든 커넥션에 대한 메시지 전파까지 함께 멈추는 현상이 발생했습니다. k6로 부하 테스트를 재현해보니 goroutine 스케줄링 대기가 폭증하는 것이 확인됐습니다. 해결 방법은 각 커넥션에 독립적인 channel 기반 메시지 큐를 부여하고, broadcast는 각 채널에 넣기만 하는 lock-free 구조로 전환하는 것이었습니다. 각 커넥션은 전담 write goroutine이 채널에서 메시지를 꺼내 WebSocket write를 처리합니다. 이렇게 하면 느린 커넥션이 자기 채널에서 느리게 처리되더라도 다른 커넥션에 전혀 영향을 주지 않습니다. 결과적으로 5,000명 동시 접속 환경에서도 CPU spike 없이 안정적인 Latency를 유지할 수 있었습니다.

**꼬리 질문: goroutine 종료는 어떻게 보장했나요?**
- context.Done() 신호를 goroutine 내 select에서 수신
- 커넥션 종료 시 context cancel 호출 → goroutine 안전 종료

---

**Q. 트랜스코더 서버에서 ZooKeeper를 어떻게 활용했나요?**

트랜스코더 서버 개선에서 가장 큰 변화는 상태 감지 방식을 polling에서 이벤트 주도로 전환한 것입니다. 기존에는 70대 트랜스코더의 상태를 1분 주기로 DB와 네트워크를 통해 polling하고 있었는데, 이 방식은 상태 변화가 없어도 주기마다 불필요한 부하를 발생시켰고 감지 지연도 최대 1분이었습니다. 이를 ZooKeeper의 ephemeral node와 Watch 이벤트 기반으로 전환했습니다. 각 트랜스코더가 ZooKeeper에 ephemeral node를 등록하고, 서버 측 goroutine이 Watch 채널을 select로 대기하다가 노드 변화 이벤트가 오면 그때만 처리하는 구조입니다. 덕분에 polling을 완전히 제거했고, 상태 변화를 실시간으로 감지할 수 있게 됐으며, enable/disable 제어도 ZooKeeper 노드 조작만으로 무중단으로 처리할 수 있었습니다.
- 관련 개념: [[topics/zookeeper/concepts]]
