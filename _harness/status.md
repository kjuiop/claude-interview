---
type: harness-status
updated: 2026-03-31
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
| golang/error-handling | ★★★★☆ | 2026-03-31 | 도메인 에러 교체(convert) 패턴, gin API 구문 정확도 |
| golang/clean-architecture | ★★★★☆ | 2026-03-31 | 레이어별 에러 변환 흐름 정착 |
| golang/hexagonal | ★★★☆☆ | 2026-03-27 | Port/Adapter 실제 코드 작성 |
| mongodb | ★★★☆☆ | 2026-03-31 | 복합 인덱스 순서, Aggregation Pipeline 개념 |
| python-fastapi | ★★★☆☆ | 2026-03-31 | asyncio vs goroutine 비교, ProcessPoolExecutor |
| java/spring | ★★★★☆ | 2026-03-31 | checked exception rollback, 별도 클래스 분리 패턴 |
| java/jpa | ★★★☆☆ | 2026-03-31 | N+1 쿼리 수(1+N=N+1), @EntityGraph, pagination 주의 |
| distributed-systems | ★★★★☆ | 2026-03-31 | 동시성 제어(낙관/비관적 락, 원자적 업데이트) |
| redis | ★★★★☆ | 2026-03-31 | Redis-DB 정합성, Redisson |
| kafka | ★★☆☆☆ | - | 전반적으로 부족 |
| elasticsearch | ★★☆☆☆ | - | 전반적으로 부족 |
| kubernetes | ★★★☆☆ | - | HPA, 장애 대응 |

---

## 다음 우선순위

1. `kafka` — 오늘 세션 이후 2회 연속 "다음 주제" 지정, 최우선
2. `java/jpa` — N+1 쿼리 수 정확히 암기, @EntityGraph, pagination 주의
3. `elasticsearch` — 공고 요구사항, 미정리
4. `golang/hexagonal` — 실제 코드 작성 연습 필요
5. `kubernetes` — 갭 분석에서 취약점으로 식별

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
