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

### 💬 면접 답변 형태로 읽기

Go에서 channel은 goroutine 간 데이터를 주고받는 동시에 실행 흐름을 동기화하는 수단입니다. Go의 설계 철학인 "공유 메모리로 통신하지 말고, 통신으로 메모리를 공유하라"는 원칙이 channel을 통해 구현됩니다.

channel의 핵심 동작 원리는 unbuffered와 buffered 두 가지로 나뉩니다. unbuffered channel은 송신자와 수신자가 동시에 준비되어야 데이터가 전달됩니다. 어느 한쪽이 먼저 도착하면 상대방이 나타날 때까지 블로킹됩니다. 이 특성이 goroutine 간 핸드셰이크처럼 동작해 강한 동기화를 보장합니다. buffered channel은 버퍼가 가득 찰 때까지 송신자가 블로킹 없이 데이터를 넣을 수 있어 송수신 goroutine 사이의 속도 차이를 흡수합니다. 여기서 중요한 점은 버퍼가 가득 찼을 때 기본 동작이 drop이 아니라 sender 블로킹이라는 것입니다. drop이 필요하다면 `select + default` 패턴을 명시적으로 작성해야 합니다.

select는 여러 channel을 동시에 대기하는 구문으로, 단순 분기가 아닌 복수의 channel 중 준비된 케이스를 골라 처리합니다. 준비된 케이스가 여러 개면 런타임이 무작위로 하나를 선택하므로 case 순서는 우선순위와 무관합니다. 실무에서 자주 쓰는 패턴은 `time.After`로 타임아웃 처리, `ctx.Done()`으로 취소 신호 수신, `default` 케이스로 non-blocking 시도를 구현하는 세 가지입니다. `ctx.Done()`은 콜백이 아니라 채널이기 때문에 부모 context가 취소되면 해당 채널이 close되어 select 케이스가 선택됩니다.

트레이드오프와 주의점으로는 nil channel과 closed channel을 혼동하지 않아야 합니다. nil channel을 select 케이스에 넣으면 panic이 아니라 해당 케이스가 영원히 선택되지 않는 비활성 상태가 됩니다. 이 특성을 활용해 조건에 따라 케이스를 동적으로 껐다 켤 수 있습니다. 닫힌 channel에서 읽으면 zero value와 false가 반환되며 panic이 발생하지 않지만, 닫힌 channel에 쓰면 panic이 발생합니다. 그래서 channel은 반드시 송신자만 닫아야 합니다.

실무에서는 WebSocket 채팅 서버에서 느린 커넥션 처리에 이 패턴을 사용했습니다. 버퍼가 가득 찬 상황에서 무한 대기하면 빠른 커넥션까지 영향을 받기 때문에, `select + default`로 non-blocking send를 구현하고 실패 횟수를 카운팅해 임계치를 넘으면 서버에서 커넥션을 종료하는 방식으로 처리했습니다. WebSocket의 단일 writer 규칙도 있어, 커넥션마다 전담 write goroutine 하나가 채널에서 메시지를 꺼내 write하는 구조를 채택했습니다.

---

## 면접 질문

**Q. unbuffered channel과 buffered channel의 차이와 선택 기준은?**

unbuffered channel과 buffered channel의 가장 큰 차이는 동기화 강도입니다. unbuffered channel은 송신자와 수신자가 동시에 준비되어야 데이터가 전달되기 때문에, 양쪽 모두 상대방이 나타날 때까지 블로킹됩니다. 이것은 goroutine 간에 핸드셰이크처럼 동작하며, 강한 동기화가 필요한 상황에 적합합니다. 반면 buffered channel은 버퍼가 가득 찰 때까지 송신자가 블로킹 없이 데이터를 넣을 수 있어 송수신 goroutine을 느슨하게 결합할 수 있습니다. 여기서 주의할 점이 있는데, 버퍼가 가득 찼을 때 기본 동작은 drop이 아니라 sender 블로킹입니다. 만약 drop이 필요하다면 `select + default` 패턴을 명시적으로 작성해야 합니다. 선택 기준은 이렇게 생각합니다. goroutine 간 타이밍을 맞춰야 하거나 강한 동기화가 목적이라면 unbuffered, 처리 속도 차이를 완충하거나 decoupling이 목적이라면 buffered를 사용합니다.

**면접 세션 피드백 (2026-03-28)**:
- 오개념: "버퍼 가득 차면 drop" → 실제로는 sender 블로킹
- 이력서 연결: 채팅 서버에서 select+default로 drop 처리한 경험과 연결할 것

