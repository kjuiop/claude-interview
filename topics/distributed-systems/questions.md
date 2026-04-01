---
tags: [distributed-systems, cap-theorem, consistency, interview-questions]
related: [kafka, redis, zookeeper, kubernetes, mysql, postgresql, system-design]
---

# Distributed Systems — 면접 질문

→ [[home]] | 개념 정리: [[topics/distributed-systems/concepts]]

---

## CAP 정리(CAP Theorem)란 무엇인가요? 왜 2개만 선택할 수 있나요?

**난이도**: 기초

**핵심 키워드**: Consistency, Availability, Partition Tolerance, CP, AP, ZooKeeper, Cassandra

**모범 답변 방향**:
- **C (Consistency)**: 모든 노드가 동시에 같은 데이터를 봄. 쓰기 후 읽기 시 항상 최신값 반환
- **A (Availability)**: 모든 요청에 응답 반환 (오류 없이). 단, 최신 데이터가 아닐 수 있음
- **P (Partition Tolerance)**: 네트워크 단절이 발생해도 시스템이 계속 동작
- **왜 2개만**: 네트워크 파티션은 실제로 반드시 발생 → P는 포기 불가 → CP 또는 AP만 선택
- **CP 예시**: ZooKeeper, HBase — 파티션 시 일관성 유지, 일부 응답 거부
- **AP 예시**: Cassandra, DynamoDB — 파티션 시 계속 응답, stale 데이터 가능
- **이력서 연결**: ZooKeeper는 CP 시스템. 리더 선출 중 쓰기 불가 → 일관성 우선

**꼬리 질문 예시**:
- Redis는 CAP에서 어느 쪽인가요? (AP에 가까움 — 파티션 시 stale 데이터 가능)
- Kafka는 CAP 관점에서? (CP — 리더 복제 완료 후 커밋)

**면접 세션 피드백 (2026-04-01)**:
- 현황: 전혀 모르는 상태. CP/AP 선택 구조와 이력서의 ZooKeeper 연결 표현 우선 암기 필요
- 모범 답변: "분산 시스템에서 C·A·P를 동시에 보장할 수 없습니다. 네트워크 파티션은 피할 수 없으므로 P는 필수, 결국 C와 A 중 하나를 선택합니다. ZooKeeper는 CP 선택으로 리더 선출 중 쓰기를 거부해 일관성을 보장합니다."

---
