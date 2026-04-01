---
tags: [golang, go, interface, duck-typing, polymorphism]
related: [goroutine, memory, error-handling, gin]
---

# Golang — Interface

→ [[home]] | [[topics/golang/overview]] | [[topics/golang/memory]]

---

## 개념

**암시적 구현** — 메서드만 맞으면 자동으로 인터페이스를 구현한 것으로 간주.

```go
type Writer interface {
    Write([]byte) (int, error)
}

// File, Buffer, net.Conn 등이 모두 Writer를 구현
// 선언 없이 자동으로
```

**작은 인터페이스 선호 (Interface Segregation)**:
```go
// 나쁜 예: 너무 큰 인터페이스
type Storage interface {
    Read() / Write() / Delete() / List() / ...
}

// 좋은 예: 필요한 것만
type Reader interface { Read() }
type Writer interface { Write() }
```

**Empty interface{}**:
```go
// Go 1.18+ any 키워드
func process(v any) {
    switch t := v.(type) {
    case int:    ...
    case string: ...
    }
}
```

---

## 면접 질문

**Q. Go의 인터페이스가 Java와 다른 점은?**
- Java: `implements` 키워드로 명시적 선언 필요
- Go: 메서드 시그니처만 맞으면 자동으로 인터페이스 구현 (Duck typing)
- 장점: 결합도 낮음 — 인터페이스 정의자와 구현자가 서로 모르는 상태로도 연결 가능
- 단점: 구현 여부를 코드에서 명시적으로 알기 어려움

**꼬리 질문: 인터페이스 변수에 nil 포인터를 할당하면 어떻게 되나요?**
```go
var err *MyError = nil
var i error = err
fmt.Println(i == nil) // false! — interface는 (타입, 값) 쌍이므로 타입이 있으면 nil이 아님
```
- interface의 zero value: 타입 포인터와 데이터 포인터 모두 nil일 때만 nil
- 함수가 error interface를 반환할 때 포인터 타입 직접 반환하면 이 함정에 빠질 수 있음

**Q. 인터페이스의 암시적 구현이 테스트와 외부 라이브러리 연동에서 어떤 이점이 있나요?**
- **테스트 Mock**: 인터페이스 메서드만 구현한 Mock struct를 `implements` 선언 없이 바로 주입 가능
- **외부 라이브러리 연동**: 수정 불가능한 외부 패키지 struct도 메서드만 맞으면 내 인터페이스에 끼워 넣을 수 있음. Java는 소스에 `implements` 선언 필요하므로 외부 코드에는 적용 불가
- **Go 관용구**: `"Accept interfaces, return structs"` — 파라미터는 인터페이스로 받아 결합도를 낮추고, 반환은 구체 타입으로 명시

**면접 세션 피드백 (2026-04-01)**:
- 잘한 점: 암시적 구현 핵심, Mock 테스트 연결, Java 비교
- 보완: 외부 라이브러리 연동 예시, `Accept interfaces, return structs` 관용구 추가로 완성도 향상

**Q. 인터페이스를 사용하면 성능에 영향이 있나요?**
- interface 값은 힙으로 escape됨 → GC 부하 증가 가능
- 인터페이스 메서드 호출은 간접 호출 (vtable lookup) → 직접 호출보다 약간 느림
- 성능이 매우 중요한 hot path에서는 concrete type 사용 고려
- 참고: [[topics/golang/memory#메모리 관리 & Escape Analysis]]
