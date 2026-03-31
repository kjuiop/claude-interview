---
tags: [mongodb, nosql, database, document-model]
related: [mysql, postgresql, redis, kafka]
---

# MongoDB — 핵심 개념

→ [[home]] | [[topics/mongodb/questions]]

---

## Document 모델 vs RDBMS

| 항목 | RDBMS | MongoDB |
|---|---|---|
| 데이터 형식 | 행(Row) + 고정 스키마 | Document(JSON/BSON) + 유연 스키마 |
| 스키마 변경 | ALTER TABLE 비용 발생 | 필드 추가/삭제 자유 |
| 조인 | JOIN으로 연관 데이터 결합 | 임베딩(Embedding) 또는 `$lookup` |
| 트랜잭션 | 강력한 ACID 지원 | 4.0+부터 멀티 다큐먼트 트랜잭션 지원 |
| 확장 | 수직 확장 | 수평 확장(샤딩) 용이 |
| 적합 케이스 | 복잡한 관계, 정형 데이터 | 비정형/반정형 데이터, 유연한 스키마 |

**MongoDB 선택 기준:**
- 스키마가 자주 바뀌거나 필드가 케이스별로 다른 경우 (채팅 메시지 메타데이터 등)
- 다른 컬렉션과의 조인이 거의 없는 경우
- 쓰기 비중이 높고 특정 필드 기준 범위 조회가 주요 패턴인 경우

---

## 채팅 메시지 저장 사례

채팅 메시지의 메타데이터(공지, 차단, 전체 메시지 등)는 메시지 유형에 따라 필드 구조가 다름.
RDBMS로 구현 시 nullable 컬럼 다수 발생 + 스키마 변경 비용 큼 → MongoDB 적합.

**인덱스 전략:**
```js
// 복합 인덱스 — roomId를 선행 키로 설정
db.messages.createIndex({ roomId: 1, createdAt: -1 })
```
- "특정 채팅방의 시간 범위 조회" 패턴: roomId로 먼저 필터링 → createdAt으로 정렬
- 인덱스 순서 중요: `(roomId, createdAt)` vs `(createdAt, roomId)` — 전자가 훨씬 효율적

---

## Aggregation Pipeline

MongoDB의 데이터 집계 프레임워크. SQL의 GROUP BY + ORDER BY + LIMIT과 동일한 역할을 파이프라인 스테이지로 처리.

```js
// 채팅방별 메시지 수 집계 — TOP 10
db.messages.aggregate([
  { $match: { createdAt: { $gte: ISODate("2026-01-01") } } }, // WHERE
  { $group: { _id: "$roomId", count: { $sum: 1 } } },         // GROUP BY
  { $sort: { count: -1 } },                                   // ORDER BY
  { $limit: 10 }                                              // LIMIT
])
```

**주요 스테이지:**

| 스테이지 | SQL 대응 | 설명 |
|---|---|---|
| `$match` | WHERE | 필터링 |
| `$group` | GROUP BY | 집계 (`$sum`, `$avg`, `$max` 등) |
| `$sort` | ORDER BY | 정렬 |
| `$limit` | LIMIT | 결과 수 제한 |
| `$project` | SELECT | 필드 선택/변환 |
| `$lookup` | JOIN | 다른 컬렉션 조인 |
| `$unwind` | - | 배열 필드를 개별 문서로 펼치기 |

**성능 팁:** `$match`를 파이프라인 앞에 두어 처리할 문서 수를 줄인다. `$match`와 `$sort`가 인덱스를 활용할 수 있도록 인덱스 설계.

---

## Redis vs MongoDB 역할 분리

| | Redis | MongoDB |
|---|---|---|
| 영속성 | 휘발성(기본), RDB/AOF로 선택적 영속화 | 기본 영속 |
| 용도 | 캐시, 세션, 실시간 카운터, pub/sub | 영속 데이터 저장 |
| 채팅 시스템 적용 | 실시간 동시 시청자 수, 최근 메시지 캐시 | 채팅 메시지 영속 저장 |

**Write-Behind 패턴 (좋아요·시청자 수 집계):**
- 순간 트래픽이 몰리는 카운터: Redis에 먼저 저장 → Kafka → ClickHouse(실시간 통계) → BigQuery(장기 보관)
- 서비스 종료 후 최종 통계는 BigQuery에 저장
