---
type: harness-status
updated: 2026-05-02
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
| kafka | ★★★★☆ | 2026-05-02 | ISR 복습 9/10 ✅ — 리더 포함 오개념 교정, NotEnoughReplicasException 암기 완료. replica.lag.time.max.ms 한 줄 추가 필요(마지막 1점). Rebalancing 7/10 — LeaveGroup 키워드 미사용. |
| rabbitmq | ★★★★☆ | 2026-04-28 | DLQ 3조건(TTL/NACK+requeue=false/x-max-length) + x-dead-letter-exchange 속성명 + x-death 헤더(count/reason/queue/exchange) 4회 연속 블로킹 드디어 해결 ✅ |
| clickhouse | ★★★★☆ | 2026-05-02 | 7/10. ORDER BY 블록 스캔 + PARTITION BY 프루닝/DROP 구분 완성. sparse index(granule + primary.idx) 개념이 "Primary Key 역할" 설명 핵심 — 아직 미언급. 선두 컬럼 설계 기준 추가 필요. |
| java/spring-batch | ★★★★☆ | 2026-05-02 | faultTolerant skip/retry 코드 패턴 9/10 ✅. skip=영구오류/retry=일시오류 케이스 구분 완성. "재처리 범위가 커진다" 표현 추가 연습 필요. |
| kotlin | ★★★★☆ | 2026-04-02 | IO 스레드 수 수치 교정(max(64,cores)), Unconfined 동작 보완, "이벤트 루프" 표현 지양 |
| kubernetes | ★★★★☆ | 2026-05-02 | 배포 전략 복습 9/10 ✅ — ArgoCD GitOps(Git SSOT+diff+sync) 교정 완료. readiness probe failureThreshold + periodSeconds 2개 추가 명시 필요(마지막 1점). HPA 6/10 — stabilizationWindowSeconds vs PDB 혼동. |
| zookeeper | ★★★★☆ | 2026-04-01 | ephemeral/Watch 이력서 연결 강점. Watch 1회성 특성 추가 필요 |
| distributed-systems | ★★★☆☆ | 2026-04-12 | 2PC 전혀 모름 — Phase 1/2 흐름, Blocking 원인, 3PC 차이 암기 필요. Saga 선택 이유 방향은 알고 있음 |
| elasticsearch | ★★★★☆ | 2026-04-28 | Analyzer 3단계 7/10 해결. Term Dictionary 이진탐색(O(log N)) + Posting List TF/position/offset 추가 암기 필요 |
| postgresql | ★★★☆☆ | 2026-04-28 | MVCC/Dead Tuple/VACUUM 6/10. XID Wraparound(32비트 트랜잭션 ID 한계, 쓰기 전면 차단) 개념 인지. xmin/xmax 내부 구조 추가 학습 필요 |
| java/spring | ★★★★★ | 2026-04-29 | AOP/프록시/Reflection 개념 완전 정리. JDK Dynamic Proxy vs CGLIB 메커니즘, Reflection 동작 원리(Class 객체·힙), @Retention RUNTIME, JPA 기본 생성자 이유까지 연결 완료 |

---

## 다음 우선순위

