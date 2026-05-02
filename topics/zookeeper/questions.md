---
tags: [zookeeper, distributed-systems, interview-questions]
related: [distributed-systems, golang, kafka, rabbitmq]
---

# ZooKeeper — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/zookeeper/concepts]]

---

**Q. Polling 방식과 Event-Driven 방식의 차이를 실제 경험을 바탕으로 설명해주세요.**

**모범 답변 (900자):**

카테노이드에서 트랜스코더 서버 70대의 상태를 관리할 때 ZooKeeper Watch 기반 Event-Driven 방식을 선택했습니다. 기존에는 마스터 서버가 트랜스코더 70대를 주기적으로 HTTP 폴링해서 생존 여부를 확인하는 방식이었는데, 서버 수가 늘어날수록 초당 수십 건의 불필요한 헬스체크 요청이 지속적으로 발생했습니다. 아무 작업도 없는 유휴 구간에도 동일한 빈도로 폴링이 일어나기 때문에 분당 70회 이상의 네트워크 요청이 낭비되고 있었습니다.

ZooKeeper Watch 방식으로 전환한 뒤 이 낭비가 없어졌습니다. 각 트랜스코더 서버가 기동될 때 ZooKeeper에 ephemeral 노드를 생성하고, 마스터는 해당 경로에 Watch를 등록합니다. 이후로는 트랜스코더 서버에 장애가 발생하거나 정상 종료되면 클라이언트 세션이 끊기면서 ephemeral 노드가 자동으로 삭제되고, 이 변경 이벤트가 Watch 콜백으로 마스터에게 전달됩니다. 마스터는 별도 감지 로직 없이 이벤트 수신만으로 서버 상태를 실시간으로 파악할 수 있습니다.

ephemeral 노드를 선택한 이유는 서버 crash 시 별도 삭제 로직 없이 자동으로 ZooKeeper가 노드를 정리해주기 때문입니다. persistent 노드였다면 서버가 죽어도 노드가 남아 있어 유효하지 않은 서버가 계속 활성 상태로 표시되는 문제가 생깁니다. 한 가지 주의할 점은 Watch가 1회성이라는 것입니다. 이벤트를 수신한 이후에는 자동으로 해제되기 때문에, 콜백 핸들러 안에서 `getData(path, watcher)` 또는 `getChildren(path, watcher)`를 다시 호출해 Watch를 재등록해야 지속적인 감시가 가능합니다.

**핵심 수치**: "분당 70회 이상 발생하던 헬스체크 요청이 제거되었습니다"
**답변 첫 문장**: 수치 또는 결과부터 시작할 것

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

---

**Q. DB 무중단 마이그레이션에서 Feature Flag + ZooKeeper 조합의 역할을 설명해주세요.**

**핵심 구조**:
1. **Feature Flag**: 애플리케이션 내부에서 DB 커넥션 전환 여부를 나타내는 상태 플래그. `getConnection()` 래핑 함수에서 플래그 확인 → 전환 상태이면 신규 커넥션 반환.
2. **ZooKeeper 역할 두 가지**:
   - **상태 저장**: 현재 어떤 DB를 바라봐야 하는지 Persistent 노드에 저장 → 신규 인스턴스가 떠도 올바른 커넥션으로 시작 보장
   - **동시 트리거**: Watch 이벤트로 모든 인스턴스에 동시에 전환 신호 전달

**전환 절차**:
1. Source DB → read-only 변경 (신규 쓰기 차단)
2. ZooKeeper 노드 업데이트 → Watch 이벤트 발생
3. 모든 애플리케이션 동시에 Feature Flag 전환 → 신규 DB 커넥션 사용
4. 다운타임: 약 2초 (커넥션 교체 자체는 1초 미만, 네트워크 전파 포함)

**Spring Boot 1.5 + 사내 ORM 제약**:
- Spring Cloud RefreshScope 사용 불가 → getConnection() 직접 래핑 방식 선택

**답변 첫 문장**: "다운타임을 3분에서 2초로 줄였습니다. 방법은 ~"으로 시작 (결과 수치 먼저)

**면접 세션 피드백 (2026-05-02 1회차)**:
- 잘한 점: Feature Flag와 ZooKeeper 역할 완벽 구분, 신규 인스턴스 전환 보장 메커니즘까지 설명, 기술 제약 배경 명확
- 보완: 답변 첫 문장에 결과 수치(3분→2초) 배치 — 면접관이 먼저 성과를 인지한 채 설명을 듣도록
