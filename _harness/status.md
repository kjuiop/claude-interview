---
type: harness-status
updated: 2026-05-05
---

# 현재 준비 상태 스냅샷

> 이 파일은 Claude가 세션 시작/종료 시 자동으로 갱신한다.
> 면접 준비 현황을 AI가 항상 인지하게 하는 Dynamic Context.

---

## 진행 중인 공고

| 회사 | 포지션 | 마지막 세션 | 상태 |
|---|---|---|---|
| 화이트큐브 | 챌린저스 백엔드 | 2026-04-10 | 준비 중 |
| 넵튠 | 솔루션개발실 백엔드 | 2026-04-10 | 신규 등록 |
| 인포뱅크 | iXpert AI 프로덕트 엔지니어 | 2026-04-17 | 준비 중 |
| 채널톡 | Enterprise Solution Engineer (AX팀) | 2026-04-17 | 준비 중 |
| 인라이플 | 백엔드 개발자(JAVA) | 2026-04-30 | 면접 2026-05-01(목) |

---

## 기술별 준비 수준

> 세션 피드백 기반으로 업데이트. 5점 만점.

| 기술 | 수준 | 마지막 확인 | 취약 포인트 |
|---|---|---|---|
| golang/goroutine | ★★★★★ | 2026-03-27 | - |
| golang/channel | ★★★★★ | 2026-04-14 | nil channel 동적 비활성화·drop count→재연결 흐름 완성. select+default non-blocking send 패턴 코드 즉시 제시 능력 확인 |
| golang/map | ★★★★★ | 2026-04-12 | hash bucket tophash 구조, sync.Map read/dirty/promote 패턴 교정 완료 |
| golang/interface | ★★★★☆ | 2026-04-01 | Accept interfaces, return structs 관용구, 외부 라이브러리 연동 예시 |
| golang/error-handling | ★★★★☆ | 2026-04-10 | gin 구문 `c.JSON(http.StatusXxx, gin.H{"error":"..."})` 반복 오류 — 구문 암기 최우선 |
| golang/clean-architecture | ★★★★☆ | 2026-03-31 | 레이어별 에러 변환 흐름 정착 |
| golang/hexagonal | ★★★★★ | 2026-04-12 | In-Memory=stateful, Adapter→Port 방향, DIP 전부 교정 완료 |
| mongodb | ★★★☆☆ | 2026-04-29 | $match/group/sort/limit 역할 파악. $group 문법(`_id: "$field"`, `$sum: 1`, `$` 접두사) + $project 역할(SELECT 절) 아직 미암기(5/10) |
| python-fastapi | ★★★★☆ | 2026-04-29 | yield setup/teardown + 요청 스코프 격리 메커니즘 8/10 해결 ✅. "각 요청이 독립된 인스턴스 → 세션 오염 없음 = 요청 스코프" 결론 표현 추가 필요 |
| java/jpa | ★★★★☆ | 2026-04-01 | fetch join+pagination @BatchSize 해결 정착, @EntityGraph 선언 위치 교정 |
| networking | ★★★★☆ | 2026-04-10 | TLS 1.3 흐름·ECDHE PFS 정착. Forward Secrecy 꼬리 질문 "모르겠습니다" → topics 보강 완료 |
| mysql | ★★★★☆ | 2026-04-30 | 복합 인덱스 컬럼 순서 7/10 — 등치→범위 원칙·결론 정확. B-Tree 선두 범위 시 이후 컬럼 인덱스 미사용 이유 미언급. 지연 조인 "커버링 인덱스로 처리" 표현 아직 미언급. |
| redis | ★★★★☆ | 2026-04-10 | Hash vs String 선택 기준 이해. ziplist/listpack 인코딩 임계값(128/64) + Redis 7.4 HEXPIRE 버전 정확도 보완 필요 |
| kafka | ★★★★★ | 2026-05-05 | exactly-once 8/10(Consumer 비원자성 표현 보완 필요), Producer 배치 9/10 ✅, Consumer lag 9/10 ✅. auto.offset.reset 동작 조건 별도 확인 필요 |
| rabbitmq | ★★★★☆ | 2026-04-28 | DLQ 3조건(TTL/NACK+requeue=false/x-max-length) + x-dead-letter-exchange 속성명 + x-death 헤더(count/reason/queue/exchange) 4회 연속 블로킹 드디어 해결 ✅ |
| clickhouse | ★★★★☆ | 2026-05-05 | ReplacingMergeTree 9/10 ✅(5회차 재출제). FINAL=쿼리 시점 강제 병합·단일 스레드, AggregatingMergeTree, event_id 멱등성 패턴 완성. eventual consistency 표현 추가 연습 필요 |
| mysql | ★★★★★ | 2026-05-05 | EXPLAIN type/key/rows/Extra 9/10 ✅. type 세부 단계(range/eq_ref/const) 추가 암기 필요 |
| java/spring-batch | ★★★★☆ | 2026-05-02 | faultTolerant skip/retry 코드 패턴 9/10 ✅. skip=영구오류/retry=일시오류 케이스 구분 완성. "재처리 범위가 커진다" 표현 추가 연습 필요. |
| kotlin | ★★★★☆ | 2026-04-02 | IO 스레드 수 수치 교정(max(64,cores)), Unconfined 동작 보완, "이벤트 루프" 표현 지양 |
| kubernetes | ★★★★☆ | 2026-05-03 | readiness/liveness/startupProbe 구분 완성 ✅(NotReady vs 재시작 vs CrashLoopBackOff). HPA 8/10 — 공식 ×100 오류 교정, PDB voluntary disruption(배포 포함) 꼬리 후 교정 완료. desiredReplicas 공식 정확한 표현 추가 연습 필요. |
| zookeeper | ★★★★★ | 2026-05-06 | Watch 1회성 + reporter 재등록 완성 ✅. 트랜스코더 Event-Driven 경험 9/10 |
| java/async | ★★☆☆☆ | 2026-05-06 | @Async ThreadPoolTaskExecutor·CompletableFuture 체이닝(thenApply/thenCombine)·I/O vs CPU bound 이유 미암기 (2/10 → 재출제 필요) |
| distributed-systems | ★★★☆☆ | 2026-04-12 | 2PC 전혀 모름 — Phase 1/2 흐름, Blocking 원인, 3PC 차이 암기 필요. Saga 선택 이유 방향은 알고 있음 |
| elasticsearch | ★★★★☆ | 2026-04-28 | Analyzer 3단계 7/10 해결. Term Dictionary 이진탐색(O(log N)) + Posting List TF/position/offset 추가 암기 필요 |
| postgresql | ★★★☆☆ | 2026-04-28 | MVCC/Dead Tuple/VACUUM 6/10. XID Wraparound(32비트 트랜잭션 ID 한계, 쓰기 전면 차단) 개념 인지. xmin/xmax 내부 구조 추가 학습 필요 |
| java/spring | ★★★★☆ | 2026-05-05 | @Transactional 9/10 ✅(suspend 표현 추가 필요). 멀티 모듈 7/10 — Gradle implementation vs api 전이 의존성 미암기. WebFlux 6/10 — "I/O 완료 콜백 재개" 표현 + R2DBC 미언급 반복 |