1. `clickhouse` — MergeTree ORDER BY = Primary Key 역할, PARTITION BY 파티션 프루닝, DROP PARTITION 세 가지 암기 필수 (1/10 → 면접 당일 최우선)
2. `kafka/ISR` — ISR 정의(replica.lag.time.max.ms 기준 집합), min.insync.replicas 미달 → NotEnoughReplicasException(쓰기 차단) 암기 (5/10)
3. `java/spring-batch` — faultTolerant().skip/retry 코드 패턴 암기. chunk 트레이드오프 커밋횟수/재처리범위 표현 정착 (7/10)
4. `java/ConcurrentHashMap` — 버킷=배열 슬롯 표현 정착, $sum: 1 카운트 구분 (복습 6/10, 버킷 오류 있음)
5. `mongodb` — $sum: 1(카운트) vs $sum: "$field"(합산) 구분, _id: "$category" 형태 (5/10 → 재복습 필요)
6. `kubernetes/HPA` — stabilizationWindowSeconds "가장 높은 값 선택" 표현 정착 + desiredReplicas 공식 암기 (6/10)
7. `mysql` — 복합 인덱스 B-Tree 원리 한 문장(선두 범위 시 이후 컬럼 인덱스 미사용), 지연 조인 "커버링 인덱스로 처리" 표현 추가
6. `java/STOMP` — 이력서(카테노이드 채팅 서버) 연결 연습. 내용은 9/10 해결 ✅
7. `elasticsearch` — Term Dictionary 이진탐색(O(log N)) + Posting List TF/position/offset 상세 암기
8. `postgresql` — xmin/xmax 내부 구조, XID Wraparound freeze 동작 상세
9. `ai-stt/Hallucination` — Temperature·Chain-of-Thought·Re-ranking 3가지 미언급. 이력서(STT→RAG) 연결 훈련 필요
10. `java/Reflection+Proxy` — 오늘 학습 완료. AOP 답변에서 Weaving·Pointcut 용어 자연스럽게 연결 연습 필요
11. `distributed-systems` — 멱등키 구현 패턴(Idempotency-Key + Redis TTL) 코드 수준 암기

---

## 누적 세션 수

| 회사 | 세션 수 | 마지막 피드백 요약 |
|---|---|---|
| 화이트큐브 | 11 | 2026-04-28 세션 — K8s StatefulSet 5/10(volumeClaimTemplates 꼬리 모름), Istio Canary 7/10, PostgreSQL MVCC 6/10(XID Wraparound 미언급) |
| 넵튠 | 5 | 2026-04-14 세션(2회차) — Go channel fan-out·nil channel 비활성화 완성. Kafka rebalancing 4조건(max.poll.interval.ms 교정). 광고 파이프라인 ZooKeeper Watch + Redis pub/sub 비교 강점. 공통 보완: 코드 직접 제시, Redis 장애 fallback 패턴 추가 필요 |
| 인포뱅크 | 7 | 2026-04-28 세션 — @Transactional NESTED 7/10(영속성 컨텍스트 불일치 원인 추가 필요), WebSocket/STOMP 5/10(@SendTo 역할 오해), FastAPI Depends() 5/10(요청 스코프 격리 미언급), RabbitMQ DLQ 9/10 ✅ 드디어 해결 |
| 채널톡 | 4 | 2026-04-21 불합격. archived |
| 버즈니 | 3 | 2026-04-28 세션 — ES Analyzer 7/10, Kafka Exactly-Once 10/10 ✅, Redis pub/sub 9/10 |
| 인라이플 | 15 | 2026-05-02 11회차 — Spring Batch faultTolerant 9/10 ✅, DB 무중단 마이그레이션 10/10 ✅, Redis Sorted Set 9/10, K8s ArgoCD GitOps 7/10(선택 이유+readiness 파라미터 보완), Kafka Rebalancing 7/10(LeaveGroup 키워드), ClickHouse ORDER BY sparse index 7/10 |
| wag | 1 | Java/Spring @Transactional, 동시성, JPA — 동시성 제어 최강점, 수치 정확도 보완 필요 (**지원 완료 → archived**) |

---

## Harness 신호 (개선 필요 항목)

> Claude가 답변을 못 하거나 잘못된 방향으로 갔을 때 여기에 기록.
> 다음 세션 전에 해당 topics 파일을 보강한다.

- **수치 정확도**: N+1 쿼리 수 "200→100"(오) → "1+N=101→1"(정), 인덱스 순서 `(createdAt, roomId)` → `(roomId, createdAt)`, `%w:%w` 이중 래핑은 비표준
- **gin API 구문**: `gin.H{http.NotFound,...}` → `c.JSON(http.StatusNotFound, gin.H{...})`
- **도메인 에러 패턴**: Service에서 `errors.Is` 후 래핑 아닌 교체(return ErrXxx)
- **레이어 분리**: Repository는 `fmt.Errorf("funcName: %w", err)` prefix 추가 권장
