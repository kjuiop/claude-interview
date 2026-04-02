---
tags: [postgresql, database, mvcc, interview-questions]
related: [mysql, distributed-systems]
---

# PostgreSQL — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/postgresql/concepts]] | 비교: [[topics/mysql/questions]]

---

## MySQL vs PostgreSQL MVCC

### Q. MySQL과 PostgreSQL의 주요 차이점은 무엇인가요? MVCC 구현 방식에서 두 DB는 어떻게 다른가요?

**핵심 답변 포인트:**

1. **기본 격리 수준 차이**
   - MySQL: REPEATABLE READ (트랜잭션 내 동일 SELECT → 항상 동일 결과)
   - PostgreSQL: READ COMMITTED (각 쿼리마다 최신 커밋 스냅샷)

2. **MVCC 이전 버전 저장 위치 차이 (가장 중요)**
   - MySQL: Undo Log (별도 rollback segment) → purge thread 자동 삭제
   - PostgreSQL: 테이블 파일 자체 (dead tuple) → VACUUM이 정리

3. **실무 영향**
   - MySQL 장기 트랜잭션: Undo Log 비대화
   - PostgreSQL VACUUM 미실행: 테이블 bloat (dead tuple 누적)

**꼬리 질문 대비:**
- "PostgreSQL에서 VACUUM이 왜 필요한가요?" → dead tuple 정리, 테이블 bloat 방지
- "READ COMMITTED에서 Phantom Read가 발생할 수 있나요?" → 발생 가능, 방지 시 SERIALIZABLE 사용
- "MySQL에서 REPEATABLE READ가 기본인 이유는?" → 트랜잭션 내 일관성 보장, 금융 등 정합성 중요 서비스에 유리

**MVCC 내부 구현 비교 (핵심 암기):**

| | MySQL (InnoDB) | PostgreSQL |
|---|---|---|
| 이전 버전 저장 위치 | **Undo Log** (별도 rollback segment) | **테이블 파일 자체** (dead tuple) |
| 정리 방법 | purge thread 자동 삭제 | **VACUUM** 필요 |
| 기본 격리 수준 | **REPEATABLE READ** | **READ COMMITTED** |
| 장기 트랜잭션 영향 | Undo Log 비대화 | 테이블 bloat (dead tuple 누적) |

**Dead Tuple 동작 (PostgreSQL만의 특성):**
- `UPDATE` 시 기존 row에 `xmax` 설정 + 새 row 추가 → heap에 두 버전 공존
- VACUUM이 dead tuple 정리, OS 반환은 `VACUUM FULL`만 가능 (배타적 lock)
- Long-running transaction은 VACUUM을 막음 → bloat의 주요 원인

**모범 답변 구조:**
MySQL MVCC(Undo Log + REPEATABLE READ) → PostgreSQL MVCC(xmin/xmax로 버전 관리, dead tuple + VACUUM + READ COMMITTED) → 실무 차이(bloat vs undo log 비대) → 격리 수준 차이 동작 예시

---

## Dead Tuple과 VACUUM

**난이도**: 기초

**핵심 키워드**: Dead Tuple, MVCC, VACUUM, VACUUM FULL, Autovacuum, 테이블 Bloat, XID Wraparound

**모범 답변 방향**:

**Dead Tuple 발생 이유:**
- PostgreSQL은 MVCC로 UPDATE/DELETE 시 기존 행을 물리적으로 삭제하지 않음
- 기존 행에 "만료됨"(`xmax`) 표시만 → Dead Tuple로 힙에 남음
- 이유: 진행 중인 다른 트랜잭션이 이전 버전을 읽어야 할 수 있어서

**VACUUM 역할:**
- Dead Tuple이 차지한 공간을 "재사용 가능"으로 표시 (파일 크기는 유지)
- `VACUUM FULL`: OS에 공간 반환, 단 전체 테이블 Lock 발생 → 운영 중 지양
- `ANALYZE`와 함께 실행 시: 통계 정보 갱신 → 쿼리 플래너 최적화

**Autovacuum 없으면 발생하는 문제:**
1. **테이블 Bloat**: Dead Tuple 누적으로 디스크 과다 사용, 순차 스캔 느려짐
2. **실행계획 오류**: 통계 정보 갱신 안 됨 → 쿼리 플래너가 잘못된 인덱스 선택
3. **XID Wraparound (최악)**: Transaction ID(32비트)가 약 21억 회 후 순환 → PostgreSQL이 안전 모드로 전환, 쓰기 불가 → 서비스 중단

**면접 한 문장:**
> "PostgreSQL은 MVCC로 UPDATE/DELETE 시 Dead Tuple을 힙에 남기며, VACUUM이 이를 재사용 가능으로 표시합니다. Autovacuum 미실행 시 테이블 Bloat, 실행계획 오류, 최악의 경우 XID Wraparound로 서비스가 중단될 수 있습니다."

**MySQL 비교 포인트:**
- MySQL InnoDB: Undo Log에 이전 버전 저장 → purge thread 자동 삭제. VACUUM 불필요.
- PostgreSQL: 테이블 파일 자체에 Dead Tuple 저장 → VACUUM 필수.

**꼬리 질문 예시:**
- "Long-running transaction이 VACUUM을 막는 이유는?" → xmin이 오래된 트랜잭션이 살아있으면 그 이후의 dead tuple을 VACUUM이 지울 수 없음
- "VACUUM FULL을 피해야 하는 이유는?" → 전체 테이블 AccessExclusive Lock → 운영 중 읽기/쓰기 모두 차단

**면접 세션 피드백 (2026-04-02 3회차)**:
- 현황: 전혀 몰랐음 → 신규 암기 최우선
- 암기 우선순위: Dead Tuple(MVCC 결과) → VACUUM(재사용 표시) → Autovacuum 없으면 Bloat+오류+XID Wraparound

---

## 작성 예정

- PostgreSQL 고급 기능 (JSONB, Window Function, CTE) 활용 질문
- VACUUM / AUTOVACUUM 설정 관련 질문
- 인덱스 종류 선택 기준 (B-tree vs GIN vs GiST)
- MySQL에서 PostgreSQL 마이그레이션 시 주의사항