---

## 다음 우선순위

1. `java/async` — @Async(ThreadPoolTaskExecutor) vs CompletableFuture(thenApply/thenCombine) 차이, I/O bound vs CPU bound 이유 암기 (2/10 → 재출제 최우선)
2. `java/concepts` — 싱글톤 thread-safe 3가지: synchronized(성능 낭비)→double-checked locking+volatile→enum(가장 안전) 암기 (4/10 → 재출제 필요)
3. `kafka/offset` — auto.offset.reset 동작 조건 암기("커밋된 오프셋 없을 때만 — 신규 CG / Retention 만료 2케이스"), commitSync vs commitAsync 선택 기준 (2/10 → 재출제 최우선)
4. `java/GC` — G1GC Region 분할 + "Garbage First = 가비지 많은 Region 우선 수집 → STW 예측 가능" 한 문장 암기 (7/10 → 재출제 필요)
5. `java/spring` — Gradle implementation vs api 전이 의존성 암기 (7/10 → 보완)
4. `clickhouse` — eventual consistency 표현 추가 연습 (9/10 → 마무리)
6. `kubernetes/HPA` — desiredReplicas 공식 정확한 표현(×100 없음), PDB minAvailable/maxUnavailable 설정 형태 암기 (8/10)
5. `mongodb` — $sum: 1(카운트) vs $sum: "$field"(합산) 구분, _id: "$category" 형태 (5/10 → 재복습 필요)
6. `java/spring-batch` — faultTolerant().skip/retry 코드 패턴 정착 (9/10)
7. `mysql` — 복합 인덱스 범위 이후 컬럼 인덱스 미사용 표현 정착 (9/10, 마지막 1점)
8. `elasticsearch` — Term Dictionary 이진탐색(O(log N)) + Posting List TF/position/offset 완성 ✅(10/10)
9. `postgresql` — xmin/xmax 내부 구조, XID Wraparound freeze 동작 상세
10. `distributed-systems` — 멱등키 구현 패턴(Idempotency-Key + Redis TTL) 코드 수준 암기

