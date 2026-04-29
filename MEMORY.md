---
type: session-memory
updated: 2026-04-29
---

# Session Memory

> 세션 간 연속성을 위한 파일. `_harness/status.md`가 "준비 수준 스냅샷"이라면,
> 이 파일은 "마지막으로 어디서 멈췄는가"를 기록한다.
> 모든 세션 시작 시 CLAUDE.md 다음으로 읽는다.

---

## 마지막 세션 요약

- **날짜:** 2026-04-29
- **진행 내용:** 개념 학습 세션 — Java Reflection·Dynamic Proxy·CGLIB·AOP 전체 개념 정리. 면접 질문 4개 추가(topics/java/). MySQL 지연 조인 1회차 세션(7/10). K8s StatefulSet/volumeClaimTemplates(8/10). AOP 면접 답변 800자 훈련(Saga 로깅 경험 연결).
- **끝낸 지점:** topics/java/concepts.md에 Reflection·JDK Proxy·CGLIB 섹션 추가. topics/java/questions.md에 질문 4개 추가. _harness/ 업데이트 완료.
- **다음 이어서 할 것:** ConcurrentHashMap 버킷=배열 슬롯 표현 재복습(6/10). MongoDB $group 문법 재복습(5/10). K8s HPA desiredReplicas 공식 암기(6/10). mysql 지연 조인 "커버링 인덱스" 표현 추가 연습.

---

## 미결 결정사항

> 아직 결론 내지 못한 것들

| 항목 | 내용 | 기한 |
|---|---|---|
| - | - | - |

---

## 세션 간 인수인계 메모

> 긴 맥락이 필요한 것들 — 다음 세션 Claude가 반드시 알아야 할 것

- **Java Reflection·Proxy 학습 완료 (2026-04-29)**: JDK Dynamic Proxy(InvocationHandler·인터페이스 필수) vs CGLIB(Enhancer·상속·final 불가). Spring Boot 2.x = 항상 CGLIB 기본. @Retention(RUNTIME) 없으면 런타임에 어노테이션 못 읽음. JPA 기본 생성자 = Hibernate가 Constructor.newInstance() 사용하기 때문.
- **AOP 면접 답변 패턴 정착**: Saga 로깅 경험(@Around + @annotation(SagaStep) + MDC + proceed() + finally) 연결 완료. Pointcut·Advice·Weaving 용어를 답변 안에 자연스럽게 녹이는 연습 계속 필요.
- **이력서 경험 연결 훈련**: 모든 기술 질문에서 카테노이드/샵라이브 경험 첫 문장 또는 마무리 연결 습관 부족 — 지속 훈련 필요
- **Istio Canary YAML 완전 암기 필요**: DestinationRule subset(labels) + VirtualService weight/headers 패턴
- **goroutine close(ch) 패턴**: "특정값 보내 nil 변경"은 오답. sender가 `defer close(ch)` 호출 → for range receiver 자동 종료
- **GC Tricolor 미완**: White=미탐색(수거 대상), Gray=자신확인+자식미확인, Black=자신+자식 모두 확인. 재복습 필요
- ZooKeeper Watch 1회성: 이벤트 수신 후 반드시 `getData(path, watcher)` 재등록
- topics/ linter 2026-04-29: 전체 파일 YAML·빈파일 이상 없음. java/ Reflection·Proxy 4개 질문 추가 완료

---

## 이 파일 업데이트 규칙

- 세션 종료 시(`/퇴근` 실행 시) 자동으로 갱신
- "마지막 세션 요약"은 오늘 내용으로 덮어씀 (누적 아님)
- "미결 결정사항"은 해결될 때까지 유지
- "세션 간 인수인계 메모"는 중요한 맥락만 유지 (오래된 것은 삭제)
