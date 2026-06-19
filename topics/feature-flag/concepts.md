---
tags: [feature-flag, deployment, release-management, devops]
related: [kubernetes, distributed-systems, redis, system-design]
---

# Feature Flag — 개념 정리

→ [[home]] | [[topics/feature-flag/questions]] | [[topics/distributed-systems/concepts]] | [[topics/kubernetes/concepts]]

---

## 피처 플래그란?

코드 배포 없이 특정 기능을 런타임에 켜고 끌 수 있는 설정값.  
**"Deployment와 Release를 분리"**하는 것이 핵심 목적이다.

> 배포(Deploy) = 코드를 서버에 올리는 행위  
> 릴리즈(Release) = 사용자에게 기능을 여는 행위  
> 피처 플래그는 이 둘을 독립적으로 제어할 수 있게 한다.

```go
// 기본 구조
if featureFlag.IsEnabled("new-payment-flow", userID) {
    return newPaymentHandler(ctx, req)
}
return legacyPaymentHandler(ctx, req)
```

---

## 피처 플래그의 4가지 유형

Martin Fowler의 분류 기준 (수명과 동적성 기준으로 구분).

### 1. Release Flag (릴리즈 플래그)

가장 일반적인 유형. **미완성 기능을 main 브랜치에 머지**하되 플래그로 숨겨두는 용도.

- 수명: **단기** (기능 완성 후 제거)
- 목적: Trunk-based development 지원, 점진적 롤아웃
- 제거 시점: 100% 롤아웃 완료 후 즉시

```go
// 10%씩 증가하며 점진적 오픈
if flagService.Percentage("checkout-v2", userID) < 10 {
    return checkoutV2(ctx)
}
```

### 2. Experiment Flag (실험 플래그)

A/B 테스트, 멀티배리언트 실험용. Boolean이 아니라 **Variant**를 반환한다.

- 수명: **단기** (실험 결론 후 승자 선택, 플래그 제거)
- 목적: 데이터 기반 의사결정
- 특징: 사용자 그룹을 안정적으로 분리하는 해싱 필요

```go
variant := flagService.Variant("button-color", userID)
// "blue", "green", "red" 중 하나 반환
// 같은 userID는 항상 같은 variant (sticky assignment)
```

### 3. Ops Flag / Kill Switch (운영 플래그)

장애 발생 시 즉시 기능을 끄는 **비상 차단기**. 재배포 없이 수초 내 적용.

- 수명: **장기** (기능이 살아있는 한 유지)
- 목적: 안정성 확보, 부하 제어
- 특징: 변경이 즉시 전파되어야 함 (캐시 TTL 최소화)

```go
// 외부 API 장애 시 fallback 전환
if flagService.IsEnabled("use-external-payment-api") {
    return externalPayment(ctx, req)
}
return internalPaymentFallback(ctx, req)
```

### 4. Permission Flag (권한 플래그)

사용자 속성(플랜, 역할, 지역)에 따라 기능 접근을 제어.

- 수명: **영구** (제품 권한 시스템의 일부)
- 목적: 요금제별 기능 차별화, GDPR 지역 제한
- 특징: 비즈니스 로직과 강하게 결합 → 제거 불가

```go
// Pro 플랜 사용자에게만 고급 분석 기능 제공
if flagService.HasPermission("advanced-analytics", user.Plan) {
    return advancedAnalytics(ctx)
}
return basicAnalytics(ctx)
```

---

## 구현 방식 비교

| 방식 | 동적 변경 | 분산 일관성 | 적합한 사용처 |
|------|-----------|-------------|---------------|
| 환경변수 | 불가 (재시작 필요) | - | 단순 on/off, 인프라 설정 |
| DB (MySQL/PostgreSQL) | 가능 | 즉시 반영 가능 | 소규모, 단순 구현 |
| Redis | 가능 | TTL 기반 지연 | 고성능, 분산 환경 |
| 외부 SaaS | 가능 | SDK 폴링 주기 | 대규모, 다양한 타겟팅 |

### 자체 구현 (Redis 기반)

