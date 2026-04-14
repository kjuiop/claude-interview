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

PostgreSQL의 Dead Tuple은 MVCC 구현 방식의 직접적인 결과입니다. PostgreSQL은 UPDATE나 DELETE 시 기존 row를 물리적으로 즉시 삭제하지 않고, 해당 row의 `xmax`(해당 row를 삭제·수정한 트랜잭션 ID) 필드에 만료 표시만 남깁니다. 이렇게 하는 이유는 진행 중인 다른 트랜잭션이 이전 버전의 데이터를 읽어야 할 수 있기 때문입니다. VACUUM은 이렇게 쌓인 Dead Tuple을 정리하는 역할을 합니다. 정확히는 OS에 디스크 공간을 반환하는 것이 아니라, 해당 공간을 다음 INSERT/UPDATE에서 재사용 가능한 상태로 표시하는 것입니다. OS에 실제로 공간을 반환하려면 `VACUUM FULL`이 필요한데, 이는 전체 테이블에 배타적 잠금을 걸기 때문에 운영 중에는 사용을 피해야 합니다. 대신 `pg_repack`을 사용하면 잠금을 최소화하면서 온라인으로 테이블을 재구성할 수 있습니다. Autovacuum이 정상적으로 동작하지 않으면 세 가지 심각한 문제가 발생합니다. 첫째는 테이블 Bloat으로, Dead Tuple이 계속 쌓이면 테이블 파일이 비대해지고 Sequential Scan 시 dead page까지 읽어야 하므로 I/O가 증가합니다. 둘째는 실행 계획 오류로, 통계 정보가 갱신되지 않으면 쿼리 플래너가 실제 데이터 분포와 맞지 않는 잘못된 인덱스를 선택할 수 있습니다. 셋째이자 가장 심각한 경우는 XID Wraparound로, Transaction ID가 32비트 정수라 약 21억 회 후에 순환하는데 이 시점이 오면 PostgreSQL이 안전 모드로 전환되어 쓰기가 전면 불가능해지고 서비스가 중단됩니다.

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

**면접 세션 피드백 (2026-04-03 1회차)**:
- VACUUM이 재사용 가능 표시라는 핵심 개념 맞음
- Dead Tuple이 쿼리 성능에 미치는 영향 누락: **Page Bloat → Seq Scan 시 dead page 읽어야 함 → I/O 증가**
- VACUUM FULL 위험 이유 오답: "데드락 발생" → **ACCESS EXCLUSIVE LOCK — SELECT조차 블록, 테이블 전체 접근 불가**
- 대안 pg_repack 미언급: `pg_repack` — Lock 최소화 온라인 재구성, autovacuum threshold 튜닝으로 예방

---

## 작성 예정

- PostgreSQL 고급 기능 (JSONB, Window Function, CTE) 활용 질문
- VACUUM / AUTOVACUUM 설정 관련 질문
- 인덱스 종류 선택 기준 (B-tree vs GIN vs GiST)
- MySQL에서 PostgreSQL 마이그레이션 시 주의사항