---

## 누적 세션 수

| 회사 | 세션 수 | 마지막 피드백 요약 |
|---|---|---|
| 화이트큐브 | 11 | 2026-04-28 세션 — K8s StatefulSet 5/10(volumeClaimTemplates 꼬리 모름), Istio Canary 7/10, PostgreSQL MVCC 6/10(XID Wraparound 미언급) |
| 넵튠 | 5 | 2026-04-14 세션(2회차) — Go channel fan-out·nil channel 비활성화 완성. Kafka rebalancing 4조건(max.poll.interval.ms 교정). 광고 파이프라인 ZooKeeper Watch + Redis pub/sub 비교 강점. 공통 보완: 코드 직접 제시, Redis 장애 fallback 패턴 추가 필요 |
| 인포뱅크 | 7 | 2026-04-28 세션 — @Transactional NESTED 7/10(영속성 컨텍스트 불일치 원인 추가 필요), WebSocket/STOMP 5/10(@SendTo 역할 오해), FastAPI Depends() 5/10(요청 스코프 격리 미언급), RabbitMQ DLQ 9/10 ✅ 드디어 해결 |
| 채널톡 | 4 | 2026-04-21 불합격. archived |
| 버즈니 | 3 | 2026-04-28 세션 — ES Analyzer 7/10, Kafka Exactly-Once 10/10 ✅, Redis pub/sub 9/10 |
| 인라이플 | 35 | 2026-05-06 5회차 — 트랜스코더 Event-Driven 9+1/10 ✅(ZooKeeper ephemeral node·Watch 재등록·disable node·Kafka 미선택 이유 완성), 동기/비동기 2/10(@Async ThreadPoolTaskExecutor·CompletableFuture 미암기), 운영 장애 대응 10+1/10 ✅(k6 재현·WebSocket 원인 특정·재발방지 완성) |
| wag | 1 | Java/Spring @Transactional, 동시성, JPA — 동시성 제어 최강점, 수치 정확도 보완 필요 (**지원 완료 → archived**) |

---

## Harness 신호 (개선 필요 항목)

> Claude가 답변을 못 하거나 잘못된 방향으로 갔을 때 여기에 기록.
> 다음 세션 전에 해당 topics 파일을 보강한다.

- **수치 정확도**: N+1 쿼리 수 "200→100"(오) → "1+N=101→1"(정), 인덱스 순서 `(createdAt, roomId)` → `(roomId, createdAt)`, `%w:%w` 이중 래핑은 비표준
- **gin API 구문**: `gin.H{http.NotFound,...}` → `c.JSON(http.StatusNotFound, gin.H{...})`
- **도메인 에러 패턴**: Service에서 `errors.Is` 후 래핑 아닌 교체(return ErrXxx)
- **레이어 분리**: Repository는 `fmt.Errorf("funcName: %w", err)` prefix 추가 권장
