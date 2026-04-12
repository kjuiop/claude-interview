---
type: session-memory
updated: 2026-04-12
---

# Session Memory

> 세션 간 연속성을 위한 파일. `_harness/status.md`가 "준비 수준 스냅샷"이라면,
> 이 파일은 "마지막으로 어디서 멈췄는가"를 기록한다.
> 모든 세션 시작 시 CLAUDE.md 다음으로 읽는다.

---

## 마지막 세션 요약

- **날짜:** 2026-04-12
- **진행 내용:** 화이트큐브 대상 총 6회차(복습 3회 포함). Go Map/sync.Map, Redis AOF Rewrite, Hexagonal Architecture, Kafka acks/Idempotent, CQRS, Go GC Tricolor, K8s StatefulSet, 2PC/Saga, Kafka Exactly-Once, MongoDB Sharding — 10개 주제 소화
- **끝낸 지점:** Go GC Tricolor 복습 중단 후 /퇴근 실행. topics/ 모범 답변 6개 파일 줄글 변환 완료(kubernetes, distributed-systems, mongodb ×2, system-design ×2)
- **다음 이어서 할 것:** Go GC Tricolor White/Gray/Black 정의 재복습 (오늘 완전 교정 못 함). kotlin/concepts.md 신규 작성

---

## 미결 결정사항

> 아직 결론 내지 못한 것들

| 항목 | 내용 | 기한 |
|---|---|---|
| - | - | - |

---

## 세션 간 인수인계 메모

> 긴 맥락이 필요한 것들 — 다음 세션 Claude가 반드시 알아야 할 것

- 화이트큐브: Go 메인 + Kubernetes/Istio 필수. gin 구문 `c.JSON(http.StatusXxx, gin.H{...})` 반복 오류 주의
- **GC Tricolor 내일 최우선**: White=미탐색, Gray=자신확인+자식미확인, Black=자신+자식 모두 확인. 오늘 복습에서 White/Black 정의 여전히 부정확
- K8s StatefulSet 오늘 교정 완료: Pod identity(ordinal index), 고정 DNS, 순서 보장 배포/종료, volumeClaimTemplates PVC 독립 바인딩
- 2PC 오늘 교정 완료: Prepare(Lock 획득+응답)/Commit 2단계, Coordinator 장애→Blocking, 3PC로 개선, Saga(Choreography 보상트랜잭션)로 대체
- MongoDB Sharding 교정 완료: Cardinality/Frequency/Monotonically Increasing 3기준, ObjectId 앞4바이트=타임스탬프→단조증가→Hotspot
- CQRS 채팅 서버 적용 여부: MongoDB+Redis 구조에서 별도 CQRS 불필요 (Redis가 이미 읽기 모델 역할). 단, 결제 등 정합성 중요 데이터는 Command DB 직접 조회 원칙
- topics/ 모범 답변 줄글 변환 완료: kubernetes, distributed-systems, mongodb ×2, system-design ×2 (총 6개 섹션)

---

## 이 파일 업데이트 규칙

- 세션 종료 시(`/퇴근` 실행 시) 자동으로 갱신
- "마지막 세션 요약"은 오늘 내용으로 덮어씀 (누적 아님)
- "미결 결정사항"은 해결될 때까지 유지
- "세션 간 인수인계 메모"는 중요한 맥락만 유지 (오래된 것은 삭제)
