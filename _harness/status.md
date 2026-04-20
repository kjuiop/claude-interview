---
type: harness-status
updated: 2026-04-20
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
| mongodb | ★★★★☆ | 2026-04-02 | Aggregation Pipeline 구조적 추론 가능. `$group` 문법(`$sum: 1`) + `$match` 앞 배치 이유 보완 필요 |
| python-fastapi | ★★★☆☆ | 2026-04-02 | DI 개념 이해 있음. Depends() yield 패턴(setup/teardown), 요청 스코프 vs @Autowired 싱글톤 차이 암기 필요 |
| java/jpa | ★★★★☆ | 2026-04-01 | fetch join+pagination @BatchSize 해결 정착, @EntityGraph 선언 위치 교정 |
| networking | ★★★★☆ | 2026-04-10 | TLS 1.3 흐름·ECDHE PFS 정착. Forward Secrecy 꼬리 질문 "모르겠습니다" → topics 보강 완료 |
| mysql | ★★★★☆ | 2026-04-10 | 복합 인덱스 순서·커버링 인덱스 정확. 인덱스 무력화 원인 정밀화 필요 — 함수 적용 시 범위 조건 변환(`>= AND <`), LIKE 앞 와일드카드 |
| redis | ★★★★☆ | 2026-04-10 | Hash vs String 선택 기준 이해. ziplist/listpack 인코딩 임계값(128/64) + Redis 7.4 HEXPIRE 버전 정확도 보완 필요 |
| kafka | ★★★★☆ | 2026-04-12 | acks/idempotence 전체 흐름 파악. PID 재시작 주체 오류(브로커→Producer 교정). acks=1 트레이드오프 표현 보완 필요 |
| rabbitmq | ★★★☆☆ | 2026-04-17 | Exchange 타입 정확. DLQ `x-dead-letter-exchange` 속성명 + `NACK+requeue=false` + `x-death` 헤더명 — 3회 연속 꼬리 질문 막힘. 코드 레벨 선언 암기 최우선 |
| kotlin | ★★★★☆ | 2026-04-02 | IO 스레드 수 수치 교정(max(64,cores)), Unconfined 동작 보완, "이벤트 루프" 표현 지양 |
| kubernetes | ★★★☆☆ | 2026-04-20 | StatefulSet 전혀 모름 — Pod identity, PVC volumeClaimTemplates, Kafka/MySQL StatefulSet 이유 암기 필요. Istio VirtualService/DestinationRule Canary 패턴 완전 미지 — YAML 암기 최우선 |
| zookeeper | ★★★★☆ | 2026-04-01 | ephemeral/Watch 이력서 연결 강점. Watch 1회성 특성 추가 필요 |
| distributed-systems | ★★★☆☆ | 2026-04-12 | 2PC 전혀 모름 — Phase 1/2 흐름, Blocking 원인, 3PC 차이 암기 필요. Saga 선택 이유 방향은 알고 있음 |
| elasticsearch | ★★★☆☆ | 2026-04-20 | Analyzer 3단계명(Character Filter/Tokenizer/Token Filter) 2회 연속 미언급 — 최우선 암기. Term Dictionary FST 구조, Posting List TF/position/offset 추가 암기 필요 |
| postgresql | ★★★☆☆ | 2026-04-02 | Dead Tuple/VACUUM 전혀 몰랐음. XID Wraparound 신규 암기 최우선 |
| java/spring | ★★★★★ | 2026-04-02 | AOP 3문제 복습 완료. 횡단 관심사·JoinPoint/Pointcut·self-invocation 모두 교정됨 |

---

## 다음 우선순위

1. `rabbitmq` — `x-dead-letter-exchange` 큐 선언 코드(`QueueBuilder.withArgument`), `NACK+requeue=false`, `x-death` 헤더명 암기 (3회 연속 막힘 — 최우선)
2. `kubernetes/Istio` — VirtualService/DestinationRule Canary YAML 패턴 (subset+weight+headers 조건) — 완전 미지 영역 (0/10)
3. `elasticsearch` — Analyzer 3단계(Character Filter → Tokenizer → Token Filter) 2회 연속 미언급. Term Dictionary FST·Posting List TF/position — 최우선 암기
4. `ai-stt/Hallucination` — Temperature·Chain-of-Thought·Re-ranking 3가지 미언급. 이력서(STT→RAG) 연결 훈련 필요
5. `mysql/oracle` — Oracle vs MySQL 4가지 차이 암기: 페이징(ROWNUM vs LIMIT), NULL(`''`=NULL vs 구분), 시퀀스 vs AUTO_INCREMENT, 기본 격리수준(READ COMMITTED vs REPEATABLE READ)
6. `java/@Transactional` — NESTED JPA 미지원 이유, REQUIRES_NEW 2커넥션 트레이드오프, savepoint 표현 정확화
7. `kafka` — exactly-once transactional API 전체 흐름 복습
8. `golang/memory` — GC Tricolor White/Gray/Black 정의 재복습
9. `distributed-systems` — 멱등키 구현 패턴(Idempotency-Key + Redis TTL) 코드 수준 암기

---

## 누적 세션 수

| 회사 | 세션 수 | 마지막 피드백 요약 |
|---|---|---|
| 화이트큐브 | 8 | 2026-04-12 세션(총 6회차) — Go Map/sync.Map/Hexagonal/Redis AOF 전부 교정 완료. K8s StatefulSet·2PC 신규 교정. GC Tricolor 색상 정의 여전히 불완전 — 내일 최우선 복습 |
| 넵튠 | 5 | 2026-04-14 세션(2회차) — Go channel fan-out·nil channel 비활성화 완성. Kafka rebalancing 4조건(max.poll.interval.ms 교정). 광고 파이프라인 ZooKeeper Watch + Redis pub/sub 비교 강점. 공통 보완: 코드 직접 제시, Redis 장애 fallback 패턴 추가 필요 |
| 인포뱅크 | 4 | 2026-04-17 세션(4회차) — Spring IoC/DI 1/10(@PostConstruct/@PreDestroy만 알고 IoC 역할·@Component vs @Bean·Singleton 스코프 완전 모름). 3회차: AI 배속 CER 7/10, Oracle vs MySQL 1/10(완전 모름), JPA fetch join+Pageable 7/10 |
| 채널톡 | 4 | 2026-04-17 세션(2회차 채널톡 질문 포함) — Webhook CRM 설계 9/10(Kafka 멱등성 3종 세트 완벽). 다음: 대표 경험 4가지(MultiCDN/S3파이프라인/ZooKeeper/CMAF) 30초 즉답 암기, 126배 수치 선제 연결 훈련 |
| wag | 1 | Java/Spring @Transactional, 동시성, JPA — 동시성 제어 최강점, 수치 정확도 보완 필요 (**지원 완료 → archived**) |

---

## Harness 신호 (개선 필요 항목)

> Claude가 답변을 못 하거나 잘못된 방향으로 갔을 때 여기에 기록.
> 다음 세션 전에 해당 topics 파일을 보강한다.

- **수치 정확도**: N+1 쿼리 수 "200→100"(오) → "1+N=101→1"(정), 인덱스 순서 `(createdAt, roomId)` → `(roomId, createdAt)`, `%w:%w` 이중 래핑은 비표준
- **gin API 구문**: `gin.H{http.NotFound,...}` → `c.JSON(http.StatusNotFound, gin.H{...})`
- **도메인 에러 패턴**: Service에서 `errors.Is` 후 래핑 아닌 교체(return ErrXxx)
- **레이어 분리**: Repository는 `fmt.Errorf("funcName: %w", err)` prefix 추가 권장
