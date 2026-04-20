---
tags: [golang, go, golangci-lint, linter, static-analysis, ci]
related: [golang]
---

# Golang — golangci-lint

→ [[home]] | [[topics/golang/overview]]

---

## 개념

Go를 위한 **메타 린터(fast linters runner)**. linter 자체가 아니라 100개 이상의 개별 linter를 하나의 도구로 묶어 **병렬 실행**하는 집계 도구.

Kubernetes, Prometheus, Terraform 등 주요 오픈소스의 사실상 표준. 2025년 현재 안정 버전 **v2.x**.

### 내부 동작 원리

```
Init → Load packages → Run linters → Postprocess issues → Print issues
```

| 메커니즘 | 설명 |
|---|---|
| **병렬 실행** | 모든 linter를 단일 프로세스 내에서 병렬 실행. 공통 작업 공유 |
| **AST 공유** | `go/packages`로 패키지를 한 번만 파싱하고 AST를 공유 → 메모리 26% 절감 |
| **빌드 캐시 재사용** | Go 빌드 캐시(`$GOCACHE`) 재사용 + 자체 캐시로 변경된 파일만 재분석 |
| **Load Mode 최적화** | 각 linter가 필요한 정보만 선언 → 그 합집합만 로드 |

> 핵심: 여러 linter를 순차 실행하면 패키지를 n번 파싱하지만, golangci-lint는 **한 번만 파싱하고 공유**.

### 주요 Linter 목록

**기본 활성화(default enabled)**

| Linter | 역할 |
|---|---|
| `govet` | Go 공식 도구. shadowing, printf 포맷 불일치 등 검출 |
| `errcheck` | 에러 반환값 무시 코드 검출 |
| `staticcheck` | 광범위한 정적 분석. 불필요한 코드, deprecated API, 버그 패턴 |
| `ineffassign` | 사용되지 않는 변수 할당 검출 |
| `unused` | 사용되지 않는 코드(변수, 함수, 타입) 검출 |

**추가 권장**

| Linter | 역할 |
|---|---|
| `gosec` | 보안 취약점 (SQL injection, 하드코딩 크리덴셜, TLS 설정) |
| `revive` | golint 후계자. 60개 이상 규칙, 세밀한 설정 |
| `gocyclo` | 함수 순환 복잡도 측정 |
| `bodyclose` | HTTP response body close 누락 검출 |
| `noctx` | context 없이 HTTP request 생성 검출 |
| `misspell` | 영어 오타 검출 |

### 설정 파일 (.golangci.yml v2)

```yaml
version: "2"

linters:
  default: standard
  enable:
    - gosec
    - revive
    - gocyclo
    - misspell
    - bodyclose
    - noctx
  disable:
    - depguard

linters-settings:
  gocyclo:
    min-complexity: 15
  gosec:
    excludes:
      - G104

issues:
  exclude-rules:
    - path: "_test\\.go"
      linters:
        - gosec
  max-issues-per-linter: 50

run:
  timeout: 5m
  go: "1.22"
```

**v2 주요 변경점 (2025):**
- `enable-all` / `disable-all` → `linters.default` 단일 옵션으로 대체
- `golangci-lint fmt` 신규 추가 (포맷팅 통합)

### CI/CD 연동 (GitHub Actions)

```yaml
# .github/workflows/lint.yml
name: golangci-lint
on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: stable
      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.11
          args: --timeout=5m
```

### 실무 Best Practices

| 실천 항목 | 내용 |
|---|---|
| **점진적 도입** | 한 번에 모든 linter 켜지 말고, 팀이 적응하며 매월 1개씩 추가 |
| **nolint 정책** | `//nolint:lintername` 사용 시 반드시 이유 주석 포함 |
| **테스트 파일 예외** | `_test.go`에서 gosec 등 일부 linter 제외 |
| **버전 고정** | CI와 로컬 환경의 버전 일치 필수 |
| **--fix 활용** | `golangci-lint run --fix`로 자동 수정 가능한 문제는 자동화 |
| **PR 단위 린팅** | `--new-from-rev=HEAD~1`로 변경된 파일만 린팅 |

---

## 면접 질문

**Q. golangci-lint가 무엇이고, 왜 사용하나요?**

**핵심 키워드**: 메타 린터, 정적 분석, 코드 품질, 자동화 코드 리뷰

**모범 답변 방향**:
golangci-lint는 100개 이상의 Go linter를 하나의 도구로 묶어서 병렬로 실행하는 메타 린터입니다. 사용하는 이유는 크게 두 가지입니다. 첫째, 개별 linter를 따로 실행하면 각각이 패키지를 별도로 파싱해서 n번의 AST 분석이 발생하지만, golangci-lint는 AST를 공유하고 결과를 캐시하기 때문에 한 번만 파싱해서 훨씬 빠릅니다. 둘째, CI에 연동해서 PR마다 자동으로 코드 품질을 검증하는 "자동화된 코드 리뷰어" 역할을 합니다. 사람이 코드 리뷰할 때 마다 놓칠 수 있는 에러 처리 누락, 보안 취약점, 성능 문제 등을 자동으로 잡아줍니다.

