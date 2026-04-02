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

---

## 데이터베이스 동시성 문제 유형과 트랜잭션 격리 수준으로 어떻게 해결하나요?

**난이도**: 중급

**핵심 키워드**: Dirty Read, Non-repeatable Read, Phantom Read, Lost Update, Write Skew, 격리 수준, Serial Schedule

**모범 답변 방향**:

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

**비관적 잠금 (Pessimistic Lock)**:
- "충돌이 자주 발생한다"는 가정 → 데이터 접근 시점에 즉시 잠금 획득
- 구현: `SELECT ... FOR UPDATE` → Exclusive Lock 획득, 다른 트랜잭션 대기
- 장점: 충돌 시 재처리 비용 없음, 강한 일관성 보장
- 단점: 대기 증가 → 처리량 저하, Deadlock 위험
- 적합: 충돌 빈도 높음, 경합이 심한 데이터 (재고 차감, 좌석 예약)

**낙관적 잠금 (Optimistic Lock)**:
- "충돌이 드물다"는 가정 → 수정 시점에 버전 확인
- 구현: `version` 컬럼 추가 → UPDATE 시 `WHERE version = :읽을때버전` 조건 포함 → 0 row 업데이트면 충돌 → 재시도
- 장점: 잠금 없이 높은 처리량, Deadlock 없음
- 단점: 충돌 빈도 높으면 재시도 폭발, 재시도 로직 직접 구현 필요
- 적합: 충돌 빈도 낮음, 읽기가 압도적으로 많은 경우

**MVCC와의 관계**:
- MVCC는 낙관적 동시성 제어의 변형 — 버전(snapshot) 정보를 롤백 세그먼트에 저장
- 잠금 없이 읽기 일관성 제공 → **Reader가 Writer를 블로킹하지 않음**이 핵심
- Lock-Based Protocol: read-read만 허용, 나머지는 block → 처리량 낮음
- MVCC: read-write 간 경합 제거 → 처리량 대폭 향상

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

> 출처: https://monday9pm.com/mvcc-multi-version-concurrency-control-알아보기-e4102cd97e59
