---
tags: [mysql, database, index, interview-questions]
related: [postgresql, redis]
---

# MySQL — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/mysql/concepts]]

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
