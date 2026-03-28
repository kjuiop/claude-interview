---
tags: [redis, 면접질문, distributed-lock, cache]
related: [redis/concepts]
---

# Redis 면접 질문

→ [[home]] | [[topics/redis/concepts]]

---

## Redis 분산락을 어떻게 구현하나요? SETNX와 Redlock의 차이도 설명해주세요.

**난이도**: 심화

**핵심 키워드**: SETNX, NX PX, Lua 스크립트, Redlock, 과반수, TTL

**모범 답변 방향**:
- 분산락이 필요한 시나리오 먼저 제시 (재고 차감, 중복 결제 방지 등)
- `SET key value NX PX {ttl}` 원자적 획득
- 락 해제는 Lua 스크립트로 확인+삭제 원자 처리 (단순 DEL 금지)
- SETNX 단점: 단일 노드 장애 시 락 유실
- Redlock: N개 노드 과반수 획득으로 단일 장애 대응
- 선택 기준: 고가용성 보장 여부에 따라 SETNX vs Redlock

**꼬리 질문 예시**:
- TTL이 만료되기 전에 작업이 끝나지 않으면 어떻게 되나요? (락 갱신/watchdog 패턴)
- 락을 해제할 때 단순히 DEL을 쓰면 안 되는 이유는? → GET 후 DEL 사이 race condition → Lua 스크립트로 원자적 처리 필요
- Redis Cluster 환경에서 SETNX를 쓰면 안 되는 케이스가 있나요?
- Lua 스크립트를 쓰지 않고 GET + DEL을 따로 실행하면 어떤 문제가 생기나요?

**면접 세션 피드백 (2026-03-28)**:
- 잘한 점: SET NX PX 원자성 + 단일 스레드 이유, DEL 위험 시나리오 설명
- 보완: Lua 스크립트 코드 수준으로 암기, TTL watchdog 패턴 숙지

> 출처: Redis 공식 문서 - https://redis.io/docs/manual/patterns/distributed-locks/

---

## Redis MULTI/EXEC 트랜잭션이란 무엇이고, RDBMS 트랜잭션과 어떻게 다른가요?

**난이도**: 중급

**핵심 키워드**: MULTI, EXEC, QUEUED, 롤백 없음, 격리, DISCARD

**모범 답변 방향**:
- MULTI로 시작 → 명령들이 QUEUED로 쌓임 → EXEC로 일괄 실행 (격리 보장)
- RDBMS와 핵심 차이: **롤백 없음** — 중간 명령 실패해도 나머지 명령은 계속 실행됨
- 원자성은 보장하지만 일관성 보장은 애플리케이션 책임
- DISCARD로 트랜잭션 취소 가능

**꼬리 질문 예시**:
- MULTI/EXEC에서 명령 하나가 실패하면 어떻게 되나요?
- WATCH를 함께 쓰는 이유는 무엇인가요?

> 출처: https://redis.io/docs/latest/develop/using-commands/transactions/

---

## go-redis에서 Pipeline과 TxPipeline의 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: Pipeline, TxPipeline, MULTI/EXEC 래핑, 원자성, 네트워크 왕복

**모범 답변 방향**:
- Pipeline: 여러 명령을 한 번의 네트워크 왕복으로 전송 → 네트워크 최적화. **원자성 없음**
- TxPipeline: 내부적으로 MULTI + 명령들 + EXEC를 묶어 전송 → **원자성 보장**
- TxPipeline 단독 사용보다 `rdb.Watch() + tx.TxPipelined()`로 낙관적 잠금 구현할 때 주로 사용
- 실패 시 `redis.TxFailedErr` 반환 → 애플리케이션에서 재시도 처리

**꼬리 질문 예시**:
- TxPipeline을 단독으로 쓰는 것과 Watch와 함께 쓰는 것의 차이는?
- 낙관적 잠금이 비관적 잠금(분산락)보다 적합한 상황은 언제인가요?

> 출처: https://redis.uptrace.dev/guide/go-redis-pipelines.html

---

## MULTI/EXEC와 Lua 스크립트의 차이점과 선택 기준은?

**난이도**: 심화

**핵심 키워드**: 조건부 로직, 원자성, 네트워크 효율, 분산락 해제

**모범 답변 방향**:
- 둘 다 원자성 보장, 롤백 없음
- MULTI/EXEC: 단순 다중 명령 묶음, 조건부 처리 불가
- Lua: 서버에서 실행 → 네트워크 왕복 1회, if/else 같은 조건부 로직 가능
- 선택 기준: 단순 묶음 → MULTI/EXEC / 조건부 처리(GET 값 보고 분기) → Lua
- 분산락 해제(확인+삭제)가 Lua를 써야 하는 대표 사례

**꼬리 질문 예시**:
- 분산락 해제를 MULTI/EXEC로 구현할 수 없는 이유는?
- Lua 스크립트의 단점은 무엇인가요? (디버깅 어려움, 길어지면 가독성 저하)

> 출처: https://dgle.dev/redis-multi-lua/

---
