---
tags: [golang, go, error-handling, errors, wrapping]
related: [interface]
---

# Golang — 에러 핸들링

→ [[home]] | [[topics/golang/interface]]

---

## 개념

```go
// 기본 패턴
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)  // %w로 wrap
}

// errors.Is / errors.As (Go 1.13+)
if errors.Is(err, ErrNotFound) { ... }

var dbErr *DatabaseError
if errors.As(err, &dbErr) { ... }
```

**핵심 원칙:**
- 한 곳에서만 처리 (로깅 또는 반환, 둘 다 하지 말 것)
- `%w`로 wrap하여 에러 체인 유지
- 외부로 노출 시 내부 상세 정보 제거 (보안)

**sentinel error vs error type:**
```go
// sentinel error: 특정 에러 값 비교
var ErrNotFound = errors.New("not found")
if errors.Is(err, ErrNotFound) { ... }

// error type: 구조체로 에러 정보 포함
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string { return e.Message }

var ve *ValidationError
if errors.As(err, &ve) { fmt.Println(ve.Field) }
```

---

## 면접 질문

**Q. Go의 에러 핸들링 철학과 best practice는?**
- exception 없이 명시적 에러 반환 → 제어 흐름이 명확
- `%w`로 wrap: `fmt.Errorf("context: %w", err)` → errors.Is/As로 unwrap 가능
- 한 곳에서만 처리: 로깅하거나 반환하거나, 둘 다 하지 말 것
- 외부 노출 시 내부 상세 정보 제거 (DB 스키마, 파일 경로 등 보안)

**꼬리 질문: errors.Is와 errors.As의 차이는?**
- `errors.Is`: 에러 체인에 특정 **값**(sentinel error)이 있는지 확인
- `errors.As`: 에러 체인에 특정 **타입**이 있는지 확인하고 값 추출
- `%w`로 wrap된 에러는 체인을 따라 unwrap하며 탐색
