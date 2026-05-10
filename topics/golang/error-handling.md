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

### 💬 면접 답변 형태로 읽기

Go의 에러 핸들링 철학은 exception을 사용하지 않고 에러를 일반 반환값으로 명시적으로 다루는 것입니다. 함수가 실패할 수 있으면 `(result, error)` 형태로 반환하고, 호출자가 즉시 처리 여부를 결정합니다. 이 접근 방식은 제어 흐름이 명확해지고 에러 처리가 누락되면 컴파일 타임에 드러나는 장점이 있습니다.

에러 래핑의 핵심은 `fmt.Errorf("context: %w", err)` 패턴입니다. `%w` 동사를 사용하면 원본 에러가 체인으로 보존되어, `errors.Is()`로 체인 전체를 탐색해 특정 sentinel error 값과 일치하는지 확인할 수 있고, `errors.As()`로 특정 타입의 에러를 추출할 수 있습니다. Repository 레이어에서는 `fmt.Errorf("userRepo.FindByID: %w", err)` 형태로 함수명을 prefix로 붙여 에러 발생 위치를 추적할 수 있게 합니다.

클린 아키텍처에서 에러는 레이어를 거치며 변환됩니다. Repository에서는 `%w`로 체인을 유지한 채 컨텍스트를 붙여 올립니다. Service에서는 `errors.Is(err, sql.ErrNoRows)`로 인프라 에러를 감지한 뒤 도메인 에러로 교체합니다. 이때 래핑이 아닌 교체가 핵심인데, `sql.ErrNoRows` 같은 인프라 세부 사항을 그대로 올리면 Service 레이어가 DB 구현에 결합됩니다. Handler에서는 `errors.Is(err, ErrUserNotFound)`로 도메인 에러를 HTTP 상태 코드로 매핑합니다.

에러 처리의 주요 원칙으로, 한 곳에서만 처리해야 합니다. 에러를 로깅하고 다시 반환하면 상위 레이어에서 또 로깅해 중복 로그가 쌓입니다. 로깅하거나 반환하거나 둘 중 하나만 해야 합니다. 외부 API 응답에는 DB 스키마, 파일 경로 같은 내부 상세 정보를 노출하지 않아야 보안 위험을 줄일 수 있습니다. Go 1.20부터는 `errors.Join`으로 여러 에러를 하나로 묶어 반환할 수 있어, 유효성 검사처럼 실패를 모두 수집해 한 번에 반환해야 하는 상황에 유용합니다.

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

**면접 세션 피드백 (2026-04-10 1회차)**:
- 잘한 점: Repository %w 전파 → Service errors.Is 분기 → 도메인 에러 교체 흐름 정확. wrap vs replace 개념 구분 명확.
- 보완:
  - **Repository prefix 패턴**: `fmt.Errorf("userRepo.FindByID: %w", err)` — 함수명을 prefix로 붙여 에러 발생 위치 추적
  - **Gin 구문 오류**: `gin.H{HttpStatus.BadRequest, data}` ❌ → `c.JSON(http.StatusBadRequest, gin.H{"error": "..."})` ✅
    - `gin.H`는 `map[string]any` → 반드시 `{"key": value}` 형식
    - Go 상수: `http.StatusBadRequest` (HttpStatus ❌)
    - gin.Context 메서드: `c.JSON()` 으로 호출
  - **이중 래핑 비표준 이유**: 레이어 결합도뿐 아니라 → Go 1.20+ `%w` 두 개는 복합 에러 체인 생성, errors.Is 탐색 경로 모호해짐

---

**Q. Go의 에러 핸들링 철학과 best practice는?**

Go는 exception 대신 에러를 일반 반환값으로 처리합니다. 함수가 실패할 수 있으면 마지막 반환값으로 `error` 인터페이스를 반환하고, 호출자가 `if err != nil`로 즉시 처리 여부를 결정합니다. 이 구조는 제어 흐름이 예측 가능하고, 에러 처리가 누락되면 컴파일러나 linter가 잡아낼 수 있다는 장점이 있습니다. `%w` 동사로 에러를 래핑하면 원본 에러가 체인에 보존되어, `errors.Is()`로 sentinel error 값을 탐색하거나 `errors.As()`로 특정 타입의 에러를 추출할 수 있습니다. 에러를 한 곳에서만 처리하는 원칙도 중요합니다. 에러를 로깅한 뒤 다시 반환하면 상위 레이어에서 또 로깅해 중복 로그가 쌓입니다. 로깅하거나 반환하거나 둘 중 하나만 해야 합니다. 외부 API 응답에는 DB 스키마나 파일 경로 같은 내부 정보가 포함되지 않도록 도메인 에러로 교체한 뒤 노출해야 합니다.

