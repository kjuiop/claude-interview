---
tags: [distributed-systems, cap-theorem, consistency, transaction, backend]
related: [kafka, redis, zookeeper, kubernetes, mysql, postgresql, system-design]
---

# Distributed Systems — 핵심 개념

→ [[home]] | 질문 모음: [[topics/distributed-systems/questions]]

---

## CAP 정리 (CAP Theorem)

분산 시스템은 아래 세 속성을 **동시에 모두 만족할 수 없고**, 최대 2개만 선택 가능하다.

| 속성 | 설명 |
|------|------|
| **C — Consistency** | 모든 노드가 동시에 같은 데이터를 봄. 쓰기 후 읽기 시 항상 최신값 반환 |
| **A — Availability** | 모든 요청에 응답 반환 (오류 없이). 단, 최신 데이터가 아닐 수 있음 |
| **P — Partition Tolerance** | 네트워크 단절이 발생해도 시스템이 계속 동작 |

### 왜 2개만 가능한가

네트워크 파티션은 실제로 반드시 발생 → **P는 포기 불가** → CP 또는 AP 선택

```
네트워크 단절 발생
  ├── 일관성 유지 (CP): 일부 노드 응답 거부 → Availability 희생
  └── 가용성 유지 (AP): stale 데이터 반환 → Consistency 희생
```

### CP vs AP 예시

| 분류 | 시스템 | 특징 |
|------|--------|------|
| **CP** | ZooKeeper, HBase, etcd | 파티션 시 쓰기 거부. 리더 선출 중 불가 |
| **AP** | Cassandra, DynamoDB, CouchDB | 파티션 시 계속 응답. eventual consistency |
| **CP** | Kafka | 리더 복제 완료 후 커밋 → 일관성 우선 |
| **AP** | Redis Cluster | 파티션 시 stale 데이터 반환 가능 |

### 이력서 연결

- **ZooKeeper 경험**: CP 시스템. 리더 선출 중 쓰기 불가 → 일관성 우선 선택
- **Redis**: AP에 가까움. Cluster 파티션 시 stale 데이터 가능 → 캐시 용도에 적합

> 참고: [[topics/distributed-systems/questions#cap-정리cap-theorem란-무엇인가요-왜-2개만-선택할-수-있나요]] | [[topics/zookeeper/concepts]]
