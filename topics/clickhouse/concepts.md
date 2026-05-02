---
tags: [clickhouse, olap, columnar, mergetree]
related: [mysql, kafka, elasticsearch]
---

# ClickHouse 핵심 개념

## 컬럼형 스토리지 (Columnar Storage)

ClickHouse는 OLAP 워크로드에 특화된 컬럼형 데이터베이스입니다.

**행 기반 vs 컬럼 기반:**
- **행 기반** (MySQL/MariaDB): 한 행의 모든 컬럼을 연속으로 저장 → OLTP 단건 조회에 유리
- **컬럼 기반** (ClickHouse): 같은 컬럼의 데이터를 연속으로 저장 → 집계 쿼리 I/O 최소화

**집계 쿼리 I/O 감소 원리:**
- `SELECT count(*), sum(clicks) FROM ad_events WHERE date = '2026-05-01'`
- 필요한 컬럼(date, clicks)만 디스크에서 읽음 → 나머지 컬럼은 스캔하지 않음
- 같은 컬럼끼리 연속 저장이라 압축률도 높음

**단건 INSERT가 느린 이유:**
- 행 하나를 INSERT할 때 모든 컬럼 파일을 각각 열어서 쓰는 I/O 오버헤드
- 배치 INSERT 권장: Kafka나 배치를 통해 묶어서 적재

---

## MergeTree 엔진

ClickHouse의 핵심 스토리지 엔진. 대부분의 프로덕션 테이블에서 사용.

### ORDER BY (Primary Key 역할)

```sql
CREATE TABLE ad_events (
    advertiser_id UInt64,
    event_date Date,
    clicks UInt32
) ENGINE = MergeTree()
ORDER BY (advertiser_id, event_date);
```

- ORDER BY는 단순 정렬이 아니라 **Primary Key 역할**
- 데이터를 ORDER BY 기준으로 정렬해서 디스크에 저장
- WHERE 조건이 ORDER BY 컬럼을 포함하면 **전체 스캔 없이 범위 탐색** 가능
- 선두 컬럼부터 순서대로 효과적 (MySQL 복합 인덱스 원리와 동일)

**설계 기준:**
- 가장 자주 필터링하는 컬럼을 선두에
- 카디널리티 낮은 컬럼 → 높은 컬럼 순서가 일반적 (advertiser_id → event_date)

### PARTITION BY (파티션 분할)

```sql
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (advertiser_id, event_date);
```

- 데이터를 물리적으로 분리
- **파티션 프루닝**: 쿼리 실행 시 해당 파티션만 스캔 → I/O 대폭 감소
- `DROP PARTITION`: 오래된 파티션 한 번에 삭제 가능 (DELETE보다 훨씬 빠름)
- 월별/일별 파티션이 일반적

**ORDER BY vs PARTITION BY 역할:**
| | ORDER BY | PARTITION BY |
|---|---|---|
| 역할 | 파티션 내 데이터 정렬·범위 탐색 | 파티션 단위로 물리 분리 |
| 효과 | 스파스 인덱스로 블록 스킵 | 파티션 프루닝으로 파일 스킵 |
| 변경 | 불변 (테이블 재생성 필요) | 파티션 단위 DROP 가능 |

---

## OLAP vs OLTP 선택 기준

| 기준 | ClickHouse (OLAP) | MySQL/MariaDB (OLTP) |
|---|---|---|
| 워크로드 | 집계, 분석, 대용량 스캔 | 단건 CRUD, 트랜잭션 |
| 쓰기 방식 | 배치 INSERT | 단건 INSERT |
| 강점 | GROUP BY, COUNT, SUM 집계 | 복잡한 조인, 트랜잭션 |
| 광고 사용 예 | 이벤트 로그 집계, 통계 | 캠페인 설정, 결제 처리 |
