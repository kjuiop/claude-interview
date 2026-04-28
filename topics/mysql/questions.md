---
tags: [mysql, database, index, interview-questions]
related: [postgresql, redis, elasticsearch]
---

# MySQL — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/mysql/concepts]]

---

**Q. 트랜잭션의 ACID 속성을 설명해주세요. MySQL의 기본 격리 수준과 Dirty Read / Non-Repeatable Read / Phantom Read를 연결해서 설명해주세요.**

**난이도**: 기초

**핵심 키워드**: Atomicity, Consistency, Isolation, Durability, REPEATABLE READ, MVCC, Dirty Read, Non-Repeatable Read, Phantom Read, Next-Key Lock

**모범 답변:**

트랜잭션의 ACID 속성은 데이터 무결성을 보장하는 네 가지 원칙입니다. Atomicity는 원자성으로, 트랜잭션 내 작업이 모두 성공하거나 모두 실패해야 한다는 원칙입니다. 주문과 재고 차감을 하나의 트랜잭션으로 묶으면 둘 중 하나만 성공하는 상황을 막을 수 있습니다. Consistency는 일관성으로, 트랜잭션 전후에 데이터 무결성 제약 조건이 유지되어야 한다는 것입니다. FK 제약이나 Unique 키 같은 데이터베이스 규칙이 항상 지켜집니다. Isolation은 격리성으로, 동시에 실행되는 트랜잭션 간에 서로 간섭하지 않아야 한다는 원칙입니다. 격리 수준으로 조절하며, 너무 강하면 성능이 낮아지고 너무 약하면 Dirty Read 같은 이상 현상이 발생합니다. Durability는 지속성으로, 한 번 커밋된 데이터는 시스템 장애가 발생해도 영구적으로 보존됩니다. MySQL에서는 WAL(Write-Ahead Log) 방식으로 트랜잭션을 먼저 Redo Log에 기록하기 때문에 장애 후에도 복구가 가능합니다. MySQL에서 기본 격리 수준은 REPEATABLE READ입니다. 이 수준은 Dirty Read와 Non-Repeatable Read를 방지합니다. Dirty Read는 아직 커밋되지 않은 데이터를 다른 트랜잭션이 읽는 현상으로, 원 트랜잭션이 롤백되면 읽은 값이 실제로는 존재하지 않는 값이 됩니다. REPEATABLE READ에서는 발생하지 않습니다. Non-Repeatable Read는 같은 트랜잭션 내에서 동일 row를 두 번 읽었을 때 그 사이에 다른 트랜잭션의 UPDATE나 DELETE가 커밋되어 다른 값이 나오는 현상인데, InnoDB는 MVCC 스냅샷으로 방지합니다. Phantom Read는 동일한 범위 쿼리를 두 번 실행할 때 그 사이에 다른 트랜잭션의 INSERT로 새 row가 나타나는 현상입니다. SQL 표준에서는 REPEATABLE READ에서 발생할 수 있지만 InnoDB는 Next-Key Lock(Gap Lock)으로 이를 실질적으로 방지합니다. 와그에서 결제/예약 서비스를 개발할 때 초과 예약 방지를 위해 REPEATABLE READ와 SELECT FOR UPDATE를 함께 사용해서 Phantom Read와 Lost Update를 동시에 방지했습니다. 격리 수준을 높일수록 일관성이 강해지는 대신 잠금 경쟁이 심해져 처리량이 낮아지므로, 서비스 요구사항에 맞는 균형을 찾는 것이 중요합니다.

**ACID 정의 (빠른 암기):**
- A(Atomicity): 트랜잭션 내 작업이 모두 성공하거나 모두 실패 (all or nothing)
- C(Consistency): 트랜잭션 전후 데이터 무결성 제약 조건 유지 (FK, Unique 등)
- I(Isolation): 동시 트랜잭션 간 간섭 없음 (격리 수준으로 조절)
- D(Durability): 커밋된 데이터는 시스템 장애 후에도 영속

**MySQL 기본 격리 수준: REPEATABLE READ**

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 해결 | 발생 | 발생 |
| REPEATABLE READ | 해결 | 해결 (MVCC) | 발생(표준) / InnoDB는 Gap Lock으로 방지 |
| SERIALIZABLE | 해결 | 해결 | 해결 |

- **Dirty Read**: 롤백된(커밋 안 된) 데이터를 읽는 현상
- **Non-Repeatable Read**: 같은 트랜잭션에서 동일 row를 두 번 읽었을 때 다른 값 (UPDATE 발생)
- **Phantom Read**: 동일 범위 쿼리를 두 번 실행 시 새 row가 나타남 (INSERT 발생)
- **InnoDB 특이점**: REPEATABLE READ에서도 **Next-Key Lock(Gap Lock)** 으로 Phantom Read 실질적 방지 (SQL 표준과 다름)

**실무 연결 (와그 결제/예약 도메인)**:
- 초과 예약 방지: REPEATABLE READ + `SELECT ... FOR UPDATE`로 명시적 X-lock
- 결제 원자성: @Transactional + Atomicity 보장

**꼬리 질문 예시:**
- "격리 수준을 높이면 어떤 비용이 발생하나요?"
- "SELECT FOR UPDATE와 일반 SELECT의 차이는?"

