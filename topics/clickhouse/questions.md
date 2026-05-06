---
tags: [clickhouse, olap, mergetree, interview]
related: [mysql, kafka]
---

# ClickHouse 면접 질문

## MergeTree 엔진

### Q. ClickHouse에서 MergeTree 엔진의 `ORDER BY`와 `PARTITION BY` 설정이 집계 쿼리 성능에 어떤 영향을 미치는지 설명해주세요.

**난이도:** 중급
**핵심 키워드:** ORDER BY = Primary Key, 범위 탐색, PARTITION BY 파티션 프루닝, DROP PARTITION, 배치 INSERT

**모범 답변 방향:**

ORDER BY는 단순 정렬이 아니라 Primary Key 역할을 합니다. MergeTree는 데이터를 ORDER BY 기준으로 정렬해서 디스크에 저장하기 때문에, WHERE 조건이 ORDER BY 컬럼을 포함하면 전체 스캔 없이 해당 범위만 읽는 범위 탐색이 가능합니다. 광고 집계 테이블에서 `ORDER BY (advertiser_id, event_date)`로 설정하면, 특정 광고주의 날짜 범위 집계 쿼리가 해당 블록만 읽어서 처리할 수 있습니다.

PARTITION BY는 데이터를 물리적으로 분리합니다. `PARTITION BY toYYYYMM(event_date)`처럼 날짜 단위로 설정하면, 쿼리 실행 시 해당 파티션만 스캔하는 파티션 프루닝이 동작해서 I/O를 대폭 줄일 수 있습니다. 오래된 파티션은 `DROP PARTITION`으로 한 번에 삭제 가능합니다.

단건 INSERT는 컬럼 파일 분산 기록으로 느리기 때문에 Kafka나 배치를 통해 묶어서 적재하는 구조가 일반적입니다.

**꼬리 질문:**
- ORDER BY 컬럼 순서를 결정하는 기준은 무엇인가요?
- PARTITION BY를 날짜로 나누면 어떤 쿼리에서 파티션 프루닝이 동작하지 않나요?

---

## OLAP vs OLTP

### Q. ClickHouse를 MySQL 대신 선택하는 기준은 무엇인가요?

**난이도:** 기초
**핵심 키워드:** OLAP 집계, 컬럼형 스토리지, 배치 INSERT, 단건 CRUD 부적합

**모범 답변 방향:**

ClickHouse는 집계·분석 워크로드(OLAP)에 특화돼 있어 `GROUP BY`, `COUNT`, `SUM` 같은 집계 쿼리에서 MySQL 대비 압도적으로 빠릅니다. 컬럼형 스토리지이기 때문에 집계에 필요한 컬럼만 읽어 I/O가 최소화됩니다.

반면 단건 INSERT, 복잡한 JOIN, 트랜잭션이 필요한 OLTP 워크로드에는 적합하지 않습니다. 광고 플랫폼에서는 캠페인 설정·결제·정산처럼 트랜잭션이 필요한 데이터는 MySQL/MariaDB에, 클릭·노출 이벤트 로그 집계는 ClickHouse에 저장하는 구조가 일반적입니다.

**꼬리 질문:**
- ClickHouse에서 단건 INSERT가 느린 이유를 설명해주세요.
- 광고 이벤트 데이터를 ClickHouse에 적재할 때 권장하는 방법은?

---

## MergeTree ORDER BY vs PARTITION BY

### Q. ClickHouse MergeTree에서 ORDER BY가 Primary Key 역할을 하는 이유를 설명하고, PARTITION BY와 역할 차이를 구분해주세요.

**핵심 키워드:** sparse index(희소 인덱스), granule(8,192행), primary.idx, 이진 탐색, granule 스킵, PARTITION BY, 파티션 프루닝, DROP PARTITION, 자주 필터링하는 컬럼 선두 배치

