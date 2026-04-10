---
tags: [golang, go, map, hmap, concurrent, sync]
related: [concurrency, goroutine]
---

# Golang — Map

→ [[home]] | [[topics/golang/concurrency]] | [[topics/golang/goroutine]]

---

### 💬 면접 답변 형태로 읽기

Go의 map은 내부적으로 `*hmap` 포인터를 갖는 참조 타입입니다. `make(map[K]V)`를 호출하면 런타임이 힙에 `hmap` 구조체를 할당하고 변수는 그 포인터를 갖습니다. 따라서 map을 함수에 전달하거나 다른 변수에 대입하면 같은 hmap을 가리키는 포인터만 복사되어 양쪽에서 동일한 데이터를 수정하게 됩니다. 반면 `int`, `bool`, `string`, `array`, `struct` 같은 값 타입은 할당이나 함수 전달 시 값 전체가 복사되어 원본과 독립됩니다.

map 내부 구조를 보면, hmap은 버킷 배열을 관리하며 버킷 하나에 최대 8개의 키-값 쌍이 들어갑니다. 키 해시의 상위 8비트인 tophash를 먼저 비교해 탐색을 최적화하고, 버킷이 꽉 차면 overflow bucket으로 체이닝합니다. 로드팩터가 기본값 6.5를 초과하면 버킷을 두 배로 늘려 rehash합니다.

nil map과 empty map의 차이도 중요합니다. `var m map[K]V`로 선언만 하면 hmap이 할당되지 않은 nil map이 됩니다. nil map에서 읽기는 zero value를 반환해 안전하지만, 쓰기를 시도하면 `assignment to entry in nil map` 런타임 panic이 발생합니다. struct 필드로 map을 선언만 하고 초기화하지 않은 채 쓰기를 시도하는 것이 대표적인 실수 패턴으로, `make(map[K]V)`로 먼저 초기화해야 합니다.

map의 순회 순서가 비결정적인 이유는 Go 런타임이 `range` 순회 시작 위치를 의도적으로 무작위화하기 때문입니다. 개발자가 순서에 의존하는 코드를 작성하지 못하도록 Go 1.0부터 도입된 정책이며, 버킷 해시 구조상 삽입 순서를 보장할 수 없고 rehash 시 버킷이 재배치되는 구조적 이유도 있습니다. 정렬 순회가 필요하면 키를 slice로 추출해 정렬한 후 순회해야 합니다.

concurrent 접근에서 Go map은 thread-safe하지 않습니다. 여러 goroutine이 동시에 읽기/쓰기를 하면 Go 1.6 이후 런타임이 `concurrent map read and map write` panic을 발생시킵니다. 해결책은 세 가지로, `sync.RWMutex`는 읽기를 동시에 허용하고 쓰기만 배타 락을 걸어 범용적으로 사용하기 좋습니다. `sync.Map`은 내부적으로 read-only map과 dirty map을 유지해 읽기가 압도적일 때 lock-free 읽기로 성능이 우수하지만, 쓰기가 빈번하면 dirty 맵 승격 비용으로 오히려 `RWMutex`보다 느릴 수 있습니다. 쓰기가 heavy하고 고성능이 필요하다면 map을 N개 샤드로 분할하고 샤드별 뮤텍스를 두는 Sharded Map 패턴을 사용합니다. 일반적인 상황에서는 동작이 예측 가능한 `sync.RWMutex`를 기본으로 선택하는 것이 적합합니다.

---

## 값 타입 vs 참조 타입

### 값 타입 (Value Type)
변수 할당·함수 전달 시 값 전체가 **복사**됨.

| 타입 | 비고 |
|---|---|
| `int`, `float64`, `bool` | 기본 숫자/논리 타입 |
| `string` | 불변 바이트 시퀀스 |
| `array` | `[N]T` — 길이가 타입의 일부, 전체 복사 |
| `struct` | 필드 전체 복사 (필드에 포인터가 있으면 shallow copy 주의) |

### 참조 타입 (Reference Type)
변수에 메모리 주소(포인터)가 저장됨.

| 타입 | 내부 구조 |
|---|---|
| `map` | `*hmap` 포인터 |
| `slice` | `{ptr, len, cap}` 헤더 |
| `channel` | `*hchan` 포인터 |
| `pointer` | 직접 주소 |
| `interface` | `{타입 포인터, 데이터 포인터}` |
| `function` | 함수 포인터 |

```go
// 값 타입: 복사 후 독립
a := [3]int{1, 2, 3}
b := a
b[0] = 99
fmt.Println(a[0]) // 1 — 원본 유지

// 참조 타입: 같은 메모리 공유
m1 := map[string]int{"a": 1}
m2 := m1
m2["a"] = 99
fmt.Println(m1["a"]) // 99 — m1도 변경됨!
```

---

## Map 내부 구조 (hmap)

`make(map[K]V)` 호출 시 런타임이 힙에 `hmap` 구조체를 할당하고, map 변수는 그 포인터를 갖는다.

