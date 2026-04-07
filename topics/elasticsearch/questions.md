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
- 역인덱스 = "단어 → 문서 ID 목록" 매핑 구조 설명
- Analyzer가 저장 시점에 형태소 분리 → 각 토큰을 Term Dictionary 키로 등록
- RDBMS LIKE `%keyword%`는 Full Scan, ES는 Term Dictionary 조회 → 속도 차이 설명
- 선택 기준: 정형 데이터/ACID → RDBMS, 비정형 텍스트 검색/형태소/오타 → ES

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
- **색인 시**: 텍스트 → Analyzer 파이프라인(Character Filter → Tokenizer → Token Filter) → 토큰 → 역인덱스 등록
- **검색 시**: 쿼리도 동일 Analyzer로 분석 → Term Dictionary 이진 탐색 O(log N) → Posting List 조회
- **관련성 점수**: BM25 알고리즘 — TF(출현 빈도, 포화 적용) × IDF(희소 단어 우대) × 문서 길이 정규화
- RDBMS LIKE `%keyword%` 비교: Full Scan O(N) vs Term Dictionary 조회 O(log N)

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
- **배경**: 시계열 데이터(로그/메트릭)는 시간이 지날수록 접근 빈도 감소 → 노드를 성능 등급별 분리
- **Hot**: 신규 색인 + 활발한 검색 → 고성능 SSD 노드, 우선순위 50
- **Warm**: 7일 경과 → Shrink(샤드 축소) + Forcemerge(세그먼트 병합) → 중간 사양 노드, 우선순위 25
- **Cold**: 30일 경과 → Freeze(메모리 해제) → 대용량 저가 노드, 우선순위 0
- **Delete**: 60일 경과 → 자동 삭제로 저장 비용 관리
- ILM 정책에서 `min_age` + Action 조합으로 자동 전환

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
- primary shard: 데이터 분산 저장 단위, 한 번 설정하면 변경 불가
- replica shard: 가용성/읽기 성능 향상용 복제본
- 성능 저하 원인: 샤드 과다, dynamic mapping, 집계 쿼리 남용
- 최적화: 인덱스 매핑 명시, refresh_interval 조정, bulk API 사용

**꼬리 질문 예시**:
- "primary shard를 몇 개로 설정해야 하나요?" → 단일 샤드 권장 크기 10~50GB, 노드 수 × 1~3배
- "index와 shard의 관계는?" → index = 논리적 데이터 컨테이너, shard = 물리적 저장 단위