**ORDER BY = Primary Key 동작 원리 (Sparse Index):**
- ClickHouse는 데이터를 **granule** 단위(기본 8,192행)로 묶어 저장
- 각 granule의 첫 번째 행의 ORDER BY 컬럼 값을 **`primary.idx`** 파일에 기록 → 희소 인덱스
- WHERE 조건이 ORDER BY 컬럼에 해당 → 희소 인덱스 이진 탐색 → 불필요한 granule 전체 스킵
- 단순 정렬이 아니라 **인덱스 구조**가 Primary Key 역할의 실체

**PARTITION BY 역할:**
- 파티션 키 기준으로 데이터를 물리적으로 분리 저장
- 쿼리에 파티션 키 조건 → 해당 파티션만 스캔 (파티션 프루닝)
- **DROP PARTITION**으로 파티션 단위 즉시 삭제 가능 → 오래된 데이터 관리

**설계 기준:**
- ORDER BY 선두 컬럼 = 자주 WHERE 조건으로 필터링하는 컬럼
- PARTITION BY = 시간 단위(월/일) — `toYYYYMM(occurred_at)`

**광고 이벤트 로그 설계 예시:**
```sql
ORDER BY (advertiser_id, occurred_at)  -- 광고주 기준 조회가 가장 빈번
PARTITION BY toYYYYMM(occurred_at)     -- 월 단위 프루닝 + DROP PARTITION
```

**꼬리 질문 예시:**
- ORDER BY 컬럼이 Primary Key처럼 동작하는 내부 메커니즘은?
  → granule 단위 저장 + primary.idx 희소 인덱스 + 이진 탐색으로 granule 스킵

**면접 세션 피드백 (2026-05-02 4회차)**:
- 잘한 점: ORDER BY 블록 스캔 효율화, PARTITION BY 프루닝 + DROP PARTITION 구분, 설계 예시 연결 7/10
- 보완: sparse index(granule + primary.idx) 개념이 "Primary Key 역할" 설명의 핵심. 컬럼형 저장과 혼동하지 말 것.

---

## ReplacingMergeTree

### Q. ReplacingMergeTree 엔진이 중복 데이터를 제거하는 방식을 설명해주세요. FINAL 키워드의 역할과 트레이드오프, MergeTree/ReplacingMergeTree/AggregatingMergeTree의 사용 사례 차이, 멱등성 보장 패턴도 함께 설명해주세요.

**난이도:** 기초~중급
**핵심 키워드:** 백그라운드 비동기 병합, 병합 시점 불예측, FINAL 강제 병합, 단일 스레드 성능 저하, AggregatingMergeTree Materialized View, event_id 멱등성, eventual consistency

**FINAL 키워드**
- 병합이 완료되기 전 SELECT하면 중복 행이 그대로 보임
- FINAL: 쿼리 시점에 강제로 병합 로직 적용 → 중복 제거된 결과 반환
- 단점: 단일 스레드 처리 → 대용량 테이블에서 쿼리 성능 크게 저하
- 대안: FINAL 없이 조회 후 앱 레벨에서 최신 버전 선택, 또는 병합 완료 후 읽는 주기 조정

**엔진 선택 기준**
| 엔진 | 사용 사례 |
|---|---|
| MergeTree | 중복 없는 시계열/이벤트 로그 |
| ReplacingMergeTree | 동일 키 최신 상태만 유지 |
| AggregatingMergeTree | SUM/COUNT 집계 상태 누적 (Materialized View) |

**멱등성 보장 패턴**
- ORDER BY에 event_id 포함
- Kafka at-least-once 환경에서 중복 삽입되어도 병합 후 단일 행으로 수렴
- 삽입 자체는 멱등하지 않지만 병합 결과가 eventual consistency로 수렴

**면접 세션 피드백 (2026-05-05 1회차)**:
- 4/10 — 백그라운드 병합 개념은 맞으나 FINAL 동작 오해(불변 잠금으로 착각), AggregatingMergeTree/멱등성 패턴 미언급
- 핵심 암기: "FINAL = 쿼리 시점 강제 병합, 단일 스레드 성능 저하"
