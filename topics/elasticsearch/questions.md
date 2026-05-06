---
tags: [elasticsearch, search, inverted-index, interview-questions]
related: [mysql, postgresql]
---

# ElasticSearch — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/elasticsearch/concepts]]

---

## 역인덱스(Inverted Index) 구조와 RDBMS 비교

**난이도**: 기초

**핵심 키워드**: 역인덱스, Term Dictionary, Posting List, Analyzer, 형태소 분석, Full-Text Search

**모범 답변 방향**:

역인덱스는 "단어 → 문서 ID 목록" 방향으로 저장하는 자료구조입니다. RDBMS가 행(Row)에 데이터를 저장하는 것과 반대 방향으로, ElasticSearch는 문서를 저장할 때 Analyzer가 텍스트를 형태소 단위로 분리해 각 토큰을 Term Dictionary의 키로 등록하고, 해당 토큰이 어떤 문서 ID에 등장하는지를 Posting List로 관리합니다. 검색 시에는 쿼리 키워드를 Term Dictionary에서 이진 탐색(O(log N))으로 찾아 Posting List를 즉시 반환합니다. RDBMS의 `LIKE '%keyword%'`는 인덱스를 사용하지 못하고 전체 행을 순차 스캔(O(N))하는 반면, ES는 이미 분리된 토큰 목록에서 조회하므로 수억 건 규모에서도 빠릅니다. 기술 선택 기준으로는 ACID 트랜잭션이 필요한 정형 데이터, 정확한 조건 필터링이 중심이라면 RDBMS가 적합하고, 형태소 분석·오타 허용·자동완성처럼 비정형 텍스트 검색이 핵심이라면 ES를 선택합니다.

**꼬리 질문 예시**:
- "ES가 RDBMS보다 항상 빠른가요?" → 정확한 조건 필터(WHERE id = 123)는 RDBMS가 빠름. ES는 텍스트 검색에 특화.
- "샤드(Shard)가 무엇인가요?" → ES 인덱스를 수평으로 분할하는 단위. 여러 노드에 분산되어 병렬 처리 가능.
- "Near Real-Time이란?" → 문서 저장 후 검색 가능해지기까지 약 1초의 지연 발생 (refresh interval)

**면접 세션 피드백 (2026-03-30 1회차)**:
- 잘한 점: LIKE 검색의 Full-Text 인덱스 병용 불가 문제, 형태소/오타 검색 강점, 사용 케이스 설명
- 보완: 역인덱스 구조(Term→Posting List) 설명 누락. "1만 건 이상 느려진다"는 잘못된 정보 — ES는 수억 건 처리용 설계

**면접 세션 피드백 (2026-04-02 1회차)**:
- 잘한 점: 역인덱스 방향(term → doc ID) 정확히 잡음
- 보완: Analyzer 파이프라인 순서 오류(Tokenizer가 상위 개념이 아닌 Analyzer 안에 포함됨), Term Dictionary/Posting List 구조 모름
- 핵심 추가 암기:
  - Analyzer = Character Filter → Tokenizer → Token Filter (순서 중요)
  - Term Dictionary: 정렬된 단어 목록, 이진 탐색 O(log n)
  - Posting List: `(문서ID, 빈도, 위치)` 목록
  - RDBMS 비교: "LIKE '%케이스%'는 전체 행 순차 스캔 O(n), ES는 Term Dictionary 이진 탐색 O(log n)"

**면접 세션 피드백 (2026-04-20 2회차)**:
- 잘한 점: LIKE 검색 B-tree 미활용 이유(앞 와일드카드 → 풀스캔) 정확히 연결.
- 보완: Analyzer 단계명 여전히 미언급(Character Filter / Tokenizer / Token Filter). Term Dictionary FST 구조 미언급. Posting List에 TF/position/offset 저장 사실 모름. 이력서 경험 연결 없음.
- **반복 패턴**: 2회차 연속으로 Analyzer 단계명 미언급 — 반드시 암기 최우선