**꼬리 질문 예시**:
- golangci-lint가 빠른 이유는? (AST 공유 + 병렬 실행 + 빌드 캐시)
- go vet과 staticcheck의 차이는?

---

**Q. golangci-lint를 실제 프로젝트에 도입할 때 어떤 전략을 사용하나요?**

**핵심 키워드**: 점진적 도입, .golangci.yml, nolint, CI 연동, 팀 컨벤션

**모범 답변 방향**:
golangci-lint를 기존 프로젝트에 처음 도입할 때 가장 큰 함정은 모든 linter를 한꺼번에 켜는 것입니다. 기존 코드 위반이 수천 개가 쏟아지면 팀 저항이 생겨서 결국 포기하게 됩니다. 점진적 도입이 현실적입니다. 1단계에서는 기본 linter(`govet`, `errcheck`, `staticcheck`)만 활성화하고 기존 위반을 수정합니다. 팀이 적응하면 2단계에서 보안 관련 `gosec`, 코드 스타일 `revive`, HTTP 응답 바디 닫힘 검사 `bodyclose` 같은 linter를 단계적으로 추가합니다. `//nolint:lintername // 이유 명시` 주석으로 특정 위반을 억제할 수 있는데, 이유 주석 없는 nolint는 기술 부채이므로 팀 정책으로 이유 명시를 필수화합니다. CI에서는 `--new-from-rev=HEAD~1` 옵션으로 변경된 파일만 린팅해서 PR 피드백 속도를 높입니다.

---

**Q. golangci-lint에서 gosec linter는 어떤 역할을 하고, 어떤 상황에서 제외하나요?**

**핵심 키워드**: gosec, 보안 취약점, G104, SQL injection, TLS, nolint

**모범 답변 방향**:
gosec은 Go 코드에서 보안 취약점을 정적으로 검출하는 linter입니다. SQL injection 위험이 있는 쿼리 문자열 조합, 하드코딩된 크리덴셜, 약한 TLS 설정, 에러가 무시된 경우 등을 찾아냅니다. 특히 G104 규칙(에러 무시)은 테스트 코드에서 false positive가 자주 발생합니다. `defer resp.Body.Close()` 같은 테스트 정리 코드에서 에러를 무시하는 게 일반적인 패턴인데, gosec이 이를 위반으로 잡습니다. 이런 경우 `.golangci.yml`의 `exclude-rules`에서 `_test.go` 파일에 대해 G104를 제외합니다. 실무 원칙은 외부 입력이 들어오는 경로인 HTTP handler, DB 쿼리, 파일 처리 코드에는 gosec을 반드시 통과시키고, 테스트나 mock 코드에서만 예외를 허용하는 것입니다.

---

**Q. linter와 formatter의 차이는 무엇이고, Go에서는 각각 무엇을 사용하나요?**

**모범 답변 방향**:
formatter와 linter는 목적이 다릅니다. formatter는 코드 스타일을 정규화하는 도구입니다. 들여쓰기, 공백, import 정렬 같은 형식을 통일하지만 버그는 잡지 않습니다. Go 공식 formatter는 `gofmt`이고, import를 자동으로 추가·정렬해주는 `goimports`도 많이 씁니다. linter는 버그 패턴, 보안 취약점, 코드 품질 문제를 검출합니다. 에러 처리 누락, null 역참조 위험, 사용하지 않는 변수 등을 찾습니다. Go에서는 `go vet`(기본 정적 분석), `staticcheck`(고급 정적 분석), `golangci-lint`(메타 린터)를 씁니다. golangci-lint v2부터는 `golangci-lint fmt` 서브커맨드로 포맷팅까지 통합해서 하나의 도구로 포맷팅과 린팅을 모두 처리할 수 있습니다.

---

## 참고 링크
- [golangci-lint 공식 문서](https://golangci-lint.run/)
- [Architecture](https://golangci-lint.run/docs/contributing/architecture/)
- [Welcome to v2](https://ldez.github.io/blog/2025/03/23/golangci-lint-v2/)
- [당근 테크 블로그](https://medium.com/daangn/golangci-lint%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-%EC%BD%94%EB%94%A9-%EC%8A%A4%ED%83%80%EC%9D%BC%EC%9D%84-%ED%9A%A8%EA%B3%BC%EC%A0%81%EC%9C%BC%EB%A1%9C-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0-4bd0e24e1bbd)
