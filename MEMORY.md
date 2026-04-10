---
type: session-memory
updated: 2026-04-10
---

# Session Memory

> 세션 간 연속성을 위한 파일. `_harness/status.md`가 "준비 수준 스냅샷"이라면,
> 이 파일은 "마지막으로 어디서 멈췄는가"를 기록한다.
> 모든 세션 시작 시 CLAUDE.md 다음으로 읽는다.

---

## 마지막 세션 요약

- **날짜:** 2026-04-10
- **진행 내용:** 화이트큐브+넵튠 대상 5회차 면접 세션. channel/redis/error-handling 복습, mysql/kafka/redis/Istio/TLS/광고파이프라인/Rate Limiting 신규 출제
- **끝낸 지점:** Q7(API Rate Limiting) 완료로 오늘 질문 10개 전부 소화. /퇴근 linter 실행 중.
- **다음 이어서 할 것:** Istio Ambient Mode 심화, Forward Secrecy ECDHE 원리, distributed-systems CAP 정리

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
- 넵튠: Kafka 이벤트 파이프라인, 광고 플랫폼 설계 중심. Redis HINCRBY로 CTR 집계 패턴 강점
- Istio 오늘 처음 배움 — Envoy sidecar 주입, mTLS STRICT/PERMISSIVE, VirtualService Canary 배포 정리 완료
- 광고 파이프라인 설계 강점: Event Collector → Kafka(linger.ms+batch.size) → Redis HINCRBY → ClickHouse
- Rate Limiting 취약점: Token Bucket refill rate 개념, Sorted Set score=timestamp 구조 추가 암기 필요
- topics/ 모범 답변 3분 형태 변환 진행 중. golang/channel, goroutine, map, error-handling, kafka, redis, kotlin questions 는 /퇴근 agent 처리 중

---

## 이 파일 업데이트 규칙

- 세션 종료 시(`/퇴근` 실행 시) 자동으로 갱신
- "마지막 세션 요약"은 오늘 내용으로 덮어씀 (누적 아님)
- "미결 결정사항"은 해결될 때까지 유지
- "세션 간 인수인계 메모"는 중요한 맥락만 유지 (오래된 것은 삭제)
