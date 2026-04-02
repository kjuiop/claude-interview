---
tags: [mongodb, nosql, database, interview]
related: [mysql, postgresql, redis]
---

# MongoDB — 면접 질문

→ [[home]] | [[topics/mongodb/concepts]]

---

## Document 모델 vs RDBMS

**Q. MongoDB Document 모델과 RDBMS의 차이를 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** 유연 스키마, BSON, 임베딩 vs 참조, 수평 확장, 조인 없음

**모범 답변 방향:**
- RDBMS: 고정 스키마(ALTER TABLE 비용), 정규화, JOIN으로 관계 표현
- MongoDB: 유연 스키마(필드 추가/삭제 자유), Document 임베딩으로 조인 최소화
- 선택 기준: 스키마 변경 빈도 + 조인 필요성 + 쓰기 패턴

**실무 연결 포인트:** 채팅 메시지 메타데이터(공지·차단·전체 메시지)가 유형마다 다른 필드 구조 → 유연 스키마 필요 → MongoDB 선택

---

## Aggregation Pipeline

**Q. MongoDB의 Aggregation Pipeline을 설명하고 사용 예시를 들어주세요.**

**난이도:** 중급

**핵심 키워드:** $match, $group, $sort, $limit, $project, $lookup, 파이프라인 스테이지

**모범 답변 방향:**
- SQL의 집계 쿼리와 동일한 역할을 스테이지 체인으로 처리
- `$match`를 앞에 두어 처리 문서 수 최소화 (성능 핵심)
- 직접 경험 없어도 SQL 대응 관계로 설명 가능

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

---

## 인덱스 전략

**Q. MongoDB에서 채팅 메시지 조회 성능을 위한 인덱스 전략을 설명해주세요.**

**난이도:** 중급

**핵심 키워드:** 복합 인덱스, 선행 키, ESR Rule (Equality-Sort-Range)

**모범 답변 방향:**
- "특정 채팅방의 시간 범위 조회" 패턴: `(roomId, createdAt)` 복합 인덱스
- roomId가 선행 키 — Equality 조건이 먼저, Range 조건이 뒤에 오는 것이 원칙
- `(createdAt, roomId)` 순서는 비효율: createdAt으로 전체 범위 스캔 후 roomId 필터링

```js
db.messages.createIndex({ roomId: 1, createdAt: -1 })
```

**주의:** 인덱스 필드 순서는 쿼리 패턴에 따라 결정. "같음 조건(Equality) → 정렬(Sort) → 범위(Range)" 순서 권장.
