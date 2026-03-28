---
tags: [golang, go, channel, select, concurrency]
related: [goroutine, context, concurrency]
---

# Golang — Channel & Select

→ [[home]] | [[topics/golang/goroutine]] | [[topics/golang/context]] | [[topics/golang/concurrency]]

---

## 개념

goroutine 간 **통신 및 동기화** 수단. "공유 메모리 대신 통신으로 동기화"

### Unbuffered vs Buffered
```go
ch1 := make(chan int)     // unbuffered: 송수신 동시에 준비돼야 함
ch2 := make(chan int, 10) // buffered: 버퍼 가득 찰 때까지 non-blocking
```

- **Unbuffered**: 강한 동기화. 송신자가 수신자를 기다림
- **Buffered**: 느슨한 결합. 버퍼가 완충재 역할

### Select
여러 channel을 동시에 기다릴 때 사용:
```go
select {
case msg := <-ch1:
    // ch1에서 수신
case ch2 <- data:
    // ch2에 송신
case <-time.After(3 * time.Second):
    // 타임아웃
case <-ctx.Done():
    // 취소
}
```

### Channel 닫기 규칙
- 송신자만 close
- 닫힌 channel에 send → panic
- 닫힌 channel에서 receive → zero value + false

---

## 면접 질문

**Q. unbuffered channel과 buffered channel의 차이와 선택 기준은?**
- unbuffered: **송수신 양쪽 모두 블로킹** — 상대방이 준비될 때까지 둘 다 기다림. 강한 동기화, goroutine 간 핸드셰이크
- buffered: 버퍼 가득 찰 때까지 non-blocking → **버퍼가 꽉 차면 sender 블로킹** (drop이 아님!)
- ⚠️ **오개념 주의**: "버퍼 가득 차면 drop" — 틀림. 블로킹됨. Drop하려면 `select + default` 명시 필요
- 선택: 동기화가 목적이면 unbuffered, throughput/decoupling이 목적이면 buffered

**면접 세션 피드백 (2026-03-28)**:
- 오개념: "버퍼 가득 차면 drop" → 실제로는 sender 블로킹
- 이력서 연결: 채팅 서버에서 select+default로 drop 처리한 경험과 연결할 것

---

**Q. select statement를 어떤 상황에서 사용하나요?**
- 여러 channel을 동시에 기다려야 할 때
- timeout 구현: `case <-time.After(3 * time.Second)`
- 취소 처리: `case <-ctx.Done()`
- non-blocking 시도: `default` 케이스 추가
- 실무: 트랜스코더 서버에서 ZooKeeper watch 이벤트 + timeout + 취소 신호를 select로 처리

---

**꼬리 질문: 느린 커넥션의 채널 버퍼가 가득 찼을 때 어떻게 처리했나요?** (채팅 서버)
- select + default 패턴으로 non-blocking send 구현
```go
select {
case conn.msgCh <- msg:
    failCount = 0
default:           // 버퍼 꽉 참 → non-blocking
    failCount++
    if failCount > threshold {
        conn.close()
    }
}
```
- 일정 횟수 초과 시 서버에서 종료 커맨드 → 클라이언트 재연결 유도
- WebSocket은 단일 writer 제약이 있음 → 커넥션마다 goroutine 하나가 write를 전담해야 함