**면접 세션 피드백 (2026-03-30)**:
- select를 "분기처리"로 설명 — 핵심은 **여러 채널을 동시에 대기**, 준비된 케이스가 여러 개면 **non-deterministic 선택**
- case 순서는 우선순위와 무관
- ctx.Done()은 callback이 아닌 채널 — 취소 시 채널이 close되어 읽힘. `case <-ctx.Done(): return` 패턴 암기 필요

---

**Q. select statement를 어떤 상황에서 사용하나요?**

select는 여러 채널을 동시에 대기해야 할 때 사용합니다. 핵심은 분기 처리가 아니라 복수의 채널 중 준비된 케이스를 골라 처리한다는 점입니다. 준비된 케이스가 여러 개면 런타임이 무작위로 하나를 선택하므로, case 순서는 우선순위와 무관합니다. 실무에서 가장 자주 쓰는 패턴은 세 가지입니다. 첫째, `case <-time.After(3 * time.Second)` 로 타임아웃을 구현합니다. 둘째, `case <-ctx.Done()` 으로 취소 신호를 수신합니다. ctx.Done()은 콜백이 아니라 채널이기 때문에, 부모 context가 취소되면 해당 채널이 close되어 이 케이스가 선택됩니다. 셋째, `default` 케이스를 추가해 블로킹 없이 non-blocking 시도를 구현합니다. 실제로 트랜스코더 서버를 개발할 때 ZooKeeper watch 이벤트 채널, 타임아웃, 취소 신호를 select 하나로 처리한 적이 있는데, 이 구조 덕분에 이벤트 주도 방식으로 전환하면서 polling을 완전히 제거할 수 있었습니다.

---

**Q. nil 채널을 select 케이스에 넣으면 어떻게 되나요? 언제 활용하나요?**

**난이도**: 중급

nil 채널을 select 케이스에 넣으면 panic이 발생하는 게 아니라, 해당 케이스가 영원히 선택되지 않는 상태가 됩니다. 즉, 비활성화된 케이스처럼 동작합니다. 이 특성을 활용하면 특정 케이스를 동적으로 껐다 켤 수 있습니다. 예를 들어 secondary 채널 변수를 nil로 초기화해두면 select 루프에서 해당 케이스는 완전히 무시되고, 나중에 `secondary = make(chan int)` 로 실제 채널을 할당하는 순간부터 다시 활성화됩니다.
```go
var secondary chan int  // nil

select {
case v := <-primary:      // 정상 작동
    handle(v)
case v := <-secondary:    // nil → 절대 선택 안 됨 (비활성화 상태)
    handleSecondary(v)
case <-ctx.Done():
    return
}
// secondary 활성화하려면: secondary = make(chan int)
```

**닫힌 채널(closed channel) 읽기 vs 쓰기:**
- 읽기: **(zero value, false)** 반환 — panic ❌
- 쓰기: **panic** 발생
- 암기: "닫힌 채널은 써야 panic, 읽으면 zero+false"
```go
v, ok := <-ch  // ok == false → 채널 닫혔음을 감지
```

**면접 세션 피드백 (2026-04-10 1회차)**:
- 오개념: "닫힌 채널 읽기 → panic", "nil 채널 select → panic" — 둘 다 틀림
- nil 채널 = 영원히 블록 (비활성 케이스). 닫힌 채널 읽기 = zero+false
- 오개념: "버퍼 채널 가득 차면 drop" — 기본 동작은 sender Block. drop은 select+default 명시 시에만

---

**꼬리 질문: 느린 커넥션의 채널 버퍼가 가득 찼을 때 어떻게 처리했나요?** (채팅 서버)

WebSocket 채팅 서버에서 느린 커넥션 문제를 다뤘을 때, 버퍼가 가득 찬 상황에서 메시지를 무한정 기다리면 빠른 커넥션까지 영향을 받는 구조였기 때문에, select + default 패턴으로 non-blocking send를 구현했습니다. 메시지 채널에 넣을 수 없으면 즉시 default로 빠져나와 실패 카운터를 올리고, 일정 횟수를 초과하면 서버에서 종료 커맨드를 보내 클라이언트가 재연결하도록 유도했습니다. 또 한 가지 중요한 제약이 있었는데, WebSocket은 단일 writer 규칙이 있어서 커넥션마다 전담 write goroutine 하나가 채널에서 메시지를 꺼내 write하는 구조를 취해야 했습니다.
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