```go
type FlagService struct {
    redis  *redis.Client
    cache  sync.Map        // 로컬 인메모리 캐시
    ttl    time.Duration   // 캐시 TTL (일반적으로 30s~5m)
}

func (f *FlagService) IsEnabled(flag string, userID string) bool {
    cacheKey := flag + ":" + userID

    // 1. 로컬 캐시 우선 조회 (Redis 부하 감소)
    if val, ok := f.cache.Load(cacheKey); ok {
        return val.(bool)
    }

    // 2. Redis 조회
    val, err := f.redis.Get(ctx, "feature:"+flag).Result()
    if err != nil {
        return false // fail-closed (기본값 off)
    }

    enabled := f.evaluateFlag(val, userID)
    f.cache.Store(cacheKey, enabled)

    // TTL 후 캐시 만료
    go func() {
        time.Sleep(f.ttl)
        f.cache.Delete(cacheKey)
    }()

    return enabled
}
```

### 주요 외부 서비스

| 서비스 | 특징 |
|--------|------|
| **LaunchDarkly** | 가장 성숙한 엔터프라이즈급, 강력한 타겟팅 |
| **Unleash** | 오픈소스, 자체 호스팅 가능 |
| **GrowthBook** | A/B 테스트 특화, 오픈소스 |
| **ConfigCat** | 간단한 설정, 합리적인 가격 |

---

## 배포 전략과의 연계

### Trunk-based Development + Release Flag

```
main 브랜치
    │
    ├── feat: 새 결제 API 추가 (flag=false로 비활성)
    ├── feat: 결제 UI 변경 (flag=false)
    ├── fix: 결제 버그 수정
    │
    └── [플래그 ON] → 사용자에게 새 결제 흐름 노출
```

장점: 장기 feature 브랜치 없이 지속적으로 main에 통합 → 머지 충돌 최소화.

### 카나리 배포 vs 피처 플래그

| 구분 | 카나리 배포 | 피처 플래그 |
|------|-------------|-------------|
| 제어 단위 | 서버 인스턴스 | 사용자/요청 단위 |
| 롤백 속도 | 트래픽 라우팅 변경 (~분) | 플래그 OFF (~초) |
| 인프라 필요 | 복수 인스턴스 | 단일 인스턴스 가능 |
| 특정 유저 타겟 | 불가 | 가능 |

실무에서는 **두 가지를 함께** 사용: 카나리로 인스턴스 안정성 확인 + 플래그로 사용자 단위 점진적 오픈.

---

## 점진적 롤아웃 (Progressive Delivery)

```
1% → 5% → 10% → 25% → 50% → 100%

각 단계에서:
- 에러율 모니터링
- 응답시간 p99 확인
- 사용자 행동 지표 측정
- 이상 시 즉시 0%로 롤백
```

**사용자 스티키 할당 (Sticky Assignment)**:  
같은 사용자는 항상 같은 variant를 받아야 함.  
`hash(userID + flagName) % 100` 으로 결정적(deterministic) 할당.

```go
func (f *FlagService) Percentage(flag string, userID string) int {
    h := fnv.New32a()
    h.Write([]byte(flag + ":" + userID))
    return int(h.Sum32() % 100)
}
```

---

## 타겟팅 전략

단순 on/off를 넘어 **세그먼트 기반 타겟팅**:

```go
type FlagRule struct {
    // 조건들 (AND 연산)
    UserIDs     []string  // 특정 사용자
    Countries   []string  // 지역 제한
    Plans       []string  // 요금제
    Percentage  int       // 퍼센트 기반
    Emails      string    // 이메일 패턴 (내부 테스터: @company.com)
}
```

**내부 테스터 우선 공개 패턴**:
1. 단계 1: 내부 직원만 (이메일 기반)
2. 단계 2: 베타 사용자 (userID 목록)
3. 단계 3: 특정 국가 (지역 기반)
4. 단계 4: 퍼센트 기반 점진적 확대

---

## 분산 환경에서의 일관성 문제

### 캐시 일관성 (Cache Consistency)

플래그 변경 후 모든 인스턴스에 즉시 반영되지 않는 문제.

```
플래그 변경
    │
    ├── Instance A: 캐시 TTL 5분 → 5분 후 반영
    ├── Instance B: 캐시 TTL 5분 → 2분 후 반영 (타이밍 차이)
    └── Instance C: 캐시 TTL 5분 → 5분 후 반영

→ 같은 사용자의 요청이 다른 인스턴스에 가면 다른 경험
```