**면접 세션 피드백 (2026-04-28 3회차)**:
- 잘한 점: 3단계 흐름 순서 정확. Token Filter 예시(소문자화, 불용어, 어근 추출) 구체적. Character Filter 이름 꼬리 질문에서 정확히 답변. nori 형태소 분석 이유 언급.
- 보완:
  - **Character Filter 이름**: 초기 답변에서 "문자 단위 전처리"라고만 표현 → 반드시 "Character Filter"라는 이름을 먼저 말할 것.
  - **Term Dictionary**: Analyzer 결과 토큰이 Term Dictionary에 정렬 저장, 이진 탐색 O(log N)으로 조회. 이 연결 고리 미언급.
  - **Posting List 상세**: 문서 ID뿐 아니라 TF(등장 빈도), position(위치), offset도 저장. BM25 relevance score 계산에 활용.
  - **nori 이유 구체화**: "한국어 교착어" — 어근 동일 다수 표면 형태 검색 연결.
- 점수: 7/10 (Character Filter 꼬리에서 답변, Term Dictionary/Posting List 상세 미언급)

---

## Index / Shard / Replica 구조

**난이도**: 기초

**핵심 키워드**: Index, Primary Shard, Replica, Lucene, Small Shard Problem, JVM 힙, 가용성, 읽기 성능, 쓰기 성능

**관계 구조**:
- **Index**: RDBMS의 Table에 해당하는 논리적 저장 단위. 여러 Shard로 물리 분산됨
- **Primary Shard**: Index를 물리적으로 분할한 단위. 각 Shard는 독립적인 **Lucene 인스턴스**. **Primary Shard 수는 Index 생성 후 변경 불가** — 변경하려면 Reindex 필요
- **Replica**: Primary Shard의 복사본. 가용성 + 읽기 성능 목적

**Primary Shard 과다 설정 문제**:
- 각 Shard = 독립적인 Lucene 인스턴스 → Shard 수만큼 **JVM 힙 메모리 + 파일 핸들 소비**
- **Small Shard Problem**: 데이터가 적은데 Shard가 많으면 검색 쿼리를 모든 Shard로 분산해야 해 코디네이션 비용이 오히려 증가
- Elastic 권장: Shard 하나당 **10~50GB** 유지

**Replica 트레이드오프**:

| | 가용성 | 읽기 성능 | 쓰기 성능 |
|---|---|---|---|
| Replica 증가 | ↑ (Primary 장애 시 자동 승격) | ↑ (Primary + Replica 분산) | ↓ (모든 Replica에 동기화) |
| Replica 0 | ↓ (노드 장애 = 데이터 유실) | 그대로 | ↑ |

**꼬리 질문 예시**:
- Primary Shard 수를 나중에 바꾸고 싶으면 어떻게 하나요? → Reindex(새 Index 생성 후 데이터 이동) 필요
- Replica 0으로 설정하면 성능은 좋지만 언제 문제가 되나요? → 해당 노드 장애 시 해당 Shard 데이터 유실 → 프로덕션에서 최소 Replica 1 권장

**면접 세션 피드백 (2026-04-07 3회차)**:
- Index/Shard/Replica 구조 완전 미숙지 상태 → 모범 답변으로 대체
- 핵심 암기 포인트: Primary Shard = Index 생성 후 변경 불가, Shard 하나당 10~50GB, Replica 증가 = 쓰기↓ 읽기↑

---

## Elasticsearch에서 전문 검색(Full-Text Search)이 동작하는 원리를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Analyzer, Character Filter, Tokenizer, Token Filter, BM25, TF, IDF, 관련성 점수

**모범 답변 방향**:

전문 검색은 색인(Indexing)과 검색(Query) 두 단계로 동작합니다. 색인 시에는 저장할 텍스트가 Analyzer 파이프라인을 통과합니다. 파이프라인은 Character Filter(HTML 태그 제거, 특수문자 치환) → Tokenizer(공백·형태소 기준 토큰 분리) → Token Filter(소문자화, 불용어 제거, 동의어 처리) 순서로 처리되며, 각 토큰이 역인덱스에 등록됩니다. 검색 시에는 쿼리 텍스트도 동일한 Analyzer로 분석해 검색 토큰을 만들고, Term Dictionary에서 이진 탐색(O(log N))으로 조회한 뒤 Posting List에서 문서 ID 목록을 가져옵니다. 이후 BM25 알고리즘으로 관련성 점수를 계산해 내림차순으로 정렬합니다. BM25는 TF(문서 내 단어 출현 빈도, 일정 횟수 이상은 포화 효과 적용) × IDF(전체 문서 중 희소한 단어일수록 가중치) × 문서 길이 정규화의 조합입니다. RDBMS의 `LIKE '%keyword%'`는 인덱스를 사용하지 못해 O(N) 전체 스캔인 반면, ES는 Term Dictionary 이진 탐색으로 O(log N)에 처리할 수 있다는 점이 핵심 차이입니다.

