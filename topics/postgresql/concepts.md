---
tags: [postgresql, database, mvcc, backend]
related: [mysql, distributed-systems, elasticsearch]
---

# PostgreSQL — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/postgresql/questions]] | 비교: [[topics/mysql/concepts]]

---

## MVCC 구현 방식

### MySQL vs PostgreSQL — 핵심 차이

| | MySQL (InnoDB) | PostgreSQL |
|---|---|---|
| 이전 버전 저장 위치 | **Undo Log** (별도 rollback segment) | **테이블 파일 자체** (dead tuple) |
| 정리 방법 | purge thread 자동 삭제 | **VACUUM**이 dead tuple 정리 |
| 기본 격리 수준 | **REPEATABLE READ** | **READ COMMITTED** |
| 장기 트랜잭션 영향 | Undo Log 비대화 | 테이블 bloat (dead tuple 누적) |

### MySQL MVCC
- 기본값: **REPEATABLE READ**
- 트랜잭션 시작 후 첫 번째 SELECT 시점 스냅샷 생성
- 같은 트랜잭션 내 동일 SELECT → 항상 동일한 결과 보장 (Non-Repeatable Read 방지)
- 이전 버전은 Undo Log에 저장, 불필요해지면 purge thread가 자동 삭제

### PostgreSQL MVCC
- 기본값: **READ COMMITTED**
- 각 쿼리(statement) 실행 시점의 최신 커밋 스냅샷 사용
- 같은 트랜잭션 내라도 다른 트랜잭션이 커밋하면 이후 쿼리에서 새 데이터 보임 (Non-Repeatable Read 허용)
- 이전 버전 row를 dead tuple로 테이블 파일 자체에 보존
- VACUUM 미실행 시 dead tuple 누적 → 테이블 bloat 발생

### READ COMMITTED vs REPEATABLE READ 동작 차이
```
-- 트랜잭션 A가 실행 중
BEGIN;
SELECT count(*) FROM orders; -- 결과: 100

-- 트랜잭션 B가 새 주문 INSERT + COMMIT

SELECT count(*) FROM orders;
-- MySQL (REPEATABLE READ): 100  ← 트랜잭션 시작 시점 스냅샷 유지
-- PostgreSQL (READ COMMITTED): 101 ← 최신 커밋 반영
COMMIT;
```

---

## Dead Tuple과 VACUUM

### Dead Tuple이란
PostgreSQL UPDATE/DELETE 시 기존 row를 즉시 삭제하지 않고 테이블 파일(heap)에 **dead tuple**로 남겨둔다.

**UPDATE 시 내부 동작:**
```
UPDATE users SET age = 25 WHERE id = 1

Before:  [tuple v1: xmin=100, xmax=NULL]
After:   [tuple v1: xmin=100, xmax=105]  ← dead tuple (xmax 설정)
         [tuple v2: xmin=105, xmax=NULL]  ← live tuple (새 버전)
```

- `xmin`: 해당 row를 INSERT한 트랜잭션 ID
- `xmax`: 해당 row를 DELETE/UPDATE한 트랜잭션 ID (없으면 NULL)
- 트랜잭션은 xmin/xmax와 자신의 snapshot을 비교해 visibility를 결정

### Table Bloat 문제
- Dead tuple이 쌓이면 테이블 파일이 비대해짐 → **Table Bloat**
- Sequential scan 시 dead tuple을 건너뛰며 I/O 낭비
- Index도 dead tuple을 참조하는 entry 유지 → **Index Bloat** 동반
- **MySQL과의 차이**: MySQL은 Undo Log(별도 저장소)에 이전 버전 저장 → 테이블 파일은 항상 최신 버전만 유지

### VACUUM 동작 방식
1. Heap 스캔 → dead tuple 감지 및 TID 수집
2. Index 정리 → dead tuple 참조하는 index entry 제거 (가장 비용 큰 단계)
3. Heap 회수 → dead tuple 제거, Free Space Map 업데이트

**중요한 제약**: 일반 VACUUM은 OS에 디스크 공간 반환 안 함 (같은 테이블 재사용만 가능). OS 반환은 `VACUUM FULL` 필요 (배타적 lock).

```sql
-- Dead tuple 현황 모니터링
SELECT relname, n_live_tup, n_dead_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_ratio,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### VACUUM을 막는 Long-running Transaction
- Active transaction이 있으면 그 트랜잭션 시작 이후의 모든 dead tuple을 정리할 수 없음
- **idle in transaction** 상태도 동일하게 VACUUM을 막음
- 실무에서 장기 트랜잭션은 PostgreSQL bloat의 주요 원인

---

## 작성 예정

- PostgreSQL 고급 기능 (JSONB, 배열 타입, CTE, Window Function)
- 인덱스 종류 (B-tree, GIN, GiST, BRIN)
- 파티셔닝 전략
- MySQL과의 성능 비교 시나리오
