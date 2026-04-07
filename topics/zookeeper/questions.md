---
tags: [zookeeper, distributed-systems, interview-questions]
related: [distributed-systems, golang, kafka-rabbitmq]
---

# ZooKeeper — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/zookeeper/concepts]]

---

**Q. Polling 방식과 Event-Driven 방식의 차이를 실제 경험을 바탕으로 설명해주세요.**
- 모범 답변 구조: 비즈니스 배경 → Polling 문제(불필요한 비용) → ZooKeeper Watch 기반 해결 → ephemeral 선택 이유 → 수치
- 핵심 수치: "분당 70회 이상 발생하던 헬스체크 요청이 제거되었습니다"
- **답변 첫 문장**: 수치 또는 결과부터 시작할 것

**꼬리 질문: ephemeral node와 persistent node의 차이와 선택 이유는?**
- ephemeral: 클라이언트 세션 끊기면 자동 삭제 → crash 감지 자동화
- persistent: 명시적 삭제 전까지 유지 → 영구 설정, disable 목록에 활용
- 선택 이유: "서버 crash 시 별도 감지 로직 없이 Watch 이벤트로 자동 처리되기 때문"

**꼬리 질문: 서버가 비정상 종료(crash)됐을 때 어떻게 처리되나요?**
- ephemeral node → 세션 타임아웃 후 자동 삭제 → Watch 이벤트 발생
- 별도 crash 감지 로직 불필요 — ephemeral 선택 자체가 이미 해결책
- 참고: [[topics/zookeeper/concepts#2. ZNode (노드)]]

---

**면접 세션 피드백 (2026-04-01)**:
- 잘한 점: ephemeral/persistent 차이 + 트랜스코더 70대 관리 실사례 연결. polling vs Watch 효율성 비교가 핵심 강점
- 보완: Watch는 **1회성** — 이벤트 수신 후 재등록 필요. 작업 할당 메커니즘(children 목록 조회 → 가용 서버 분배) 한 줄 추가하면 완성도 향상
- 마무리 표현: "70대 서버 polling은 70번의 주기적 네트워크 요청이지만, Watch는 변경 시에만 이벤트가 오므로 불필요한 부하가 없었습니다"

---

---

**Q. ZooKeeper Watch 이벤트와 Feature Flag — DB 무중단 마이그레이션 실무 경험**

**Watch 이벤트 동작 원리**:
- 특정 노드를 감시 등록 → 해당 노드에 변화 발생 시 콜백으로 이벤트 수신
- **Polling 방식 아님**: 변경이 발생할 때만 이벤트를 서버로 푸시 → 불필요한 네트워크 요청 없음
- **이벤트 타입**: NodeCreated, NodeDeleted, NodeDataChanged, NodeChildrenChanged
- **일회성(One-time)**: Watch는 한 번 발동 후 자동 해제 → 지속 감지하려면 이벤트 수신 후 `getData(path, watcher)` 재등록 필요

**Feature Flag 구현 방법**:
- 설정 저장 노드는 **Persistent 노드**가 맞음 → Ephemeral은 세션 끊기면 노드 자체가 삭제되어 모든 Watch 등록 서버에 NodeDeleted 이벤트 발생 → 예상치 못한 동작
- 데이터 변경 Watch: `setData()`로 노드 데이터 변경 시 `NodeDataChanged` 이벤트 트리거
- N대 서버에 동시에 전환 신호 전달 → 일관된 시점에 모든 인스턴스 전환

**면접 세션 피드백 (2026-04-07)**:
- 잘한 점: Watch polling이 아닌 callback 방식 정확. 실무 맥락(cut-over 동시 전환 필요성) 명확하게 설명. "데이터 변경 Watch"라는 구체적 메커니즘 보완 설명 좋음
- 보완: Watch 일회성(재등록 패턴) 미언급. `setData()` API 명 미사용. "Ephemeral 노드 타입 상관없다"는 부정확 — Feature Flag에는 Persistent가 적절

---

**Q. ZooKeeper로 Feature Flag를 어떻게 구현했나요?**
- ZooKeeper Watch로 설정 노드 변경 감지
- N대 서버에 동시에 전환 신호 전달 → 일관된 시점에 모든 인스턴스 전환
- RDS → Aurora 마이그레이션 시 DB 커넥션 전환에 활용 (다운타임 3분 → 2초)
- 참고: [[topics/zookeeper/concepts#Feature Flag (분산 설정 전환)]]
