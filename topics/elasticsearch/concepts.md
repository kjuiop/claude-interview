---
tags: [elasticsearch, search, inverted-index, full-text-search, ilm, backend]
related: [mysql, postgresql, kafka]
---

# ElasticSearch — 핵심 개념

→ [[home]] | 질문 모음: [[topics/elasticsearch/questions]]

---

## 전문 검색(Full-Text Search) 동작 원리

### 색인(Indexing) 과정
문서를 저장할 때 Analyzer가 텍스트를 처리해 역인덱스에 등록한다.

```
입력 텍스트
    ↓
[Character Filter]  — HTML 태그 제거, 특수문자 치환 등
    ↓
[Tokenizer]         — 문자열을 토큰으로 분리 (공백, 형태소 단위)
    ↓
[Token Filter]      — 소문자화, 불용어 제거, 동의어 처리, 어간 추출
    ↓
역인덱스에 Term 등록
```

- Analyzer = Character Filter + Tokenizer + Token Filter 파이프라인
- **한국어**: `nori` analyzer — 형태소 기반 분리 (복합어 분해)
- **영어**: `standard` analyzer — 유니코드 기준 분리 + 소문자화

### 검색(Query) 과정
1. 쿼리 텍스트도 동일한 Analyzer로 분석 → 검색 토큰 생성
2. 각 토큰으로 Term Dictionary 조회 (이진 탐색, O(log N))
3. Posting List `(문서ID, 빈도, 위치)` 조회
4. **BM25 알고리즘**으로 관련성 점수 계산 → 내림차순 정렬

### BM25 관련성 점수
ES 5.0+부터 기본 유사도 알고리즘. TF-IDF를 개선한 확률적 랭킹 모델.

| 요소 | 의미 | 효과 |
|---|---|---|
| **TF (Term Frequency)** | 문서 내 단어 출현 빈도 | 많이 나올수록 점수 ↑ (포화 효과 적용) |
| **IDF (Inverse Document Frequency)** | 전체 문서 중 해당 단어의 희소성 | 흔한 단어(the, 그리고)는 점수 낮춤 |
| **문서 길이 정규화** | 평균 문서 길이 대비 현재 문서 길이 | 짧은 문서에서 키워드 밀도가 높으면 유리 |

**TF 포화(Saturation)**: 단어가 12번 나온다고 1번보다 12배 관련성 높지 않음 → BM25는 일정 횟수 이상 증가 효과를 급감시킴

> 출처: https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables

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

---

## Hot-Warm-Cold 아키텍처 & ILM (Index Lifecycle Management)

### 개념
시계열 데이터(로그, 메트릭)는 시간이 지날수록 접근 빈도가 감소한다. 이 패턴을 반영해 노드를 성능 등급별로 분리하는 비용 최적화 전략.

> "오늘 데이터 = 활발히 색인/검색, 이번 주 = 가끔 검색, 지난달 = 거의 안 봄"

### 각 단계(Phase) 특징

| Phase | 진입 조건 | 노드 타입 | 주요 작업 | 인덱스 우선순위 |
|---|---|---|---|---|
| **Hot** | 신규 인덱스 | 고성능 SSD, 높은 CPU | 색인(write) + 활발한 검색 | 50 (높음) |
| **Warm** | 7일 경과 | 중간 사양 | Shrink(샤드 축소), Forcemerge(세그먼트 병합) | 25 |
| **Cold** | 30일 경과 | 대용량 저가 HDD | Freeze(메모리 해제), 검색 드물게 | 0 |
| **Delete** | 60일 경과 | - | 인덱스 자동 삭제 | - |

### ILM 주요 Action

| Action | 설명 |
|---|---|
| **Rollover** | 크기(50GB) 또는 기간(30일) 기준으로 인덱스 새로 생성 |
| **Shrink** | Primary Shard 수 축소 → 리소스 절약 (Warm 단계) |
| **Forcemerge** | Lucene 세그먼트 병합 → 검색 성능 최적화, 디스크 절약 |
| **Allocate** | 특정 속성의 노드로 이동 (`node.attr.data=warm`) |
| **Freeze** | 인덱스를 읽기 전용으로 메모리 해제 → Cold 단계 메모리 절약 |
| **Delete** | 인덱스 삭제 |

### 노드 설정 방법
```bash
# 노드 속성 부여 (elasticsearch.yml 또는 시작 인수)
node.attr.data: hot    # Hot 노드
node.attr.data: warm   # Warm 노드
node.attr.data: cold   # Cold 노드

# ILM 정책에서 Allocate action으로 이동 대상 지정
"allocate": {
  "require": { "data": "warm" }
}
```

### Beats/Logstash 연동
```yaml
# Beats
output.elasticsearch:
  ilm.enabled: true

# Logstash
output {
  elasticsearch {
    ilm_enabled => true
  }
}
```

### 주의사항
- **정책 변경은 단계 전환 시에만 적용**: 현재 Hot 단계 인덱스는 Warm으로 전환될 때 새 정책 반영
- `GET /<index>/_ilm/explain` API로 현재 단계/상태 확인 가능
- Frozen 인덱스는 검색 시 on-demand로 메모리 로드 → 첫 쿼리 지연 발생

> 출처: https://www.elastic.co/kr/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management