**꼬리 질문: errors.Is와 errors.As의 차이는?**

`errors.Is`와 `errors.As`는 모두 `%w`로 래핑된 에러 체인을 재귀적으로 unwrap하며 탐색한다는 공통점이 있지만, 비교 대상이 다릅니다. `errors.Is`는 에러 체인에 특정 sentinel error **값**이 존재하는지 확인합니다. 예를 들어 `errors.Is(err, sql.ErrNoRows)`는 체인 어딘가에 `sql.ErrNoRows`와 동일한 값이 있는지 검사합니다. `errors.As`는 에러 체인에 특정 **타입**이 존재하는지 확인하고, 해당 타입으로 캐스팅된 에러 값을 추출합니다. `errors.As(err, &ve)`처럼 사용하면 체인을 탐색해 `*ValidationError` 타입을 찾고 그 값을 `ve`에 넣어줍니다. 따라서 에러의 존재 여부만 판단할 때는 `errors.Is`, 에러의 구체적인 필드를 꺼내야 할 때는 `errors.As`를 사용합니다.

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

`fmt.Errorf("%w")`와 `errors.Join`은 모두 에러를 다루지만 목적이 다릅니다. `fmt.Errorf("%w")`는 단일 에러에 컨텍스트를 추가하는 래핑으로, 호출 스택을 따라 에러를 전파할 때 사용합니다. Repository에서 `fmt.Errorf("userRepo.FindByID: %w", err)`처럼 함수명을 prefix로 붙여 에러 발생 위치를 추적할 수 있게 하는 방식이 대표적입니다.

반면 `errors.Join`은 Go 1.20에서 도입된 함수로, 독립적인 여러 에러를 하나로 합칠 때 사용합니다. 유효성 검사처럼 여러 필드를 검사해 실패를 모두 수집한 뒤 한 번에 반환해야 하는 상황, 또는 여러 리소스를 정리하다가 일부 실패한 에러를 합산해야 하는 상황에 적합합니다. `errors.Join`이 반환한 에러는 내부적으로 `Unwrap() []error` 메서드를 구현하므로, `errors.Is`와 `errors.As`로 개별 에러를 탐색할 수 있습니다. 즉 `errors.Join(ErrNotFound, ErrPermission)`으로 합친 에러에 대해 `errors.Is(combined, ErrNotFound)`를 호출하면 true가 반환됩니다. nil 에러는 자동으로 걸러지므로 `if err != nil` 조건 없이 `errs = append(errs, err)`만 해도 안전하게 수집됩니다. defer 패턴에서도 유용한데, `defer func() { err = errors.Join(err, f.Close()) }()` 처럼 사용하면 함수 본체 에러와 Close 에러를 모두 호출자에게 전달할 수 있습니다. 두 함수를 선택하는 기준은 단순합니다. 하나의 에러를 컨텍스트와 함께 전파할 때는 `fmt.Errorf("%w")`, 여러 에러를 수집해서 한꺼번에 반환할 때는 `errors.Join`입니다. 주의할 점은 `errors.Join`은 에러 메시지 포맷을 직접 제어할 수 없어서 커스텀 포맷이 필요한 경우에는 `joinError` 같은 타입을 직접 구현하거나, Go 1.20 이전 방식으로 사용되던 `uber-go/multierr` 라이브러리를 고려해야 합니다.

**꼬리 질문 예시**:
- errors.Join으로 합친 에러에서 특정 에러가 포함됐는지 확인하려면?
- defer에서 errors.Join을 활용해 Close 에러를 함께 반환하는 패턴을 설명해보세요
- uber-go/multierr와 표준 errors.Join의 차이는?
