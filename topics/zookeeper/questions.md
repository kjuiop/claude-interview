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

**Q. ZooKeeper로 Feature Flag를 어떻게 구현했나요?**
- ZooKeeper Watch로 설정 노드 변경 감지
- N대 서버에 동시에 전환 신호 전달 → 일관된 시점에 모든 인스턴스 전환
- RDS → Aurora 마이그레이션 시 DB 커넥션 전환에 활용 (다운타임 3분 → 2초)
- 참고: [[topics/zookeeper/concepts#Feature Flag (분산 설정 전환)]]
