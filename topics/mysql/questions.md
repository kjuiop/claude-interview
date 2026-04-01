---
tags: [mysql, database, index, interview-questions]
related: [postgresql, redis]
---

# MySQL — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/mysql/concepts]]

---

**Q. 트랜잭션의 ACID 속성을 설명해주세요. MySQL의 기본 격리 수준과 Dirty Read / Non-Repeatable Read / Phantom Read를 연결해서 설명해주세요.**

**난이도**: 기초

**핵심 키워드**: Atomicity, Consistency, Isolation, Durability, REPEATABLE READ, MVCC, Dirty Read, Non-Repeatable Read, Phantom Read, Next-Key Lock

**ACID 정의**:
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
- 트레이드오프: 인덱스 많을수록 조회↑, 쓰기↓, 디스크 추가 사용
- 선택 기준: 자주 사용되는 조회 API → slow query → 카디널리티 높은 컬럼 우선
- 추가 포인트: EXPLAIN으로 설계 후 반드시 검증
- 참고: [[topics/mysql/concepts#1. 인덱스 기본]]

**꼬리 질문: 복합 인덱스 컬럼 순서를 어떻게 결정하나요?**
- 등치(=) → 범위(>, <) → 정렬(ORDER BY) 순서
- 카디널리티 높은 컬럼 우선
- 정렬 방향(ASC/DESC)도 인덱스에 맞춰야 효율적
- 참고: [[topics/mysql/concepts#2. 복합 인덱스 (Composite Index)]]

**꼬리 질문: 인덱스를 걸었는데 성능이 개선되지 않는 경우는?**
- LIKE 앞 와일드카드: `LIKE '%검색어'` → Full Scan
- 함수 적용: `DATE(created_at)` → Full Scan → 범위 조건으로 재작성
- 암묵적 타입 변환: INT 컬럼에 문자열 비교
- OR 조건: 인덱스 활용 어려움 → UNION 고려
- 참고: [[topics/mysql/concepts#4. 인덱스가 무력화되는 경우]]

---

**Q. 커버링 인덱스란 무엇이고 어떻게 활용하나요?**
- SELECT 컬럼이 모두 인덱스에 포함 → 실제 row 접근 없이 인덱스만으로 완결
- EXPLAIN에서 `Using index` 확인
- 활용: 자주 조회하는 컬럼 조합을 인덱스에 포함시켜 I/O 감소
- 참고: [[topics/mysql/concepts#3. 커버링 인덱스 (Covering Index)]]

---

**Q. EXPLAIN으로 무엇을 확인하나요?**
- `type`: ALL이면 Full Scan → ref/range/const로 개선 목표
- `key`: 실제 사용된 인덱스
- `rows`: 예상 스캔 행 수
- `Extra`: `Using index`(커버링), `Using filesort`(정렬 최적화 필요) 확인
- 인덱스 설계 후 반드시 실행 계획 검증하는 습관
- 참고: [[topics/mysql/concepts#5. EXPLAIN으로 실행 계획 확인]]
