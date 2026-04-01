---
type: harness-status
updated: 2026-04-01
---

# 현재 준비 상태 스냅샷

> 이 파일은 Claude가 세션 시작/종료 시 자동으로 갱신한다.
> 면접 준비 현황을 AI가 항상 인지하게 하는 Dynamic Context.

---

## 진행 중인 공고

| 회사 | 포지션 | 마지막 세션 | 상태 |
|---|---|---|---|
| 화이트큐브 | 챌린저스 백엔드 | 2026-03-31 | 준비 중 |
| wag | 백엔드 | 2026-03-31 | 준비 중 |

---

## 기술별 준비 수준

> 세션 피드백 기반으로 업데이트. 5점 만점.

| 기술 | 수준 | 마지막 확인 | 취약 포인트 |
|---|---|---|---|
| golang/goroutine | ★★★★★ | 2026-03-27 | - |
| golang/channel | ★★★★☆ | 2026-03-27 | select + context 조합 |
| golang/map | ★★★★☆ | 2026-03-27 | concurrent 접근 패턴 |
| golang/interface | ★★★★☆ | 2026-04-01 | Accept interfaces, return structs 관용구, 외부 라이브러리 연동 예시 |
| golang/error-handling | ★★★★☆ | 2026-03-31 | 도메인 에러 교체(convert) 패턴, gin API 구문 정확도 |
| golang/clean-architecture | ★★★★☆ | 2026-03-31 | 레이어별 에러 변환 흐름 정착 |
| golang/hexagonal | ★★★☆☆ | 2026-03-27 | Port/Adapter 실제 코드 작성 |
| mongodb | ★★★☆☆ | 2026-03-31 | 복합 인덱스 순서, Aggregation Pipeline 개념 |
| python-fastapi | ★★★☆☆ | 2026-03-31 | asyncio vs goroutine 비교, ProcessPoolExecutor |
| java/spring | ★★★★☆ | 2026-03-31 | checked exception rollback, 별도 클래스 분리 패턴 |
| java/jpa | ★★★★☆ | 2026-04-01 | fetch join+pagination @BatchSize 해결 정착, @EntityGraph 선언 위치 교정 |
| networking | ★★★★☆ | 2026-04-01 | TLS Handshake 흐름 2회차에 완전 교정. Client Hello 용어 추가 필요 |
| mysql | ★★★★☆ | 2026-04-01 | ACID 정의 정확. Next-Key Lock(Gap Lock) Phantom Read 방지 심화 추가 필요 |
| redis | ★★★★☆ | 2026-04-01 | Hash vs String 선택 기준(메모리 효율, TTL 제한) 보완 필요 |
| kafka | ★★★☆☆ | 2026-04-01 | Exactly-Once 흐름 교정. sendOffsetsToTransaction 두 실패 케이스 정착 |
| kotlin | ★★★☆☆ | 2026-04-01 | suspend fun Continuation 미인지, Dispatcher 타입 오개념(싱글스레드 아님) |
| kubernetes | ★★★☆☆ | 2026-04-01 | Pod/Service/Deployment 기본 맞음. Label Selector, Service 타입 추가 필요 |
| zookeeper | ★★★★☆ | 2026-04-01 | ephemeral/Watch 이력서 연결 강점. Watch 1회성 특성 추가 필요 |
| distributed-systems | ★★☆☆☆ | 2026-04-01 | CAP 정리 전혀 모름 → 즉시 암기 필요 |
| elasticsearch | ★★☆☆☆ | - | 전반적으로 부족 |

---

## 다음 우선순위

1. `distributed-systems` (CAP 정리) — 오늘 세션에서 전혀 모름. 즉시 암기 최우선
2. `kotlin` — suspend fun Continuation, Dispatcher 타입, launch vs async 오개념 교정
3. `kubernetes` — Label Selector, Service 타입(ClusterIP/NodePort/LoadBalancer)
4. `redis` — Hash vs String 선택 기준 심화, 각 자료구조 패턴
5. `elasticsearch` — 공고 요구사항, 미정리
6. `golang/hexagonal` — 실제 코드 작성 연습 필요

---

## 누적 세션 수

| 회사 | 세션 수 | 마지막 피드백 요약 |
|---|---|---|
| 화이트큐브 | 2 | Go 에러 핸들링, MongoDB, Python FastAPI — 실무 기반 설명 강점 |
| wag | 1 | Java/Spring @Transactional, 동시성, JPA — 동시성 제어 최강점, 수치 정확도 보완 필요 |

---

## Harness 신호 (개선 필요 항목)

> Claude가 답변을 못 하거나 잘못된 방향으로 갔을 때 여기에 기록.
> 다음 세션 전에 해당 topics 파일을 보강한다.

- **수치 정확도**: N+1 쿼리 수 "200→100"(오) → "1+N=101→1"(정), 인덱스 순서 `(createdAt, roomId)` → `(roomId, createdAt)`, `%w:%w` 이중 래핑은 비표준
- **gin API 구문**: `gin.H{http.NotFound,...}` → `c.JSON(http.StatusNotFound, gin.H{...})`
- **도메인 에러 패턴**: Service에서 `errors.Is` 후 래핑 아닌 교체(return ErrXxx)
- **레이어 분리**: Repository는 `fmt.Errorf("funcName: %w", err)` prefix 추가 권장