**Analyzer 파이프라인 순서 암기**:
```
Character Filter → Tokenizer → Token Filter
(HTML 제거)        (토큰 분리)   (소문자화/불용어/동의어)
```

**꼬리 질문 예시**:
- BM25에서 TF 포화(Saturation)란? → 단어가 많이 나올수록 점수 증가폭이 점점 줄어듦 → 키워드 반복 어뷰징 방지
- nori analyzer가 필요한 이유는? → 한국어는 공백 기준으로 나누면 복합어/조사 처리 불가 → 형태소 분리 필요
- `match` 쿼리와 `term` 쿼리의 차이는? → match: Analyzer 적용 후 검색 (전문 검색용), term: Analyzer 없이 정확한 값 비교 (keyword 타입 필터용)

> 출처: https://www.elastic.co/docs/solutions/search/full-text/how-full-text-works

---

## Elasticsearch Hot-Warm-Cold 아키텍처와 ILM을 설명해주세요.

**난이도**: 심화

**핵심 키워드**: Hot/Warm/Cold Phase, ILM, Rollover, Shrink, Forcemerge, Freeze, node.attr, 비용 최적화

**모범 답변 방향**:

Hot-Warm-Cold 아키텍처는 시계열 데이터(로그, 메트릭)가 시간이 지날수록 접근 빈도가 줄어드는 특성을 활용해 노드를 성능 등급별로 분리하는 비용 최적화 전략입니다. Hot 단계는 신규 색인과 활발한 검색이 일어나는 구간으로 고성능 SSD 노드에 인덱스 우선순위 50을 부여합니다. 7일이 경과하면 Warm 단계로 전환되며, 이 때 Shrink로 Primary Shard 수를 축소하고 Forcemerge로 Lucene 세그먼트를 병합해 검색 성능을 최적화한 뒤 중간 사양 노드로 이동합니다. 30일이 지나면 Cold 단계로 Freeze 처리해 메모리에서 인덱스를 해제하고, 대용량 저가 HDD 노드에서 관리합니다. 60일 이후에는 Delete 단계로 인덱스를 자동 삭제해 저장 비용을 통제합니다. ILM(Index Lifecycle Management) 정책에서 각 단계의 `min_age`와 Action 조합으로 자동 전환이 이루어지며, `GET /<index>/_ilm/explain` API로 현재 단계를 확인할 수 있습니다.

**핵심 Action 암기**:

| Action | 단계 | 효과 |
|---|---|---|
| Rollover | Hot | 크기/시간 초과 시 새 인덱스 생성 |
| Shrink | Warm | 샤드 수 축소 (Primary 1개로) |
| Forcemerge | Warm | Lucene 세그먼트 병합 → 검색 속도 향상 |
| Freeze | Cold | 메모리에서 인덱스 제거, 검색 시 on-demand 로드 |

**꼬리 질문 예시**:
- Forcemerge를 Hot 단계에서 하면 안 되는 이유는? → 색인이 활발한 Hot 단계에서는 지속적으로 새 세그먼트 생성 → Forcemerge 해도 즉시 새 세그먼트 생김. 색인이 끝난 Warm에서 해야 효과적
- ILM 정책 변경이 기존 인덱스에 즉시 반영되지 않는 이유는? → 현재 단계가 완료되고 다음 단계 전환 시 새 정책 적용. Explain API로 현재 상태 확인 가능
- Frozen 인덱스 검색 시 첫 쿼리가 느린 이유는? → 메모리에서 언로드된 상태 → 검색 시 on-demand로 다시 로드 → Cold 단계는 낮은 빈도 검색을 전제로 설계

> 출처: https://www.elastic.co/kr/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management

---

## ElasticSearch의 샤드 설계와 성능 튜닝

**난이도**: 중급

**핵심 키워드**: primary shard, replica shard, refresh interval, bulk indexing, mapping

**모범 답변 방향**:

