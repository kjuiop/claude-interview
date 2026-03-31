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

## 레이어별 에러 변환 패턴

클린 아키텍처에서 에러는 각 레이어를 지나며 **변환(convert)** 된다. 래핑(wrap)과 다름에 주의.

```go
// domain/errors.go — 도메인 에러 정의
var ErrUserNotFound = errors.New("user not found")

// repository.go — DB 에러 체인 유지
func (r *userRepo) GetByID(id int) (*User, error) {
    var u User
    err := r.db.QueryRow(...).Scan(&u)
    if err != nil {
        return nil, fmt.Errorf("getUserByID: %w", err) // sql.ErrNoRows 체인 유지
    }
    return &u, nil
}

// service.go — sql 에러를 도메인 에러로 교체
func (s *userService) GetUser(id int) (*User, error) {
    u, err := s.repo.GetByID(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound // sql 에러 노출 X, 도메인 에러로 교체
        }
        return nil, fmt.Errorf("userService.GetUser: %w", err)
    }
    return u, nil
}

// handler.go — 도메인 에러를 HTTP 상태 코드로 매핑
func (h *userHandler) GetUser(c *gin.Context) {
    u, err := h.service.GetUser(id)
    if err != nil {
        if errors.Is(err, service.ErrUserNotFound) {
            c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
            return
        }
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.JSON(http.StatusOK, u)
}
```

**핵심:** Repository는 `%w`로 체인 유지 → Service에서 `errors.Is()`로 감지 후 도메인 에러로 **교체** → Delivery에서 `errors.Is()`로 HTTP 상태 코드 결정

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

---

## errors.Join (Go 1.20+)

여러 개의 에러를 하나로 묶어 반환하는 함수. Go 1.20에서 도입.

```go
// 시그니처
func Join(errs ...error) error

// 기본 사용
err1 := errors.New("validation failed")
err2 := errors.New("db connection failed")
combined := errors.Join(err1, err2)
fmt.Println(combined)
// validation failed
// db connection failed
```

**내부 동작:**
- nil 에러는 자동으로 버린다
- 모든 인자가 nil이면 nil 반환
- 반환된 에러는 `Unwrap() []error` 메서드를 구현 → `errors.Is` / `errors.As` 로 각 에러 검사 가능
- 에러 메시지는 개행(`\n`)으로 구분하여 결합

```go
// Unwrap() []error 덕분에 개별 에러 탐색 가능
combined := errors.Join(ErrNotFound, ErrPermission)
fmt.Println(errors.Is(combined, ErrNotFound))   // true
fmt.Println(errors.Is(combined, ErrPermission)) // true
```

**언제 쓰는가:**

| 상황 | 이유 |
|---|---|
| 폼/요청 유효성 검사 | 필드별 에러를 모두 수집해 한 번에 반환 |
| 여러 리소스 정리 (defer) | 일부 실패해도 나머지 정리를 계속하고 에러를 합산 |
| 고루틴 에러 수집 | 복수 goroutine 결과를 하나로 합쳐 반환 |
| 루프 내 부분 실패 허용 | 전체 중단 없이 실패 목록 누적 |

```go
// 사용 예시 1: 유효성 검사
func validateUser(u User) error {
    var errs []error
    if u.Name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    if u.Age < 0 {
        errs = append(errs, errors.New("age must be non-negative"))
    }
    return errors.Join(errs...)
}

// 사용 예시 2: 정리 작업 (defer 패턴)
func processFile(path string) (err error) {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer func() {
        err = errors.Join(err, f.Close())
    }()
    // ... 처리
}

// 사용 예시 3: 고루틴 에러 수집
func runAll(tasks []Task) error {
    errs := make([]error, len(tasks))
    var wg sync.WaitGroup
    for i, t := range tasks {
        wg.Add(1)
        go func(i int, t Task) {
            defer wg.Done()
            errs[i] = t.Run()
        }(i, t)
    }
    wg.Wait()
    return errors.Join(errs...)
}
```

**errors.Join vs fmt.Errorf("%w")**

| | `errors.Join` | `fmt.Errorf("%w")` |
|---|---|---|
| 목적 | 복수 에러 병합 | 단일 에러에 컨텍스트 추가 |
| Unwrap | `[]error` 반환 | 단일 error 반환 |
| 메시지 | 개행으로 나열 | 직접 포맷 지정 |
| 사용 시점 | 에러 수집 후 합산 | 호출 스택 따라 래핑 |

**주의:**
- `errors.Join`은 에러 메시지 포맷을 직접 제어할 수 없다 → 커스텀 포맷이 필요하면 직접 구현
- 고루틴 환경에서 slice에 직접 쓸 경우 race condition 주의 (index 고정 or mutex 사용)
- `uber-go/multierr`는 같은 목적의 서드파티 라이브러리 (1.20 이전에 많이 사용)

> 출처: https://pkg.go.dev/errors | https://lukas.zapletalovi.com/posts/2022/wrapping-multiple-errors/ | https://tutorialedge.net/golang/joining-errors-with-errors-join-in-go/

---

## 면접 질문 — errors.Join

**Q. errors.Join은 언제 사용하고, fmt.Errorf("%w")와 어떻게 다른가요?**

**난이도**: 중급

**핵심 키워드**: errors.Join, 복수 에러 병합, Unwrap() []error, Go 1.20

**모범 답변 방향**:
- `fmt.Errorf("%w")`는 단일 에러에 컨텍스트를 추가할 때, `errors.Join`은 독립적인 여러 에러를 하나로 합칠 때 사용
- 유효성 검사처럼 "실패를 모두 수집해 한 번에 반환"해야 할 때 적합
- 반환된 에러는 `Unwrap() []error`를 구현하므로 `errors.Is` / `errors.As`로 개별 에러 탐색 가능
- nil은 자동으로 걸러지므로 `if err != nil` 없이 `errs = append(errs, err)`만 해도 됨

**꼬리 질문 예시**:
- errors.Join으로 합친 에러에서 특정 에러가 포함됐는지 확인하려면?
- defer에서 errors.Join을 활용해 Close 에러를 함께 반환하는 패턴을 설명해보세요
- uber-go/multierr와 표준 errors.Join의 차이는?