```
hmap 주요 필드:
- count      : 현재 저장된 키-값 쌍 수
- B          : 버킷 수의 지수 (buckets = 2^B 개)
- buckets    : bmap(버킷) 배열 포인터
- oldbuckets : rehash 중 이전 버킷 포인터
- loadFactor : 부하율 기준 (기본 6.5)
```

**bmap(버킷) 구조:**
- 버킷 1개 = 최대 **8개** 키-값 쌍
- 키 해시의 상위 8비트(`tophash`)를 먼저 비교 → 탐색 최적화
- 버킷이 꽉 차면 overflow bucket으로 체이닝

> 핵심: map 변수는 결국 `*hmap` 포인터이므로, 함수에 전달해도 같은 맵을 수정하게 됨.

---

## nil map vs empty map

```go
var m1 map[string]int        // nil map (zero value)
m2 := make(map[string]int)   // empty map
m3 := map[string]int{}       // empty map
```

| 구분 | nil map | empty map |
|---|---|---|
| 읽기 | 안전 (zero value 반환) | 안전 |
| 쓰기 | **panic: assignment to entry in nil map** | 안전 |
| `len()` | 0 | 0 |
| `== nil` | `true` | `false` |

```go
// 흔한 실수 패턴
type Config struct {
    Tags map[string]string
}
c := Config{}
c.Tags["env"] = "prod" // panic! Tags가 nil map
// 해결: c.Tags = make(map[string]string) 먼저
```

---

## map 순회 순서가 non-deterministic인 이유

Go 런타임이 순회 시작 위치를 **의도적으로 무작위화** (Go 1.0부터 정책).

이유:
1. 순서에 의존하는 코드 작성을 원천 차단
2. 버킷 해시 구조상 삽입 순서 보장 불가
3. rehash 시 버킷 재배치로 순서 변동

```go
// 정렬 순회가 필요한 경우
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

---

## concurrent map 접근 — Race Condition

map은 기본적으로 **thread-safe하지 않음**. 여러 goroutine이 동시에 읽기/쓰기하면 런타임 패닉 또는 데이터 손상.

Go 1.6+에서 동시 접근 감지 시 런타임이 `concurrent map read and map write` 패닉 발생.

**해결책:**

| 방법 | 특징 | 적합한 시나리오 |
|---|---|---|
| `sync.RWMutex` | 읽기 동시 허용, 쓰기 배타 락 | 일반적인 읽기/쓰기 혼재 |
| `sync.Map` | 읽기 heavy에 최적화, lock-free 읽기 | 읽기 압도적, 키가 자주 바뀌지 않음 |
| Sharded Map | map을 N샤드로 분할, 샤드별 뮤텍스 | 쓰기 heavy, 고성능 필요 |

```go
// sync.RWMutex 패턴 (가장 범용)
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}
func (s *SafeMap) Get(key string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}
func (s *SafeMap) Set(key string, val int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = val
}