샤드 설계는 ElasticSearch 성능의 기반입니다. Primary Shard는 인덱스를 물리적으로 분할하는 단위로 각 샤드가 독립적인 Lucene 인스턴스이며, 인덱스 생성 이후에는 변경이 불가능합니다. 변경이 필요하다면 새 인덱스로 Reindex해야 합니다. Replica Shard는 Primary의 복사본으로 노드 장애 시 가용성을 보장하고 읽기 요청을 분산해 읽기 성능을 높입니다. 성능 저하의 주요 원인은 샤드 수 과다(Small Shard Problem: 데이터가 적은데 샤드가 많으면 쿼리 코디네이션 비용 증가), dynamic mapping(필드 타입이 자동 추론되어 불필요한 색인 발생), 집계 쿼리 남용 등입니다. 최적화 방법으로는 인덱스 매핑을 명시적으로 정의하고, 대량 색인 시 refresh_interval을 늘려 세그먼트 생성 빈도를 줄이며, Bulk API로 배치 색인하는 방법이 있습니다. Elastic에서 권장하는 샤드 크기는 하나당 10~50GB입니다.

**꼬리 질문 예시**:
- "primary shard를 몇 개로 설정해야 하나요?" → 단일 샤드 권장 크기 10~50GB, 노드 수 × 1~3배
- "index와 shard의 관계는?" → index = 논리적 데이터 컨테이너, shard = 물리적 저장 단위

---

## 역인덱스 구조

### Q. Elasticsearch의 역인덱스가 RDB B-Tree 인덱스와 다른 방식으로 텍스트 검색을 빠르게 하는 이유를 설명하고, Analyzer 파이프라인과 Term Dictionary + Posting List 동작 방식을 설명해주세요.

**RDB vs 역인덱스 비교:**
- RDB LIKE '%검색어%': 접두사 특정 불가 → B-Tree 인덱스 사용 불가 → Full Table Scan
- 역인덱스: 단어(Term) → 문서 ID 목록(Posting List) 직접 매핑 → Full Scan 없이 O(log N) 검색

**Analyzer 파이프라인 (순서 중요):**
1. **Character Filter**: HTML 태그, 특수문자, 이모티콘 제거
2. **Tokenizer**: 공백/언어 규칙 기준으로 의미 있는 토큰 단위 분리
3. **Token Filter**: 조사 제거, 소문자 변환, 동의어 처리

**Term Dictionary + Posting List:**
- Term Dictionary: 토큰을 정렬된 형태로 저장 → 이진탐색 O(log N) 조회
- Posting List: 각 토큰에 연결된 문서 ID 목록 + 빈도(TF) + 위치 정보
- AND 검색: 여러 토큰의 Posting List를 교집합 연산

**면접 세션 피드백 (2026-05-03 2회차)**:
- 완벽 답변 10/10. LIKE full scan 한계, Analyzer 3단계, Term Dictionary O(log N) 이진탐색, Posting List 전부 커버.

---

## Shard와 Replica 구조

**난이도**: 기초

**핵심 키워드**: Primary Shard, Replica Shard, 라우팅 공식, hash(routing) % shard수, reindex, 장애 복구, 읽기 처리량

**모범 답변 방향**:

Primary Shard는 실제 데이터를 저장하고 색인을 처리하는 주 단위다. 문서가 색인될 때 `hash(routing) % Primary Shard 수` 공식으로 어느 샤드에 저장될지 결정된다. Replica Shard는 Primary의 완전한 복사본으로 두 가지 역할을 한다. 첫째, 읽기 요청을 Primary와 함께 분산 처리해 처리량을 높인다. 둘째, Primary 노드 장애 시 Replica 중 하나가 자동으로 Primary로 승격되어 데이터 유실 없이 서비스를 유지한다.

**Primary Shard 수를 변경할 수 없는 이유**: 라우팅 공식 `hash(routing) % Primary Shard 수` 때문이다. 수가 바뀌면 기존 문서의 위치가 모두 달라져 조회 불가. 변경하려면 새 인덱스 생성 + reindex 작업 필요. Replica Shard 수는 라우팅 공식에 영향을 주지 않아 언제든 변경 가능.

**노드 장애 시 복구 순서**: Replica → Primary 승격 → 클러스터가 새 Replica를 다른 노드에 자동 생성해 설정된 replica 수 유지.

**설계 기준**: Primary Shard 수는 처음부터 최대 예상 데이터 크기 기준으로 설계. 샤드당 10~50GB 수준 권장.

**면접 세션 피드백 (2026-05-04 1회차)**:
- "잘 모르겠습니다"로 답변. 완전 미학습 영역 확인. 핵심 3가지(Primary 역할, 변경 불가 이유, Replica 읽기 분산) 집중 암기 필요.
