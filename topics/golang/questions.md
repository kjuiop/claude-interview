---
tags: [golang, go, concurrency, interview-questions]
related: [distributed-systems, kubernetes]
---

# Golang — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/golang/concepts]]

---

## 기초

**Q. goroutine과 OS 스레드의 차이는?**
- OS 스레드: ~1MB 스택, OS가 스케줄링, 컨텍스트 스위칭 비용 큼
- goroutine: ~2KB 스택 (동적 확장), Go 런타임이 스케줄링, 수백만 개 생성 가능
- G-P-M 모델: N개 goroutine을 M개 OS 스레드에 M:N 매핑

---

**Q. unbuffered channel과 buffered channel의 차이와 선택 기준은?**
- unbuffered: 송수신이 동시에 준비돼야 함 → 강한 동기화, goroutine 간 핸드셰이크
- buffered: 버퍼 가득 찰 때까지 non-blocking → 느슨한 결합, 일시적 burst 처리
- 선택: 동기화가 목적이면 unbuffered, throughput이 목적이면 buffered
- 참고: [[topics/golang/concepts#3. Channel]]

---

**Q. Mutex와 Channel을 어떻게 구분해서 사용하나요?**
- Mutex: 공유 상태 보호 (캐시, 카운터, 맵)
- Channel: goroutine 간 데이터 전달, 소유권 이전, 결과 수집
- 실무 경험: 채팅 서버에서 broadcast에 Mutex를 사용했다가 경합으로 Latency 13초 → channel 기반 per-connection queue로 전환 → 103ms
- 참고: [[topics/golang/concepts#4. Mutex vs Channel]]

---

## 동시성 심화

**Q. goroutine leak이 발생하는 상황과 방지 방법은?**
- 원인: 아무도 읽지 않는 channel에 전송, 종료 신호 없는 무한 루프, context 미전파
- 방지: context.Context로 취소 신호 전달, select + ctx.Done() 패턴
- 탐지: `runtime.NumGoroutine()` 모니터링, `uber-go/goleak` 테스트
- 참고: [[topics/golang/concepts#Goroutine Leak (면접 단골)]]

---

**Q. Context를 왜 사용하고, 어떻게 올바르게 전달하나요?**
- 목적: goroutine 취소 전파, 타임아웃/데드라인, request-scoped 값 전달
- 올바른 사용: 함수의 첫 번째 파라미터로 전달, struct 필드로 저장 금지
- 부모 context 취소 시 자식 goroutine 전체 자동 취소
- 실무: HTTP handler에서 `r.Context()` 받아서 DB 쿼리까지 전파 → 클라이언트 연결 끊기면 쿼리도 취소
- 참고: [[topics/golang/concepts#5. Context]]

---

**Q. select statement를 어떤 상황에서 사용하나요?**
- 여러 channel을 동시에 기다려야 할 때
- timeout 구현: `case <-time.After(3 * time.Second)`
- 취소 처리: `case <-ctx.Done()`
- non-blocking 시도: `default` 케이스 추가
- 실무: 트랜스코더 서버에서 ZooKeeper watch 이벤트 + timeout + 취소 신호를 select로 처리

---

## 메모리 & 성능

**Q. Escape Analysis란 무엇이고 왜 중요한가요?**
- 컴파일러가 변수를 스택에 할당할지 힙에 할당할지 결정하는 분석
- 스택 할당: 빠름, GC 불필요. 힙 할당: 느림, GC 대상
- 확인: `go build -gcflags="-m" ./...` → "escapes to heap" 출력
- 최적화: 포인터 반환 최소화, 인터페이스 사용 줄이기 (인터페이스 값은 힙으로 이스케이프)
- 참고: [[topics/golang/concepts#7. 메모리 관리 & Escape Analysis]]

---

**Q. Go GC의 특징과 성능에 미치는 영향은?**
- Concurrent tri-color mark-and-sweep — STW pause 1ms 이하
- 힙이 GOGC% 증가하면 GC 트리거 (기본 100 = 2배 되면 실행)
- 튜닝: `GOGC=200` (GC 빈도 줄이기), `GOMEMLIMIT` (메모리 상한 설정, Go 1.19+)
- 대용량 트래픽에서 GC pause가 Latency spike 유발 가능 → pprof로 분석

---

## 에러 핸들링

**Q. Go의 에러 핸들링 철학과 best practice는?**
- exception 없이 명시적 에러 반환 → 제어 흐름이 명확
- `%w`로 wrap: `fmt.Errorf("context: %w", err)` → errors.Is/As로 unwrap 가능
- 한 곳에서만 처리: 로깅하거나 반환하거나, 둘 다 하지 말 것
- 외부 노출 시 내부 상세 정보 제거 (DB 스키마, 파일 경로 등 보안)

---

## Go 1.22 이후

**Q. Go 1.22에서 for loop 변수 스코프가 왜 바뀌었나요?**
- 이전: 모든 iteration이 같은 변수를 공유 → goroutine 클로저에서 항상 마지막 값 출력하는 버그
- Go 1.22+: 각 iteration마다 새 변수 → 의도대로 동작
- 실무 영향: 기존 코드 중 이 버그를 우회하기 위해 `i := i` 복사하던 패턴 불필요

---

## 이력서 연계 심화 질문

**Q. WebSocket 채팅 서버에서 Mutex 경합을 어떻게 해결했나요?**
- 문제: broadcast 경로에 Mutex Lock → 느린 커넥션 하나가 전체 전파 블로킹
- 분석: k6 부하 테스트로 재현, goroutine 스케줄링 증가 확인
- 해결: 각 커넥션에 독립적인 channel 기반 메시지 큐 부여 → lock-free broadcast
- 결과: Latency 13초 → 103ms (126배), 5,000명 동시 접속에서도 CPU spike 없음
- **답변 시작은 반드시 수치로**: "이 구조 전환으로 메시지 지연이 13초에서 103ms로 줄었습니다"

**꼬리 질문: 느린 커넥션의 채널 버퍼가 가득 찼을 때 어떻게 처리했나요?**
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

**꼬리 질문: goroutine 종료는 어떻게 보장했나요?**
- context.Done() 신호를 goroutine 내 select에서 수신
- 커넥션 종료 시 context cancel 호출 → goroutine 안전 종료

---

**Q. 트랜스코더 서버에서 ZooKeeper를 어떻게 활용했나요?**
- 문제: 70대 트랜스코더의 상태를 1분 주기 polling → 불필요한 네트워크/DB 부하
- 해결: ZooKeeper ephemeral node + Watch 이벤트 → goroutine이 Watch channel을 select로 대기
- 결과: polling 제거, 상태 변화 시에만 처리, 무중단 enable/disable 제어 가능
- 관련 개념: [[topics/zookeeper/concepts]]