**면접 세션 피드백 (2026-04-01 3회차)**:
- REPEATABLE READ·MVCC 연결 정확. Dirty Read·Non-Repeatable Read 정의 정확.
- 보완: Phantom Read = "범위 쿼리에서 INSERT로 새 row 출현". InnoDB의 Gap Lock으로 실질 방지 언급 필요.

---

**Q. MySQL에서 인덱스를 잘못 설계했을 때 발생하는 문제와, 실무에서 인덱스를 어떻게 선택하나요?**

인덱스는 조회 성능을 높여주지만, 무분별하게 설계하면 오히려 시스템 전체에 부담을 줄 수 있습니다. 인덱스가 많아질수록 SELECT는 빨라지지만, INSERT·UPDATE·DELETE 시 인덱스를 함께 갱신해야 하기 때문에 쓰기 성능이 저하되고 디스크 공간도 추가로 소모됩니다. B-Tree 인덱스의 각 리프 노드도 갱신해야 하기 때문에 인덱스 개수가 늘어날수록 쓰기 비용이 비례해서 증가합니다. 실제로 샵라이브에서 라이브 스트리밍 중 시청자 이벤트 로그를 대량으로 INSERT하는 시나리오에서, 인덱스가 5개 이상 걸려 있던 테이블의 쓰기 성능이 병목이 된 경험이 있었습니다. 이벤트 로그처럼 쓰기가 압도적으로 많고 조회 패턴이 단순한 테이블은 인덱스를 최소화하고, 집계는 별도 파이프라인으로 처리하는 방향이 더 효율적이었습니다. 그래서 저는 실무에서 인덱스를 설계할 때 "일단 다 걸어두자"는 접근보다는, 먼저 slow query log를 통해 실제로 느린 쿼리를 파악하고, 그 쿼리에서 WHERE 절이나 JOIN 조건으로 자주 등장하는 컬럼을 후보로 삼습니다. 그다음 해당 컬럼의 카디널리티, 즉 고유 값의 비율을 확인해서 카디널리티가 높은 컬럼을 우선합니다. 카디널리티가 낮은 컬럼, 예를 들어 성별이나 상태 플래그 같은 컬럼은 인덱스를 걸어도 옵티마이저가 Full Scan을 선택하는 경우가 많아 효과가 미미합니다. 또한 복합 인덱스를 설계할 때는 등치 조건(=)을 앞에, 범위 조건(BETWEEN, >, <)을 뒤에 두어야 인덱스 탐색 범위를 최소화할 수 있습니다. 설계를 마친 뒤에는 반드시 EXPLAIN으로 실행 계획을 검증해서 실제로 해당 인덱스가 사용되고 있는지, `type` 컬럼이 range나 ref 이상인지, `rows` 예상 스캔 수가 합리적인지 확인하는 것을 습관으로 삼고 있습니다. 인덱스는 만들기는 쉽지만 삭제하기 어렵고, 운영 중 삭제 시 서비스에 영향이 생길 수 있기 때문에 처음부터 신중하게 설계하는 것이 장기적으로 더 효율적입니다.
- 참고: [[topics/mysql/concepts#1. 인덱스 기본]]

**꼬리 질문: 복합 인덱스 컬럼 순서를 어떻게 결정하나요?**

복합 인덱스의 컬럼 순서는 쿼리 패턴에 따라 결정하며, 기본 원칙은 등치 비교(=) 조건을 가장 앞에, 범위 조건(>, <, BETWEEN)을 그 다음에, 정렬(ORDER BY)에 쓰이는 컬럼을 마지막에 두는 것입니다. B-Tree 인덱스는 앞에서부터 순서대로 탐색하기 때문에, 중간에 범위 조건이 들어오면 그 이후 컬럼은 인덱스 탐색에 활용되지 않습니다. 카디널리티 측면에서도 고유 값이 많은 컬럼을 앞에 배치해야 초반에 더 많이 걸러낼 수 있어 효율적입니다. 또한 ORDER BY 컬럼의 ASC·DESC 방향도 인덱스 정의와 맞춰야 filesort 없이 인덱스 정렬을 그대로 활용할 수 있습니다.
- 참고: [[topics/mysql/concepts#2. 복합 인덱스 (Composite Index)]]

**꼬리 질문: 인덱스를 걸었는데 성능이 개선되지 않는 경우는?**

인덱스가 존재하더라도 무력화되는 대표적인 패턴이 몇 가지 있습니다. 가장 흔한 경우는 LIKE 앞 와일드카드인데, `LIKE '%검색어'`처럼 앞에 %가 붙으면 B-Tree 탐색이 불가능해 Full Scan으로 빠집니다. 두 번째는 인덱스 컬럼에 함수를 씌우는 경우로, `DATE(created_at) = '2024-01-01'`처럼 작성하면 옵티마이저가 인덱스를 활용하지 못합니다. 이 경우는 `created_at BETWEEN '2024-01-01' AND '2024-01-01 23:59:59'`처럼 범위 조건으로 바꿔야 합니다. 세 번째는 암묵적 타입 변환으로, INT 컬럼을 문자열과 비교하면 내부적으로 형변환이 발생해 인덱스가 무력화됩니다. 마지막으로 OR 조건은 인덱스 활용이 어렵기 때문에 UNION으로 분리하는 것을 고려해야 합니다.
- 참고: [[topics/mysql/concepts#4. 인덱스가 무력화되는 경우]]

---

**Q. 커버링 인덱스란 무엇이고 어떻게 활용하나요?**

커버링 인덱스란 SELECT 절에서 조회하는 컬럼이 모두 인덱스 안에 포함되어 있어서, 실제 테이블 row에 접근하지 않고 인덱스만으로 쿼리가 완결되는 상태를 말합니다. 일반적으로 인덱스 탐색은 인덱스 리프 노드에서 PK를 찾고, 그 PK로 다시 클러스터드 인덱스(테이블 본체)에 접근하는 두 번의 I/O가 발생합니다. 첫 번째는 인덱스 B-Tree 탐색이고, 두 번째는 PK를 이용해 실제 데이터 페이지에 접근하는 랜덤 I/O입니다. 이 두 번째 랜덤 I/O가 특히 대용량 조회에서 병목이 되는데, 커버링 인덱스가 적용되면 이를 완전히 생략할 수 있어 성능 차이가 매우 크게 납니다. 디스크 랜덤 I/O는 순차 I/O보다 수십 배 느리기 때문에, 수십만 건 이상의 조회 쿼리에서 커버링 인덱스 적용 전후의 성능 차이는 체감될 만큼 큽니다. EXPLAIN 실행 계획에서 `Extra` 컬럼에 `Using index`가 표시되면 커버링 인덱스가 적용된 것입니다. `type` 컬럼의 `const`나 `ref`는 조회 범위에 대한 개념이고 커버링 인덱스 여부는 `Extra`의 `Using index`로 별도로 확인해야 하므로 혼동하지 않아야 합니다. 활용 방법은 자주 조회하는 SELECT 컬럼 조합을 인덱스에 포함시키는 것으로, 예를 들어 `WHERE user_id = ? ORDER BY created_at`이면서 `id, status`만 SELECT한다면 `(user_id, created_at, status)`를 포함한 복합 인덱스를 설계하면 됩니다. 지연 조인 패턴에서도 커버링 인덱스가 핵심인데, SELECT *가 필요한 대용량 페이지네이션에서 서브쿼리가 커버링 인덱스로 PK만 추출하고 외부 쿼리가 PK JOIN으로 실제 row를 로드하면 랜덤 I/O를 최소화할 수 있습니다. 카테노이드에서 채팅 메시지 목록 조회 시 user_id, room_id로 필터링하고 최신 메시지 순으로 정렬하는 쿼리에서, 커버링 인덱스를 적용해 테이블 랜덤 I/O를 제거함으로써 응답 시간을 유의미하게 단축했습니다. 다만 커버링 인덱스를 위해 인덱스에 컬럼을 무분별하게 추가하면 인덱스 크기가 커져 쓰기 성능에 영향을 줄 수 있으므로, 실제로 자주 실행되고 성능이 중요한 쿼리에 한해 선택적으로 적용하는 것이 좋습니다.
- 참고: [[topics/mysql/concepts#3. 커버링 인덱스 (Covering Index)]]

---

**Q. EXPLAIN으로 무엇을 확인하나요?**

EXPLAIN은 쿼리가 실제로 어떻게 실행되는지를 옵티마이저 관점에서 보여주는 실행 계획 도구입니다. 제가 가장 먼저 보는 컬럼은 `type`인데, `ALL`이면 테이블 풀 스캔이므로 즉시 개선이 필요한 상태고, `index`는 인덱스 풀 스캔으로 ALL보다는 낫지만 여전히 위험 신호입니다. `index` 타입은 인덱스 전체를 처음부터 끝까지 읽는 방식이라 대용량 테이블에서는 심각한 성능 문제를 유발할 수 있습니다. 실질적으로 인덱스를 잘 활용하고 있는 상태는 `range`, `ref`, `eq_ref`, `const` 수준입니다. `const`는 PK나 UNIQUE 인덱스로 정확히 1행을 조회할 때, `eq_ref`는 JOIN에서 PK나 UNIQUE로 1행이 매칭될 때, `ref`는 비고유 인덱스로 여러 행이 매칭될 때, `range`는 BETWEEN이나 부등호로 범위 검색할 때 나타납니다. 다음으로 `key` 컬럼에서 실제로 어떤 인덱스가 선택됐는지 확인하고, `rows` 컬럼에서 예상 스캔 행 수가 수십만 이상이면 인덱스 설계가 잘못됐거나 통계가 오래됐을 가능성을 의심합니다. `Extra` 컬럼도 중요한데, `Using index`는 커버링 인덱스 적용 상태로 긍정적인 신호지만, `Using filesort`는 ORDER BY가 인덱스를 타지 않아 별도 정렬이 발생하는 것이고, `Using temporary`는 GROUP BY, DISTINCT, UNION, 또는 ORDER BY와 GROUP BY의 기준 컬럼이 다를 때 임시 테이블이 생긴 것으로 둘 다 성능 저하의 위험 신호입니다. `Using filesort`는 ORDER BY 컬럼을 포함한 인덱스를 추가하고 정렬 방향을 맞추면 제거할 수 있고, `Using temporary`는 GROUP BY 컬럼에 인덱스를 추가하거나 SELECT 컬럼을 줄여 임시 테이블 크기를 줄이는 방식으로 개선합니다. 인덱스를 새로 설계하거나 수정한 후에는 반드시 EXPLAIN으로 실행 계획을 검증하고, 샵라이브에서 DB 마이그레이션 시 filesort를 발견하고 ORDER BY 인덱스를 추가해 다운타임을 줄인 것처럼 실제 데이터로 확인하는 습관을 유지하고 있습니다.
- 참고: [[topics/mysql/concepts#5. EXPLAIN으로 실행 계획 확인]]

---

## EXPLAIN 실행 계획 분석 — type 컬럼 상세

**난이도**: 기초

**핵심 키워드**: ALL, index, range, ref, eq_ref, const, 인덱스 풀 스캔, Small Shard Problem, 커버링 인덱스

**type 컬럼 성능 순서** (좋음 → 나쁨):
> `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`

| type | 의미 | 상태 |
|---|---|---|
| `const` | PK 또는 UNIQUE INDEX로 정확히 **1행** 조회 (예: `WHERE id = 1`) | 최선 |
| `eq_ref` | JOIN에서 PK/UNIQUE로 1행 매칭 | 매우 좋음 |
| `ref` | **비고유(non-unique) 인덱스**로 동등 비교 — 여러 행 매칭 가능 | 양호 |
| `range` | 인덱스 범위 검색 (BETWEEN, >, <, IN, LIKE 'prefix%') | 양호 |
| `index` | **인덱스 풀 스캔** — 인덱스 전체를 처음부터 끝까지 읽음 | 주의 |
| `ALL` | **테이블 풀 스캔** | 즉시 개선 |

**주의: `index` ≠ "인덱스가 잘 적용됐다"**
- `index`는 인덱스 풀 스캔으로 ALL보다 약간 빠르지만 **여전히 최적화 필요**
- 대용량 테이블에서는 심각한 성능 저하 유발
- `range` 이상이 실질적으로 인덱스를 활용하는 상태

**주의: `const` ≠ 커버링 인덱스**
- `const`: type 컬럼에서 PK/UNIQUE로 1행 조회
- 커버링 인덱스: `Extra` 컬럼의 `Using index`로 확인

**개선 방법**:
1. WHERE 절 컬럼에 인덱스 생성
2. 복합 인덱스: 카디널리티 높은 컬럼을 앞에, 동치 비교(=)를 범위 비교(<, >) 앞에
3. 함수/형변환으로 인덱스 무력화 제거 (예: `WHERE YEAR(created_at) = 2024` → `WHERE created_at BETWEEN ...`)
4. 커버링 인덱스로 SELECT 컬럼까지 인덱스에 포함시켜 row 접근 제거

**면접 세션 피드백 (2026-04-07 3회차)**:
- `index` 타입을 "인덱스가 잘 적용된 상태"로 오해 → 실제로는 인덱스 풀 스캔, 여전히 주의 필요
- `const` = 커버링 인덱스로 오해 → const는 type 컬럼, 커버링 인덱스는 Extra의 `Using index`
- `ref` / `eq_ref` 구분 미숙지: ref=비고유 인덱스(여러 행), eq_ref=PK/UNIQUE(1행, JOIN에서)

**면접 세션 피드백 (2026-04-20 1회차 — Using Index/filesort/temporary)**:
- 잘한 점: Using Index(커버링 인덱스·랜덤 I/O 절감) 정확. Using filesort(정렬 방향 맞춘 인덱스 추가) 최적화 방향 즉시 제시.
- 보완:
  - **Using temporary 발생 패턴 추가**: GROUP BY 외 DISTINCT, UNION, ORDER BY+GROUP BY 컬럼이 다른 경우
  - **Using temporary 최적화**: GROUP BY 컬럼 인덱스 추가(스트리밍 집계) / SELECT * → 필요 컬럼만(임시 테이블 크기 절감) / UNION → UNION ALL(중복 제거 단계 생략)
  - **이력서 연결 누락**: DB 마이그레이션(다운타임 3분→2초) 경험에서 EXPLAIN 분석으로 filesort 개선 케이스 연결 필요

**면접 세션 피드백 (2026-04-07 4회차 — slow query 디버깅 워크플로우)**:
- 잘한 점: slow query log 전체 패턴 파악 후 접근 (시니어급 사고). "100ms→3s 급변 = 쿼리 변경 실수" 가설 제시 — 실무 디버깅 경험 증거. ALL/index 위험 인식 정확.
- 보완:
  - `type` 컬럼명 명시 필요: "ALL이나 INDEX를 보면" → "`type` 컬럼이 ALL·index이면"
  - `Extra` 컬럼 추가: `Using filesort`(ORDER BY가 인덱스 미사용으로 별도 정렬), `Using temporary`(GROUP BY/DISTINCT에 임시 테이블) = 위험 신호
  - `rows` 컬럼: 예상 스캔 행 수, 수십만이면 위험
  - 인덱스 무력화 패턴 1개 언급: `WHERE YEAR(created_at) = 2024` → `BETWEEN`으로 교체 / 묵시적 형변환 주의
- 모범 답변 오프닝: *"EXPLAIN의 `type` 컬럼에서 ALL·index가 보이면 즉시 위험. `rows`가 수십만이거나 `Extra`에 Using filesort·Using temporary가 있으면 추가 확인."*

---

## 데이터베이스 동시성 문제 유형과 트랜잭션 격리 수준으로 어떻게 해결하나요?

**난이도**: 중급

**핵심 키워드**: Dirty Read, Non-repeatable Read, Phantom Read, Lost Update, Write Skew, 격리 수준, Serial Schedule

**모범 답변 방향**:

데이터베이스 동시성 문제는 크게 SQL-92 표준에서 정의한 세 가지 현상과, 실무에서 추가로 고려해야 하는 현상들로 나눌 수 있습니다. 먼저 Dirty Read는 아직 커밋되지 않은 데이터를 다른 트랜잭션이 읽었다가 원 트랜잭션이 롤백되면서 존재하지 않는 값을 읽은 결과가 되는 현상입니다. Non-repeatable Read는 같은 트랜잭션 내에서 동일한 row를 두 번 조회했을 때 그 사이에 다른 트랜잭션이 UPDATE 혹은 DELETE를 커밋해서 서로 다른 결과가 나오는 현상이고, Phantom Read는 범위 쿼리를 두 번 실행했을 때 그 사이에 INSERT가 발생해서 처음엔 없던 row가 두 번째 조회에 나타나는 현상입니다. 표준 격리 수준으로 보면, READ COMMITTED는 Dirty Read를 방지하고, REPEATABLE READ는 MVCC를 활용해 Non-repeatable Read까지 방지합니다. MVCC는 데이터 수정 시 이전 버전을 Undo Log에 보존하고, 각 트랜잭션은 자신의 시작 시점 스냅샷을 참조하기 때문에 읽기 일관성이 보장됩니다. MySQL InnoDB는 REPEATABLE READ에서도 Next-Key Lock(Gap Lock)을 사용해 Phantom Read를 실질적으로 방지하는데, 이는 SQL 표준과 다른 InnoDB만의 특징입니다. Gap Lock은 인덱스 레코드 사이의 간격에 잠금을 거는 방식으로, 범위 조건에 해당하는 구간에 새 INSERT가 들어오지 못하게 막습니다. 여기에 더해 실무에서 주의해야 할 현상이 두 가지 더 있습니다. Lost Update는 두 트랜잭션이 같은 row를 동시에 읽고 수정할 때 한 쪽의 UPDATE 결과가 다른 쪽에 덮어씌워져 유실되는 문제로, 재고 차감이나 포인트 적립 같은 숫자 누적 연산에서 특히 위험합니다. Write Skew는 두 트랜잭션이 각각 다른 row를 읽고 쓰는데 그 조합이 전체 제약 조건을 위반하게 되는 현상으로, 와그에서 예약 서비스 개발 시 동시 초과 예약 시나리오가 바로 이에 해당했습니다. 이런 경우에는 격리 수준만으로는 부족하고, `SELECT ... FOR UPDATE`로 명시적 잠금을 잡거나 SERIALIZABLE을 사용해야 안전하게 처리할 수 있습니다. 잠금을 강하게 걸수록 일관성은 높아지지만 처리량이 감소하므로, 비즈니스 요구사항에 따라 적절한 수준을 선택하는 판단이 필요합니다.

**SQL-92 표준 정의 3가지 현상**:

| 현상 | 발생 조건 | 예시 |
|---|---|---|
| **Dirty Read** | 커밋 안 된 데이터를 읽은 후 그 트랜잭션이 롤백 | A가 재고 -1 수정 중(미커밋) → B가 읽음 → A 롤백 → B는 존재하지 않는 값을 읽은 것 |
| **Non-repeatable Read** | 같은 row를 두 번 읽었는데 사이에 다른 트랜잭션이 UPDATE/DELETE | 트랜잭션 내 같은 SELECT → 다른 결과 |
| **Phantom Read** | 같은 범위 쿼리를 두 번 실행했는데 사이에 INSERT 발생 | 첫 조회엔 없던 row가 두 번째 조회에 나타남 |

**추가 동시성 문제**:
- **Lost Update**: 두 트랜잭션이 같은 row를 동시에 수정 → 한 쪽의 UPDATE 결과가 덮어씌워져 유실
- **Dirty Write**: 커밋 안 된 데이터를 덮어쓴 후 롤백 → 데이터 불일치
- **Write Skew**: 두 트랜잭션이 각각 다른 row를 읽고 쓰는데 전체 제약이 깨지는 현상 (예: 동시 초과 예약)

**격리 수준과 해결 범위**:

| 격리 수준 | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생 (InnoDB는 Gap Lock으로 실질 방지) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

- MySQL 기본: REPEATABLE READ / PostgreSQL 기본: READ COMMITTED
- 격리 수준이 높을수록 동시성 저하 → 서비스 요구사항에 맞게 선택

**꼬리 질문 예시**:
- Write Skew를 방지하려면 어떤 격리 수준이나 잠금이 필요한가요?
- InnoDB에서 REPEATABLE READ가 Phantom Read를 방지하는 원리(Gap Lock)를 설명해주세요.

> 출처: https://monday9pm.com/mvcc-multi-version-concurrency-control-알아보기-e4102cd97e59

---

## 비관적 잠금(Pessimistic Lock)과 낙관적 잠금(Optimistic Lock)의 차이와 사용 시나리오

**난이도**: 중급

**핵심 키워드**: SELECT FOR UPDATE, version 컬럼, 충돌 빈도, Deadlock, 재시도 로직

**모범 답변 방향**:

비관적 잠금과 낙관적 잠금의 핵심 차이는 충돌 가능성에 대한 가정입니다. 비관적 잠금은 "충돌이 자주 발생할 것"이라고 가정하고, 데이터에 접근하는 시점에 먼저 잠금을 획득합니다. MySQL에서는 `SELECT ... FOR UPDATE`로 Exclusive Lock을 걸면 다른 트랜잭션이 해당 row를 수정하려 할 때 잠금이 해제될 때까지 대기하게 됩니다. 충돌이 발생해도 재처리 비용 없이 강한 일관성을 보장한다는 장점이 있지만, 잠금 대기가 쌓일수록 처리량이 떨어지고 두 트랜잭션이 서로의 잠금을 기다리는 Deadlock 위험이 있습니다. Deadlock이 발생하면 MySQL이 자동으로 감지하여 한 쪽 트랜잭션을 롤백시키지만, 빈번하게 발생하면 처리량에 심각한 영향을 줍니다. 그래서 재고 차감이나 와그의 좌석 예약처럼 경합이 심하고 충돌 시 재처리 비용이 큰 경우에 주로 사용합니다. 반면 낙관적 잠금은 "충돌이 드물 것"이라고 가정하고, 잠금 없이 작업한 뒤 커밋 시점에 버전을 확인해서 충돌 여부를 판단합니다. 구현 방식은 테이블에 `version` 컬럼을 추가하고, UPDATE 시 `WHERE version = :읽을때버전` 조건을 포함해서 업데이트된 row가 0개이면 충돌로 간주하고 재시도하는 방식입니다. JPA에서는 `@Version` 어노테이션으로 간편하게 구현할 수 있습니다. 잠금이 없기 때문에 Deadlock이 발생하지 않고 처리량이 높지만, 충돌 빈도가 높아지면 재시도가 폭발적으로 증가해서 오히려 성능이 나빠질 수 있습니다. 따라서 충돌이 드문 프로필 수정이나 설정 변경 같은 시나리오에 적합합니다. MVCC는 낙관적 동시성 제어의 변형으로, 버전 정보를 Undo Log에 보관해서 읽기 트랜잭션이 잠금 없이 snapshot 시점의 데이터를 조회할 수 있게 합니다. "Readers never block Writers, Writers never block Readers"가 MVCC의 핵심이고, 이 덕분에 Lock-Based Protocol에 비해 처리량이 대폭 향상됩니다. 선택 기준은 간단히 요약하면, 충돌이 잦고 재처리 비용이 크면 비관적 잠금, 충돌이 드물고 처리량이 중요하면 낙관적 잠금입니다.

**꼬리 질문 예시**:
- 재고 차감 로직에 낙관적 잠금을 쓰면 어떤 문제가 생기나요?
- JPA에서 낙관적 잠금은 어떻게 구현하나요? (`@Version` 어노테이션)
- Deadlock이 발생했을 때 어떻게 감지하고 처리하나요?

> 출처: https://monday9pm.com/mvcc-multi-version-concurrency-control-알아보기-e4102cd97e59

---

## Lock-Based Protocol의 2PL(Two-Phase Locking)을 설명하고, MVCC가 이 문제를 어떻게 해결하나요?

**난이도**: 심화

**핵심 키워드**: 공유 잠금, 배타적 잠금, 2PL, Deadlock, Starvation, MVCC, 읽기 일관성

**모범 답변 방향**:

Lock-Based Protocol에서는 공유 잠금(S-Lock)과 배타적 잠금(X-Lock) 두 가지를 사용합니다. S-Lock은 읽기 전용이며 여러 트랜잭션이 동시에 획득할 수 있지만 쓰기는 불가능하고, X-Lock은 읽기와 쓰기 모두 가능하지만 다른 트랜잭션의 모든 접근을 차단합니다. S-Lock끼리는 공존이 가능하지만 S-X 또는 X-X 조합은 충돌합니다. 2PL은 잠금 획득과 해제를 두 단계로 나눠서, Growing Phase에서는 잠금을 획득만 할 수 있고, Shrinking Phase에서는 잠금을 해제만 할 수 있는 방식입니다. 이 규칙 덕분에 직렬 가능한 스케줄을 보장할 수 있지만, 두 트랜잭션이 서로의 잠금을 기다리는 Deadlock과 특정 트랜잭션이 무기한 대기하는 Starvation 문제가 발생할 수 있습니다. Strict 2PL은 커밋 또는 중단 시까지 모든 X-Lock을 유지해서 Cascading Rollback을 방지하는 강화된 변형입니다. 또한 read-write 간에도 잠금이 충돌하기 때문에 읽기가 많은 워크로드에서 처리량이 낮아지는 근본적인 한계가 있습니다. MVCC는 이 문제를 쓰기 연산 시 이전 버전 데이터를 Undo Log에 보존하는 방식으로 해결합니다. 읽기 트랜잭션은 잠금 없이 자신의 snapshot 시점 데이터를 조회하기 때문에 "Readers never block Writers, Writers never block Readers"가 가능해집니다. 격리 수준에 따라 snapshot 기준 시점이 달라지는데, READ COMMITTED는 각 쿼리 실행 시점의 최신 커밋을 보고, REPEATABLE READ는 트랜잭션 시작 시점의 스냅샷을 고정해서 봅니다. 카테노이드에서 채팅 서버를 운영할 때 읽기가 압도적으로 많은 메시지 조회 트래픽에서 MVCC 덕분에 읽기 잠금 없이도 일관성 있는 데이터를 제공할 수 있었습니다. 다만 MVCC에도 한계가 있는데, 버전 이력을 보관하기 위한 추가 저장공간이 필요하고, PostgreSQL에서는 갱신된 데이터가 dead tuple로 남아 테이블 Bloat이 발생하기 때문에 VACUUM을 주기적으로 실행해야 합니다. 장기 실행 트랜잭션이 있으면 그 시점 이후의 모든 이전 버전을 정리하지 못해 Undo Log가 계속 쌓이는 문제도 생깁니다.

**Lock 종류**:
- **공유 잠금(Shared Lock, S-Lock)**: 읽기 전용. 다른 트랜잭션도 읽기 잠금 획득 가능. 쓰기 불가.
- **배타적 잠금(Exclusive Lock, X-Lock)**: 읽기 + 쓰기 가능. 다른 트랜잭션의 모든 접근 차단.

**Lock 호환성**: S-S 공존 가능 / S-X, X-X 불가

**2PL(Two-Phase Locking)**:
- **Growing Phase**: 잠금 획득만 가능, 해제 불가
- **Shrinking Phase**: 잠금 해제만 가능, 새 잠금 불가
- Strict 2PL: 트랜잭션 커밋/중단 시까지 모든 X-Lock 유지 → Cascade Rollback 방지
- 문제: **Deadlock**(두 트랜잭션이 서로의 잠금을 대기), **Starvation**(특정 트랜잭션이 무기한 대기)

**MVCC의 해결**:
- 쓰기 연산 시 이전 버전을 롤백 세그먼트(Undo Log / Dead Tuple)에 보존
- 읽기 트랜잭션은 잠금 없이 자신의 snapshot 시점 데이터를 조회
- **"Readers never block Writers, Writers never block Readers"** — MVCC 핵심 원칙
- 격리 수준에 따라 snapshot 기준 시점 결정:
  - READ COMMITTED: 쿼리 실행 시점 최신 커밋
  - REPEATABLE READ: 트랜잭션 시작 시점 스냅샷

**MVCC 한계**:
- 추가 저장공간 필요 (버전 이력 보관)
- **Snapshot too old** 에러: 롤백 세그먼트가 재사용/덮어씌워져 과거 버전 참조 불가
- 트랜잭션 수준 읽기 일관성 보장 불가 (SERIALIZABLE 제외)
- PostgreSQL: dead tuple 누적 → 테이블 Bloat → VACUUM 필수

**꼬리 질문 예시**:
- MVCC 환경에서 장기 실행 트랜잭션이 시스템에 미치는 영향은?
- MySQL Serializable과 PostgreSQL Serializable(SSI)의 구현 차이는?

---

## 커버링 인덱스 + 페이지네이션 최적화 (지연 조인 패턴)

**세션 피드백 (2026-04-21)**: LIMIT OFFSET 느려지는 원인 설명은 OK. 지연 조인 SQL 패턴 미답변.

**Q. 이커머스 상품 목록에서 LIMIT OFFSET 페이지네이션이 깊어질수록 느려지는 이유와 커버링 인덱스를 활용한 해결 방법은?**

**핵심 키워드**: LIMIT OFFSET 성능 저하, 지연 조인(Deferred Join), 커버링 인덱스, 커서 기반 페이지네이션(Keyset Pagination)

**모범 답변:**

LIMIT OFFSET 페이지네이션이 깊어질수록 느려지는 근본 원인은 OFFSET 방식 자체의 동작 방식에 있습니다. `LIMIT 20 OFFSET 10000`은 데이터베이스가 10020개의 행을 실제로 읽은 다음 앞의 10000개를 버리고 마지막 20개만 반환하는 방식입니다. OFFSET이 커질수록 버려지는 행이 선형으로 증가하기 때문에, 100페이지에서의 쿼리는 1페이지 쿼리보다 100배 더 많은 데이터를 읽습니다. 특히 SELECT *처럼 전체 컬럼을 조회하는 경우, 버려질 행들에 대해서도 클러스터드 인덱스로 랜덤 I/O가 발생하기 때문에 성능 저하가 더욱 심해집니다. 인덱스가 있더라도 OFFSET이 크면 결국 인덱스 리프 노드를 10000개 순차 탐색해야 하므로 근본적인 해결이 되지 않습니다. 이를 해결하는 방법이 두 가지입니다. 첫 번째는 지연 조인(Deferred Join) 패턴으로, OFFSET 탐색 자체는 커버링 인덱스만 사용하는 서브쿼리로 처리하고, 실제 데이터 조회는 추출된 소수의 PK에 대해서만 수행하는 방식입니다. `(category_id, created_at, id)`처럼 SELECT가 필요한 id까지 인덱스에 포함시키면, 서브쿼리는 테이블 랜덤 I/O 없이 인덱스만 탐색하여 20개의 id를 추출하고, 이후 JOIN으로 실제 row를 로드하므로 랜덤 I/O가 20회로 제한됩니다. 페이지 번호 직접 이동이 필요한 경우에 적합합니다. 두 번째는 커서 기반 페이지네이션(Keyset Pagination)으로, OFFSET 자체를 제거하고 마지막으로 읽은 레코드의 ID를 기준으로 `WHERE id < last_id` 조건으로 다음 페이지를 조회합니다. 이 방법은 OFFSET에 무관하게 항상 O(1)에 가까운 탐색을 보장하지만, 임의 페이지 이동이 불가능하고 정렬 기준이 고정되어야 한다는 제약이 있습니다. 정렬 컬럼이 unique하지 않으면 동점 처리를 위한 보조 컬럼을 추가해야 하는 주의사항도 있습니다. 샵라이브에서 상품 목록과 채팅 메시지 이력 API를 설계할 때 무한 스크롤은 커서 기반으로, 페이지 번호 이동이 필요한 관리자 목록은 지연 조인 방식으로 각각 구현했습니다. 두 방식의 트레이드오프를 이해하고 서비스 요구사항에 맞게 선택하는 것이 중요합니다.

**원인**: `LIMIT 20 OFFSET 10000`은 10020개 행을 전부 스캔한 뒤 처음 10000개를 버리고 20개를 반환. OFFSET이 클수록 버려지는 행이 많아져 선형 증가.

**해결 1 — 지연 조인(Deferred Join) 패턴**:
```sql
SELECT p.* FROM products p
JOIN (
  SELECT id FROM products
  WHERE category_id = ?
  ORDER BY created_at DESC
  LIMIT 20 OFFSET 10000
) AS tmp ON p.id = tmp.id
```
- 서브쿼리는 `(category_id, created_at, id)` 커버링 인덱스만 탐색 (랜덤 I/O 없음)
- 20개 id만 추출 후 PK JOIN으로 전체 컬럼 로딩
- `SELECT *`가 필요한 상황에서도 적용 가능

**해결 2 — 커서 기반 페이지네이션(Keyset Pagination)**:
```sql
-- 첫 페이지
SELECT * FROM products WHERE category_id = ? ORDER BY id DESC LIMIT 20

-- 다음 페이지 (마지막 id 전달)
SELECT * FROM products WHERE category_id = ? AND id < {last_id} ORDER BY id DESC LIMIT 20
```
- OFFSET 자체를 제거 → 항상 O(1) 탐색
- 단점: "3페이지로 바로 이동" 불가, 정렬 기준이 고정되어야 함

**선택 기준**:
- 페이지 번호 직접 입력 필요 → 지연 조인
- 무한 스크롤 / 다음 버튼 방식 → 커서 기반이 더 효율적

**꼬리 질문**:
- SELECT * 가 필요한 상황에서 커버링 인덱스를 어떻게 활용하나요?
- 커서 기반 페이지네이션의 한계는 무엇인가요?

> 출처: https://monday9pm.com/mvcc-multi-version-concurrency-control-알아보기-e4102cd97e59

---

## Oracle vs MySQL 핵심 차이 4가지

**난이도**: 기초

**핵심 키워드**: ROWNUM, FETCH FIRST, '' = NULL, SEQUENCE, AUTO_INCREMENT, READ COMMITTED, REPEATABLE READ

**모범 답변 방향**:
1. **페이징**: MySQL `LIMIT/OFFSET` vs Oracle `ROWNUM` (ORDER BY 이전 적용 주의 → 서브쿼리 필요) / Oracle 12c+ `FETCH FIRST N ROWS ONLY`
2. **빈 문자열**: MySQL `'' ≠ NULL` vs Oracle `'' = NULL` (마이그레이션 시 데이터 불일치 위험)
3. **자동 증가**: MySQL `AUTO_INCREMENT` vs Oracle `SEQUENCE` + `NEXTVAL` / Oracle 12c+ `IDENTITY` 컬럼
4. **격리 수준**: MySQL InnoDB 기본 `REPEATABLE READ` vs Oracle 기본 `READ COMMITTED` (트랜잭션 내 스냅샷 일관성 차이)

**Oracle 경험 없을 때 어필 방법**:
> "MySQL을 6년 이상 실무에서 사용하며 인덱스 구조, 트랜잭션, JPA와의 연동을 깊이 이해하고 있습니다. RDBMS의 본질인 쿼리 최적화, 인덱스 설계, 트랜잭션 관리는 Oracle과 MySQL이 공유하는 영역입니다. 4가지 핵심 차이를 이미 파악하고 왔으며 실무 투입 후 빠르게 적응할 수 있습니다."

**꼬리 질문 예시**:
- Oracle에서 `ORDER BY`와 `ROWNUM`을 함께 쓸 때 주의점은? → ROWNUM이 ORDER BY 이전에 적용 → 서브쿼리로 감싸야 정확한 페이징
- Oracle에서 `WHERE column = ''`로 조회하면 결과가 나오나요? → NULL로 저장되므로 `IS NULL`로 조회해야 함

**면접 세션 피드백 (2026-04-27 1회차)**:
- 잘한 점: 4가지 차이 모두 정확. 격리 수준 트레이드오프까지 설명.
- 보완: Oracle 어필 방법 준비 필요 ("모르겠습니다" 금지). ROWNUM + ORDER BY 주의점 추가 언급 가능.
- 점수: 7/10 (꼬리 질문 0/2)
