---
tags: [mongodb, nosql, database, interview]
related: [mysql, postgresql, redis]
---

# MongoDB — 면접 질문

→ [[home]] | [[topics/mongodb/concepts]]

---

## Sharding

### Q. MongoDB Range Sharding과 Hash Sharding의 차이, Shard Key 선택 기준, ObjectId Hotspot 이유를 설명해주세요.

**Range vs Hash 비교:**

| | Range Sharding | Hash Sharding |
|---|---|---|
| 분산 기준 | Shard Key 값 범위 | Shard Key 해시값 |
| 범위 조회 | 빠름 (같은 샤드에 모임) | 느림 (모든 샤드 스캔) |
| 쓰기 분산 | 단조증가 키 → Hotspot | 균등 분산 |
| 적합 케이스 | 범위 조회 많을 때 | 쓰기 분산 우선 |

**Shard Key 선택 3가지 기준:**

1. **Cardinality (카디널리티)**: 값의 다양성. 낮으면 샤드 수보다 고유값이 적어 특정 샤드 과부하. 높을수록 좋음.
2. **Frequency (빈도)**: 특정 값 집중도. 한 값이 대부분이면 그 샤드가 Hotspot. 균등할수록 좋음.
3. **Monotonically Increasing (단조증가)**: 값이 계속 커지는 패턴. Range Sharding에서 새 데이터가 항상 마지막 샤드에만 쌓임. 피하거나 Hash Sharding으로 완화.

**ObjectId Hotspot 이유:**
```
ObjectId 구조: [4바이트 타임스탬프][5바이트 랜덤][3바이트 카운터]
```
앞 4바이트가 타임스탬프 → 시간이 지날수록 값이 단조증가 → Range Sharding에서 새 도큐먼트가 항상 마지막 샤드에만 쌓임.

**광고 클릭 이벤트 좋은 Shard Key 설계:**
```js
// 나쁜 설계
{ _id: ObjectId }    // 단조증가 → Hotspot
{ createdAt: 1 }     // 단조증가 → Hotspot

// 좋은 설계
{ advertiserId: 1, createdAt: 1 }
// 광고주 수만큼 카디널리티 높음 + 같은 광고주 이벤트 집계 쿼리 효율적
```

**채팅 메시지 Shard Key 설계 (이력서 연결):**
```js
{ roomId: 1, createdAt: 1 }
// 채팅방별 분산 + 같은 방 메시지 시간순 조회 효율
```

**면접 세션 피드백 (2026-04-12 5회차)**:
- 처음 접한 주제 — Range/Hash 차이 + Shard Key 3기준 + ObjectId 구조 암기 필요
- 복합 Shard Key 패턴(advertiserId + createdAt)을 이력서 경험과 연결하는 연습 필요

---

## Document 모델 vs RDBMS

**Q. MongoDB Document 모델과 RDBMS의 차이를 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** 유연 스키마, BSON, 임베딩 vs 참조, 수평 확장, 조인 없음

**모범 답변:**

RDBMS와 MongoDB의 핵심 차이는 스키마 유연성과 데이터 관계 표현 방식에 있습니다. RDBMS는 테이블 스키마를 미리 정의하고, 컬럼을 추가하거나 변경할 때 ALTER TABLE이 필요합니다. 수백만 건 이상의 테이블에서 ALTER는 잠금(lock)을 수반하거나 온라인 DDL이 필요해 운영 중 적용이 까다롭습니다. 데이터 간 관계는 정규화를 통해 여러 테이블로 분산하고 JOIN으로 다시 합쳐서 표현합니다. 반면 MongoDB는 스키마를 강제하지 않아 Document마다 필드가 달라도 되고, 관련 데이터를 한 Document 안에 임베딩해 조인 없이 단일 쿼리로 읽을 수 있습니다.

카테노이드 채팅 서버에서 MongoDB를 선택한 이유가 바로 이 유연 스키마 특성 때문이었습니다. 채팅 메시지는 일반 텍스트, 공지, 차단 알림, 전체 메시지 등 유형마다 포함되는 필드 구조가 달랐습니다. RDBMS로 이를 표현하려면 공통 컬럼 외에 유형별로 nullable 컬럼을 추가하거나 별도 테이블을 분리해야 했지만, MongoDB에서는 각 Document가 유형에 맞는 필드만 포함하면 됐기 때문에 스키마 변경 없이 새 메시지 유형을 추가할 수 있었습니다.

기술 선택 기준으로는 세 가지를 봅니다. 첫째, 스키마 변경 빈도입니다. 피처 개발이 빠르게 반복되거나 도메인 구조가 자주 바뀌는 경우에는 유연 스키마가 유리합니다. 둘째, 조인 필요성입니다. 여러 엔티티 간 정합성이 중요하고 복잡한 JOIN이 많다면 RDBMS가 적합합니다. MongoDB의 `$lookup`은 JOIN과 유사하지만 샤딩 환경에서는 제약이 있고, 인덱스 활용이 제한됩니다. 셋째, 쓰기 패턴입니다. 단조증가 ID로 대량의 이벤트를 빠르게 인서트하는 패턴, 예를 들어 채팅 메시지나 로그 수집에는 MongoDB가 수평 확장이 용이해 적합합니다.

**실무 연결 포인트:** 채팅 메시지 메타데이터(공지·차단·전체 메시지)가 유형마다 다른 필드 구조 → 유연 스키마 필요 → MongoDB 선택

---

## Aggregation Pipeline

**Q. MongoDB의 Aggregation Pipeline을 설명하고 사용 예시를 들어주세요.**

**난이도:** 중급

