---
type: session-memory
updated: 2026-04-14
---

# Session Memory

> 세션 간 연속성을 위한 파일. `_harness/status.md`가 "준비 수준 스냅샷"이라면,
> 이 파일은 "마지막으로 어디서 멈췄는가"를 기록한다.
> 모든 세션 시작 시 CLAUDE.md 다음으로 읽는다.

---

## 마지막 세션 요약

- **날짜:** 2026-04-14
- **진행 내용:** 넵튠 대상 2회차. Go channel fan-out(nil channel 비활성화, select+default drop count), Kafka Consumer Group rebalancing 4조건(max.poll.interval.ms 교정), 광고 파이프라인 설계(Redis Hash+Lua, ZooKeeper Watch, Redis pub/sub 비교, Kafka 영속화) — 3개 질문 세션 완료
- **끝낸 지점:** /퇴근 실행. topics/ linter 완료 — zookeeper related 필드 kafka-rabbitmq 분리, postgresql/mysql questions 줄글 변환, kafka/concepts KRaft ZooKeeper wikilink 추가
- **다음 이어서 할 것:** kafka exactly-once transactional API 복습. Go GC Tricolor White/Gray/Black 재복습. kotlin/concepts.md 신규 작성. Redis 장애 fallback 패턴 정리

---

## 미결 결정사항

> 아직 결론 내지 못한 것들

| 항목 | 내용 | 기한 |
|---|---|---|
| - | - | - |

---

## 세션 간 인수인계 메모

> 긴 맥락이 필요한 것들 — 다음 세션 Claude가 반드시 알아야 할 것

- 화이트큐브·넵튠 공통: gin 구문 `c.JSON(http.StatusXxx, gin.H{...})` 반복 오류 주의. 코드 직접 제시 능력 보완 필요
- **GC Tricolor 최우선**: White=미탐색(수거 대상), Gray=자신확인+자식미확인, Black=자신+자식 모두 확인. 여전히 재복습 필요
- 넵튠 강점: ZooKeeper Watch 패턴 이력서 연결 강함. nil channel 동적 비활성화 패턴 완성
- 넵튠 보완: 모든 핵심 컴포넌트 장애 fallback 시나리오를 마지막에 한 줄 추가하는 습관 필요 (Redis 장애 누락 사례)
- Kafka commitAsync 재시도: 반드시 최신 offset 기준으로만 재시도. 이전 offset 재시도 금지
- ZooKeeper Watch 1회성: 이벤트 수신 후 반드시 `getData(path, watcher)` 재등록 — 면접 답변에 명시 필요
- topics/ linter 2026-04-14: zookeeper related 필드 kafka-rabbitmq→kafka/rabbitmq 분리, postgresql/mysql 줄글 변환 완료

---

## 이 파일 업데이트 규칙

- 세션 종료 시(`/퇴근` 실행 시) 자동으로 갱신
- "마지막 세션 요약"은 오늘 내용으로 덮어씀 (누적 아님)
- "미결 결정사항"은 해결될 때까지 유지
- "세션 간 인수인계 메모"는 중요한 맥락만 유지 (오래된 것은 삭제)
