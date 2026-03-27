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
- 이전: 모든 iteration이 같은 변수를 공유 → goroutine 클로저에서 항상 마지막 값 출력하는 버그
- Go 1.22+: 각 iteration마다 새 변수 → 의도대로 동작
- 실무 영향: 기존 코드 중 이 버그를 우회하기 위해 `i := i` 복사하던 패턴 불필요

---

## 참고 링크
- [Effective Go](https://go.dev/doc/effective_go)
- [Go 1.22 Release Notes](https://go.dev/doc/go1.22)