**핵심 키워드:** $match, $group, $sort, $limit, $project, $lookup, 파이프라인 스테이지

**모범 답변:**

Aggregation Pipeline은 SQL의 GROUP BY, ORDER BY, HAVING과 같은 집계 연산을 MongoDB 방식으로 처리하는 기능입니다. 여러 스테이지를 체인처럼 연결해서 앞 스테이지의 출력이 다음 스테이지의 입력이 되는 구조입니다. 각 스테이지는 `$match`, `$group`, `$sort`, `$limit`, `$project`, `$lookup` 등으로 구성됩니다.

성능에서 가장 중요한 원칙은 `$match`를 파이프라인 앞쪽에 배치하는 것입니다. `$match`가 뒤에 오면 모든 Document를 불러온 뒤 필터링하지만, 앞에 두면 인덱스를 활용해 처리할 Document 수 자체를 줄일 수 있습니다. 예를 들어 채팅방별 메시지 수를 집계할 때 날짜 범위 조건을 `$match`로 먼저 거르면 `$group` 단계가 훨씬 적은 데이터를 처리하게 됩니다.

카테노이드에서 MongoDB로 채팅 메시지를 저장했기 때문에 이런 집계 쿼리를 Aggregation Pipeline으로 작성할 수 있었습니다. 직접 경험이 제한적이더라도, SQL 지식에서 `$match → WHERE`, `$group → GROUP BY`, `$sort → ORDER BY`, `$limit → LIMIT` 으로 대응 관계를 잡으면 처음 보는 파이프라인도 추론할 수 있습니다.

```js
// 채팅방별 메시지 수 TOP 10
db.messages.aggregate([
  { $match: { createdAt: { $gte: ISODate("2026-01-01") } } },
  { $group: { _id: "$roomId", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

**꼬리 질문 예시:**
- `$lookup`과 SQL JOIN의 차이는? 성능 측면에서 어떤 점을 주의해야 하나요?
- Aggregation Pipeline에서 인덱스가 활용되는 조건은?
- `$unwind`는 언제 사용하나요?

**면접 세션 피드백 (2026-04-02 3회차)**:
- 잘한 점: SQL 절 매핑 정확. 실무 파이프라인(`$match` 날짜범위 → `$group` roomId+sum → `$sort` -1 → `$limit`) 자연스럽게 구성. SQL 지식으로 추론하는 전략 유효.
- 보완:
  - `$group` 문법 암기: `{ _id: "$roomId", messageCount: { $sum: 1 } }`. 필드 앞 `$` 필수.
  - `$match` 앞 배치 이유 언급: 인덱스 활용으로 처리 문서 수 최소화 → 성능 핵심
  - "직접 경험 없지만 SQL 지식으로 추론" 선언 먼저 → 면접관 신뢰 확보

**면접 세션 피드백 (2026-04-29 1회차)**:
- 잘한 점: $match/group/sort/limit 역할 정확. $match 앞 배치 이유(데이터 줄여 이후 연산 비용 감소) 정확.
- 보완:
  - **$group 문법 미암기**: `{ $group: { _id: "$category", count: { $sum: 1 } } }`. 필드 참조 시 `$` 접두사 필수. `$sum: 1`이 row 하나당 1씩 카운트.
  - **$project 역할 미암기**: SQL SELECT 절과 동일. `{ $project: { name: 1, _id: 0 } }`처럼 필드 선택(1)/제외(0). 새 필드 계산도 가능.
- 점수: 5/10 ($project 모름, $group 문법 상세 미암기)

**면접 세션 피드백 (2026-04-29 5회차 복습)**:
- 5개 스테이지 SQL 매핑 정확, $project = SELECT ✅. $group 방향 맞음.
- 보완:
  - **$sum: 1 vs $sum: "$field" 구분**: 카운트 = `$sum: 1`, 필드 합산 = `$sum: "$amount"`. 혼동 주의.
  - **_id 키**: `_$id` 아님 → `_id`. 필드 참조: `_id: "$category"`
- 반드시 암기: `{ $group: { _id: "$category", count: { $sum: 1 }, avg: { $avg: "$price" } } }`
- 점수: 7/10

---

## 인덱스 전략

**Q. MongoDB에서 채팅 메시지 조회 성능을 위한 인덱스 전략을 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** 복합 인덱스, 선행 키, ESR Rule (Equality-Sort-Range)

**모범 답변 방향:**

채팅 메시지 조회의 주요 패턴은 "특정 채팅방의 최근 메시지를 시간 역순으로 가져오는 것"으로, 이 패턴에 최적화된 인덱스는 `(roomId, createdAt)` 복합 인덱스입니다. 인덱스 필드 순서 원칙은 ESR Rule로, Equality 조건이 선행 키로, Sort 기준이 그 다음, Range 조건이 마지막에 오는 것이 권장됩니다. roomId가 선행 키인 이유는 roomId는 정확한 같음(Equality) 조건으로 우선 필터링하고, createdAt은 그 결과 내에서 정렬 또는 범위 조회를 하기 때문입니다. 만약 `(createdAt, roomId)` 순서로 설정하면 createdAt 전체 범위를 먼저 스캔한 뒤 roomId로 필터링해야 하므로 훨씬 비효율적입니다. 실제로 `db.messages.createIndex({ roomId: 1, createdAt: -1 })`와 같이 설정하면 특정 방의 최신순 조회가 인덱스만으로 처리됩니다.

```js
db.messages.createIndex({ roomId: 1, createdAt: -1 })
```

**주의:** 인덱스 필드 순서는 쿼리 패턴에 따라 결정. "같음 조건(Equality) → 정렬(Sort) → 범위(Range)" 순서 권장.