// sync.Map 패턴 (읽기 heavy)
var sm sync.Map
sm.Store("key", 42)
val, ok := sm.Load("key")
sm.Range(func(k, v any) bool {
    fmt.Println(k, v)
    return true
})
```

**sync.Map vs sync.RWMutex+map:**
- `sync.Map`은 내부적으로 두 개의 맵(read-only, dirty)을 유지. 읽기가 압도적일 때 유리하지만 메모리 오버헤드가 있음.
- 쓰기가 빈번하면 dirty 맵 승격(promotion) 비용으로 오히려 느릴 수 있음.
- 일반적인 상황에서는 `sync.RWMutex`가 더 예측 가능한 성능을 냄.

---

## 면접 질문

**Q. Go에서 값 타입과 참조 타입의 차이를 설명하고, map이 어느 쪽인지 이유와 함께 말해보세요.**

Go의 타입은 값 타입과 참조 타입으로 나뉩니다. 값 타입인 `int`, `bool`, `string`, `array`, `struct`는 변수를 다른 변수에 대입하거나 함수에 전달할 때 값 전체가 복사되어 원본과 독립적으로 존재합니다. 반면 참조 타입인 `map`, `slice`, `channel`, `pointer`는 내부적으로 힙 데이터를 가리키는 포인터를 갖고 있어, 복사하더라도 같은 힙 데이터를 공유합니다. map이 참조 타입인 구체적인 이유는, `make(map[K]V)`를 호출하면 런타임이 힙에 `hmap` 구조체를 할당하고 map 변수는 그 `*hmap` 포인터를 값으로 갖기 때문입니다. 따라서 `m2 := m1`으로 map을 복사하면 두 변수가 동일한 hmap을 가리키게 되어 하나에서 수정하면 다른 쪽에도 반영됩니다. slice와 비교하면, slice는 내부적으로 `{포인터, len, cap}` 헤더가 복사되지만 `append`로 새 배열이 할당되면 두 slice가 독립됩니다. struct에 map 필드가 있을 때 struct를 복사하면 struct 자체는 복사되지만 map 필드는 shallow copy로 동일한 hmap을 가리킵니다.

**꼬리 질문 예시**:
- slice도 참조 타입인데, `m2 := m1`(map)과 `s2 := s1`(slice) 복사의 동작 차이? (map은 완전히 동일한 hmap 공유, slice는 SliceHeader가 복사되지만 내부 배열 포인터는 공유 — append 시 동작 차이)
- struct에 map 필드가 있을 때 struct를 복사하면 map도 복사되나요? (shallow copy — map 필드는 포인터 공유)

---

**Q. nil map과 empty map의 차이는 무엇인가요?**

nil map과 empty map은 모두 `len()`이 0을 반환하고 `for range`로 순회해도 아무 일도 일어나지 않아 비슷해 보이지만, 결정적인 차이가 있습니다. `var m map[K]V`로 선언만 한 nil map은 힙에 hmap이 전혀 할당되지 않은 상태입니다. nil map에서 읽기를 하면 zero value를 반환하므로 안전하지만, 쓰기를 시도하면 `assignment to entry in nil map` 런타임 panic이 즉시 발생합니다. `make(map[K]V)` 또는 `map[K]V{}`로 생성한 empty map은 빈 hmap이 할당된 상태로 읽기와 쓰기 모두 안전합니다. 실무에서 가장 흔한 실수 패턴은 struct 필드로 map을 선언만 하고 초기화하지 않은 채 쓰기를 시도하는 것입니다. 이 경우 반드시 `make(map[K]V)`로 먼저 초기화해야 합니다.

---

**Q. Go map의 순회 순서가 비결정적인 이유는?**

Go map의 순회 순서가 비결정적인 것은 우연이 아니라 Go 1.0부터 의도적으로 설계된 정책입니다. Go 런타임은 `range`로 map을 순회할 때 시작 버킷과 시작 위치를 무작위로 선택합니다. 이렇게 설계한 이유는 크게 두 가지입니다. 첫째, 개발자가 map 순회 순서에 의존하는 코드를 작성하지 못하도록 원천적으로 차단하기 위해서입니다. 만약 특정 구현에서 우연히 순서가 일정하게 나왔다면 그것에 의존하는 버그가 생길 수 있습니다. 둘째, hmap의 버킷 해시 구조상 삽입 순서를 보장할 수 없고, rehash 시 버킷이 재배치되면 순서가 달라집니다. 정렬된 순회가 필요한 경우에는 키를 slice로 추출한 뒤 `sort.Strings`로 정렬하고 순회하는 방식을 명시적으로 작성해야 합니다.

---

**Q. 여러 goroutine이 동시에 map을 읽고 쓰면 어떤 문제가 발생하고, 어떻게 해결하나요?**

Go의 map은 기본적으로 thread-safe하지 않습니다. 여러 goroutine이 동시에 읽고 쓰면 데이터 손상이 발생하거나 Go 1.6 이후 런타임이 `concurrent map read and map write` panic을 발생시킵니다. race condition이 발생하면 항상 즉각적으로 panic이 나는 것이 아니라 조용히 데이터가 손상되는 경우도 있어서 더 위험합니다. 해결책은 접근 패턴에 따라 세 가지 중에서 선택합니다. `sync.RWMutex`로 map을 감싸는 방식은 읽기는 동시에 허용하고 쓰기만 배타 락을 걸어 범용적으로 예측 가능한 성능을 냅니다. `sync.Map`은 내부적으로 read-only atomic map과 dirty map(mutex 보호)을 유지해 읽기가 압도적으로 많고 키가 자주 바뀌지 않는 상황에서 lock-free 읽기로 성능이 우수합니다. 단 쓰기가 빈번하면 dirty 맵 승격 비용으로 오히려 `RWMutex`보다 느려질 수 있으므로 주의해야 합니다. 쓰기가 heavy하고 최고 성능이 필요하다면 map을 N개 샤드로 분할하고 샤드별 뮤텍스를 두는 Sharded Map 패턴을 고려합니다. race condition 검출은 `go test -race ./...`로 레이스 디텍터를 활성화해 테스트 단계에서 발견하는 것이 가장 효과적입니다.

**꼬리 질문 예시**:
- `-race` 플래그로 race condition을 어떻게 검출하나요? (`go test -race ./...`)

**면접 세션 피드백 (2026-04-02 2회차)**:
- 잘한 점: nil map panic을 "hmap 미할당 → 참조형"으로 정확히 설명. map+mutex 선택 기준(read-modify-write 원자적 보호) 정확.
- 보완:
  - hash bucket 구조 미언급: 버킷 1개 = 최대 8개 key-value, 로드팩터 6.5 초과 시 2배 rehash
  - sync.Map 내부 구조: read map(atomic, lock-free) + dirty map(mutex). read-heavy 최적, write-heavy에서 dirty 승격 오버헤드로 느릴 수 있음
  - 선택 기준 정리: sync.Map = 읽기 압도적 + 키 변경 드문 경우 / map+RWMutex = 범용, 쓰기 빈번한 경우

---

## 참고 링크
- [Go Map Internals](https://medium.com/@omidahn/how-do-maps-work-in-go-19da057a2f57)
- [Go sync.Map Guide](https://victoriametrics.com/blog/go-sync-map/)
