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
