---
tags: [elasticsearch, search, inverted-index, backend]
related: [mysql, postgresql, kafka]
---

# ElasticSearch — 핵심 개념

→ [[home]] | 질문 모음: [[topics/elasticsearch/questions]]

---

## 역인덱스(Inverted Index)

RDBMS는 "문서 → 단어" 방향으로 저장하지만, ES는 반대로 **"단어 → 문서 ID 목록"** 방향으로 저장한다.

### 구조 예시

문서들:
- doc1: "스마트폰 케이스 추천"
- doc2: "케이스 할인"
- doc3: "스마트폰 신제품"

역인덱스:

| Term | Posting List |
|---|---|
| 스마트폰 | [doc1, doc3] |
| 케이스 | [doc1, doc2] |
| 추천 | [doc1] |
| 할인 | [doc2] |
| 신제품 | [doc3] |

### 검색 흐름
1. "케이스" 검색 → Term Dictionary에서 "케이스" 조회 → Posting List [doc1, doc2] 즉시 반환
2. LIKE '%케이스%'는 전체 행을 순회(Full Scan)하지만, 역인덱스는 Term Dictionary 조회라 O(log N)에 가까움

---

## Analyzer — 형태소 분리의 핵심

문서를 저장할 때 Analyzer가 텍스트를 형태소 단위로 분리해서 역인덱스에 등록한다.

```
입력: "스마트폰 케이스 추천합니다"
       ↓ Analyzer (한국어: nori, 영어: standard)
토큰: ["스마트폰", "케이스", "추천", "하다"]
       ↓
역인덱스에 각 토큰 등록
```

- **한국어**: nori analyzer (형태소 분리)
- **영어**: standard analyzer (소문자화, 불용어 제거)
- 오타 허용: fuzzy query (`fuzziness: AUTO`)

---

## RDBMS vs ElasticSearch 비교

| 항목 | RDBMS | ElasticSearch |
|---|---|---|
| 검색 방식 | B-Tree 인덱스 또는 Full Scan | 역인덱스 Term Dictionary 조회 |
| Full-Text 검색 | LIKE `%keyword%` → Full Scan | 형태소 분리 후 역인덱스 검색 |
| 인덱스 병용 | Full-Text + 다른 인덱스 동시 사용 어려움 | 기본으로 모든 필드 색인 가능 |
| 오타 허용 | 불가 | fuzzy query로 가능 |
| 확장성 | 수직 확장 | 샤드 수평 확장 |
| 정합성 | ACID 트랜잭션 | Near Real-Time (NRT), 최종 일관성 |

---

## ElasticSearch 선택 기준

**ES를 선택해야 하는 경우:**
- 사용자 검색창 (상품, 콘텐츠, 게시글 — 형태소/오타 허용 검색)
- 실시간 자동완성 (prefix 검색)
- 로그 분석 (ELK Stack: Elasticsearch + Logstash + Kibana)
- 다국어 형태소 검색

**RDBMS가 적합한 경우:**
- 정형 데이터 + 정확한 조건 필터 (WHERE id = 123)
- ACID 트랜잭션이 필요한 데이터 (주문, 결제)
- 관계형 데이터 JOIN이 필요한 경우

---

## 성능 관련 주의사항

ES가 느려지는 원인은 **건수**가 아니라:
- 샤드 설계 미스 (샤드 수 과다/부족)
- 집계(aggregation) 쿼리 남용
- 인덱스 매핑 최적화 부재 (dynamic mapping 문제)
- 힙 메모리 부족

ES는 수억 건 처리를 위해 설계된 시스템 — 1만 건에서 느려진다는 것은 잘못된 이해다.
