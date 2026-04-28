---
tags: [mysql, database, index, query-optimization]
related: [postgresql, redis, distributed-systems, elasticsearch, java]
---

# MySQL — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/mysql/questions]]

---

## 1. 인덱스 기본

인덱스는 **조회 성능과 쓰기 성능의 트레이드오프**.

- 인덱스가 많을수록: 조회 빠름, 쓰기 느림(정렬 저장 비용), 디스크 추가 사용
- 인덱스가 없으면: Full Table Scan → 데이터 증가에 따라 선형적으로 느려짐

### 언제 인덱스를 추가하나
- 자주 사용되는 조회 API의 WHERE, JOIN, ORDER BY 컬럼
- Slow query log에서 발견된 쿼리
- 카디널리티(중복이 적은 값)가 높은 컬럼 우선

---

## 2. 복합 인덱스 (Composite Index)

여러 컬럼을 묶어 하나의 인덱스로 구성.

### 컬럼 순서 기준
1. **등치 조건(=)** 컬럼 먼저
2. **범위 조건(>, <, BETWEEN)** 컬럼 나중에
3. **카디널리티 높은** 컬럼 우선
4. **정렬(ORDER BY)** 방향도 인덱스에 반영

```sql
-- 인덱스: (status, created_at)
-- 잘 활용되는 쿼리
WHERE status = 'active' AND created_at > '2026-01-01' ORDER BY created_at DESC

-- status는 등치(=), created_at은 범위(>) → 올바른 순서
```

---

## 3. 커버링 인덱스 (Covering Index)

SELECT 하는 컬럼이 **모두 인덱스에 포함**되면 실제 row에 접근하지 않아도 됨.

```sql
-- 인덱스: (user_id, status, created_at)
SELECT user_id, status, created_at FROM orders WHERE user_id = 1;
-- → 인덱스만으로 완결 → 실제 row I/O 없음 → 빠름
```

- EXPLAIN에서 `Using index` 가 나오면 커버링 인덱스 활용 중
- 자주 조회하는 컬럼 조합을 인덱스에 포함시켜 I/O 대폭 감소 가능

---

## 4. 인덱스가 무력화되는 경우

### 앞 와일드카드 LIKE
```sql
WHERE name LIKE '%검색어%'  -- Full Scan
WHERE name LIKE '검색어%'   -- 인덱스 사용 가능
```

### 인덱스 컬럼에 함수 적용
```sql
WHERE DATE(created_at) = '2026-03-27'  -- Full Scan
-- 개선
WHERE created_at >= '2026-03-27' AND created_at < '2026-03-28'
```

### 암묵적 타입 변환
```sql
-- user_id 컬럼이 INT인데
WHERE user_id = '123'  -- 문자열로 비교 → 타입 변환 → Full Scan
WHERE user_id = 123    -- 올바른 타입
```

### OR 조건
```sql
WHERE status = 'active' OR user_id = 1  -- 인덱스 활용 어려움
-- 개선: UNION 사용 또는 각각 인덱스 설계
```

---

## 5. EXPLAIN으로 실행 계획 확인

인덱스 설계 후 반드시 `EXPLAIN`으로 검증.

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'active';
```

**주요 확인 항목:**
- `type`: ALL(Full Scan) → ref/range/const 로 개선되어야 함
- `key`: 실제로 사용된 인덱스명
- `rows`: 예상 스캔 행 수 (적을수록 좋음)
- `Extra`: `Using index`(커버링), `Using filesort`(정렬 인덱스 미활용) 확인

---

## 6. Slow Query 튜닝 순서

1. Slow query log 활성화 → 임계치 이상 쿼리 수집
2. EXPLAIN으로 실행 계획 분석
3. 인덱스 추가 또는 쿼리 재작성
4. 재실행 후 개선 확인

---

## 참고 링크
- [MySQL EXPLAIN 공식 문서](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)

---

## 7. Oracle vs MySQL 주요 차이

> 인포뱅크 필수 스택 Oracle 대비용. 4가지 핵심 차이 암기.

| 항목 | Oracle | MySQL |
|---|---|---|
| **페이징 문법** | `WHERE ROWNUM <= N` (레거시) / `FETCH FIRST N ROWS ONLY` (12c+) | `LIMIT N OFFSET M` |
| **NULL 처리** | 빈 문자열 `''` = NULL 동일 취급 | `''`와 NULL 엄격히 구분 |
| **자동 증가** | SEQUENCE 객체 별도 생성 + `seq.NEXTVAL` 명시 | `AUTO_INCREMENT` 컬럼 선언 |
| **기본 격리 수준** | `READ COMMITTED` (Non-Repeatable Read 발생 가능) | `REPEATABLE READ` (스냅샷 유지) |

**실무 주의 포인트:**
- Oracle → MySQL 마이그레이션 시 `''` = NULL 차이로 데이터 조회 누락 발생 가능
- JPA `@GeneratedValue(strategy = SEQUENCE)`는 Oracle의 SEQUENCE 객체를 활용
- MySQL의 REPEATABLE READ는 MVCC 스냅샷 기반이라 Phantom Read도 대부분 방지됨
