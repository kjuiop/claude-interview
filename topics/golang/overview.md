---
tags: [golang, go, language-features, versions]
related: [goroutine, channel, interface, memory]
---

# Golang — 언어 개요 & 버전 변경사항

→ [[home]] | [[topics/golang/goroutine]] | [[topics/golang/channel]] | [[topics/golang/map]]

---

## Go 언어 특징

- 정적 타입, 컴파일 언어 — 빠른 실행 속도
- **동시성이 언어 레벨에서 지원** (goroutine, channel)
- GC 내장 — 메모리 관리 자동화, Stop-the-world pause 1ms 이하
- 암시적 인터페이스 구현 (Duck typing)
- 단순한 문법 — 키워드 25개

---

## Go 1.22~1.23 주요 변경사항

**Go 1.22 (2024.03) — 실무 영향 큰 변경:**
```go
// for loop 변수 스코프 수정 (오랜 버그 해결)
for i := range 10 {
    go func() { println(i) }()  // 이제 0~9 각각 출력 (이전엔 모두 10)
}

// HTTP 라우팅 개선
mux.HandleFunc("GET /users/{id}", handler)
id := r.PathValue("id")  // 메서드 매칭 + URL 파라미터
```

**Go 1.23 (2024.09):**
- Range-over-functions 정식 지원 (Iterator 패턴)
- `go mod tidy -diff` — 변경 사항만 확인

---

## 면접 질문

**Q. Go 1.22에서 for loop 변수 스코프가 왜 바뀌었나요?**

Go 1.22 이전에는 for loop의 변수가 모든 iteration에서 동일한 메모리 주소를 공유했습니다. goroutine 클로저 안에서 loop 변수를 참조하면, goroutine이 실제로 실행될 때는 이미 loop가 끝나 있어서 항상 마지막 값만 출력되는 버그가 발생했습니다. Go 1.22부터는 각 iteration마다 새로운 변수가 생성되어 클로저가 올바른 값을 캡처합니다. 실무적으로는 이 버그를 피하기 위해 `i := i`로 복사하던 관용 패턴이 더 이상 필요 없어졌습니다. 오래된 코드베이스에서는 이 패턴이 남아 있을 수 있어 Go 버전을 올릴 때 동작이 달라질 수 있다는 점을 인지해야 합니다.

---

## 참고 링크
- [Effective Go](https://go.dev/doc/effective_go)
- [Go 1.22 Release Notes](https://go.dev/doc/go1.22)
