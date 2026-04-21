---
type: session-memory
updated: 2026-04-20
---

# Session Memory

> 세션 간 연속성을 위한 파일. `_harness/status.md`가 "준비 수준 스냅샷"이라면,
> 이 파일은 "마지막으로 어디서 멈췄는가"를 기록한다.
> 모든 세션 시작 시 CLAUDE.md 다음으로 읽는다.

---

## 마지막 세션 요약

- **날짜:** 2026-04-21
- **진행 내용:** 버즈니/화이트큐브 대상 1회차 세션 — Q5(복합 인덱스+커버링 인덱스), Q7(트랜잭션 격리 수준), Q9(Istio Canary YAML). 세션 총점 14/30(47%).
- **끝낸 지점:** 1회차 세션 종료. daily 기록 및 topics/ 업데이트 완료. mysql/questions.md에 지연 조인 패턴 추가.
- **다음 이어서 할 것:** Istio VirtualService/DestinationRule YAML 손으로 쓰기 암기. 지연 조인 SQL 패턴 암기. OncePerRequestFilter + @Transactional 프록시 bypass 메커니즘 암기. 이력서 경험 첫 문장 연결 훈련(2회차 연속 전 문제 0점)

---

## 미결 결정사항

> 아직 결론 내지 못한 것들

| 항목 | 내용 | 기한 |
|---|---|---|
| - | - | - |

---

## 세션 간 인수인계 메모

> 긴 맥락이 필요한 것들 — 다음 세션 Claude가 반드시 알아야 할 것

- **이력서 경험 연결 훈련 최우선**: 모든 기술 질문에서 카테노이드/샵라이브 경험을 첫 문장 또는 마무리에 연결하는 습관 부족 — 2026-04-21 세션까지 연속 이력서 연결 0점(5회차 누적)
- **Istio Canary YAML 완전 암기 필요**: DestinationRule subset(labels) + VirtualService weight/headers 패턴. 2026-04-20 완전 미지(0/10)
- **goroutine close(ch) 패턴**: "특정값 보내 nil 변경"은 오답. sender가 `defer close(ch)` 호출 → for range receiver 자동 종료
- **GC Tricolor 미완**: White=미탐색(수거 대상), Gray=자신확인+자식미확인, Black=자신+자식 모두 확인. 재복습 필요
- ZooKeeper Watch 1회성: 이벤트 수신 후 반드시 `getData(path, watcher)` 재등록
- Kafka commitAsync 재시도: 반드시 최신 offset 기준으로만 재시도
- topics/ linter 2026-04-20: 전체 53개 파일 YAML·빈파일 이상 없음. ai-stt Hallucination·kubernetes Canary YAML·goroutine close(ch) 추가 완료

---

## 이 파일 업데이트 규칙

- 세션 종료 시(`/퇴근` 실행 시) 자동으로 갱신
- "마지막 세션 요약"은 오늘 내용으로 덮어씀 (누적 아님)
- "미결 결정사항"은 해결될 때까지 유지
- "세션 간 인수인계 메모"는 중요한 맥락만 유지 (오래된 것은 삭제)