**해결 방법**:
1. **TTL 단축**: Ops 플래그는 TTL 0~10초 (즉시 반영 우선)
2. **Pub/Sub 무효화**: Redis Pub/Sub으로 플래그 변경 시 모든 인스턴스 캐시 즉시 삭제
3. **SSE/WebSocket 스트림**: LaunchDarkly 방식 — 변경 발생 시 실시간 푸시

```go
// Redis Pub/Sub으로 캐시 즉시 무효화
func (f *FlagService) subscribeInvalidation() {
    pubsub := f.redis.Subscribe(ctx, "feature-flag:invalidate")
    for msg := range pubsub.Channel() {
        f.cache.Delete(msg.Payload) // 특정 플래그 캐시 삭제
    }
}
```

### 평가 일관성 (Evaluation Consistency)

한 요청 내에서 같은 플래그를 여러 번 평가할 때 결과가 달라지는 문제.  
→ **요청 스코프 캐시**: 동일 요청 내에서는 Context에 플래그 결과 저장.

---

## Flag Debt & Hygiene (플래그 부채)

피처 플래그의 가장 큰 위험: **제거하지 않으면 코드가 썩는다**.

### 부채 발생 시 문제

- 코드 복잡도 폭발 (n개 플래그 → 2^n 조합)
- 테스트 케이스 급증
- 신규 개발자 온보딩 어려움
- 실제 사례: **나이트 캐피탈 그룹** — 2012년 구형 플래그 코드 실수로 45분에 4.6억 달러 손실

### 라이프사이클 관리

```
[생성] → [개발] → [테스트] → [점진적 오픈] → [100%] → [제거]
  │                                                          │
  └──── 생성 시 제거 티켓 동시 생성 ───────────────────────┘
```

**플래그 생성 시 반드시 함께 해야 할 것들**:
1. 만료일 설정 (Release 플래그: 2~4주, Experiment: 실험 기간)
2. 제거 티켓 생성 (JIRA/Linear)
3. 오너 지정 (누가 책임지고 제거할지)

### 제거 시점 판단 기준

| 플래그 유형 | 제거 시점 |
|-------------|-----------|
| Release Flag | 100% 롤아웃 완료 + 안정 확인 후 |
| Experiment Flag | 실험 종료 + 승자 결정 후 |
| Ops Flag | 해당 기능 폐기 시 |
| Permission Flag | 요금제 구조 변경 시 |

---

## 모니터링 & 관찰 가능성

플래그 평가에 대한 모니터링이 필수:

```go
// 플래그 평가 시 메트릭 기록
func (f *FlagService) IsEnabled(flag string, userID string) bool {
    start := time.Now()
    result := f.evaluate(flag, userID)

    // 메트릭 기록
    metrics.Counter("feature_flag.evaluation",
        "flag", flag,
        "result", strconv.FormatBool(result),
    ).Inc()
    metrics.Histogram("feature_flag.latency_ms").Observe(
        float64(time.Since(start).Milliseconds()),
    )

    return result
}
```

**모니터링 항목**:
- 플래그 평가 횟수 (트래픽 분포 확인)
- 각 variant별 에러율, 응답시간 비교
- 플래그 서비스 자체 장애 감지 (fail-safe 동작 확인)

---

## Fail-Safe 전략

플래그 서비스 자체가 다운될 때의 동작:

- **Fail-Open**: 서비스 불가 시 기능 활성화 (가용성 우선)
- **Fail-Closed**: 서비스 불가 시 기능 비활성화 (안전성 우선, 권장)

```go
func (f *FlagService) IsEnabled(flag string, userID string) bool {
    val, err := f.fetchFlag(flag, userID)
    if err != nil {
        // Fail-closed: 기본값 false 반환
        log.Warn("flag service unavailable, defaulting to false", "flag", flag)
        return false
    }
    return val
}
```

---

## 참고 링크

- [Martin Fowler — Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Feature Flags Best Practices 2026 — ConfigCat](https://configcat.com/feature-flag-best-practices/)
- [Feature Flags and Progressive Delivery — Zylos Research](https://zylos.ai/research/2026-02-12-feature-flags)
- [LaunchDarkly — Technical Debt Management](https://launchdarkly.com/docs/guides/flags/technical-debt)
- [4 Types of Feature Flags — Octopus Deploy](https://octopus.com/devops/feature-flags/)
