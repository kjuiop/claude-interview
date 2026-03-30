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

**핵심 키워드**: 조건부 로직, 원자성, 네트워크 효율, 분산락 해제, Redis Cluster

**모범 답변 방향**:
- 둘 다 원자성 보장, 롤백 없음
- MULTI/EXEC: 단순 다중 명령 묶음, 조건부 처리 불가, Redis Cluster에서 Cross-slot 불가
- Lua: 서버에서 실행 → 네트워크 왕복 1회, if/else 같은 조건부 로직 가능, Cluster에서 KEYS 인자로 라우팅
- 선택 기준: 단순 묶음 → MULTI/EXEC / 조건부 처리(GET 값 보고 분기) → Lua
- 분산락 해제(GET 확인 + DEL 삭제 원자 처리)가 Lua를 써야 하는 대표 사례
- 재고 차감처럼 "현재값 확인 후 조건부 갱신"도 Lua 적합

**꼬리 질문 예시**:
- 분산락 해제를 MULTI/EXEC로 구현할 수 없는 이유는? → WATCH + MULTI/EXEC로 가능하지만 GET 결과를 EXEC 전에 읽어야 해서 흐름이 복잡. 조건 확인 후 분기가 불가능
- Lua 스크립트의 단점은 무엇인가요? → 디버깅 어려움(서버 사이드 실행), 스크립트 길어지면 가독성 저하, EVALSHA 캐시 휘발성
- Redis Cluster에서 MULTI/EXEC를 쓰면 안 되는 이유는? → 다른 슬롯의 키를 한 트랜잭션으로 묶으면 Cross-slot 에러

> 출처: https://dgle.dev/redis-multi-lua/

---

## Redis Lua 스크립트에서 EVAL과 EVALSHA의 차이는? 언제 EVALSHA를 사용해야 하나요?

**난이도**: 심화

**핵심 키워드**: EVAL, EVALSHA, SCRIPT LOAD, SHA1, 스크립트 캐싱, NOSCRIPT 에러

**모범 답변 방향**:
- EVAL: 매 호출마다 전체 스크립트를 서버에 전송 → 네트워크 오버헤드
- SCRIPT LOAD: 스크립트를 서버에 캐싱 후 SHA1 해시 반환
- EVALSHA: SHA1만 전송 → 스크립트 반복 호출 시 네트워크 효율적
- 주의: 스크립트 캐시는 휘발성 (재시작, SCRIPT FLUSH, 페일오버 시 소멸)
- NOSCRIPT 에러 발생 시 EVAL로 폴백하는 방어 코드 필수

**꼬리 질문 예시**:
- EVALSHA 사용 중 서버가 재시작되면 어떻게 되나요? → NOSCRIPT 에러 → EVAL로 재로드 필요
- 파이프라인에서 EVALSHA를 쓰면 안 되는 이유는? → 파이프라인 중 NOSCRIPT 에러가 발생해도 처리가 어려움 → EVAL 권장

> 출처: https://redis.io/docs/latest/develop/programmability/eval-intro/

---

## redis.call()과 redis.pcall()의 차이는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: redis.call, redis.pcall, 에러 처리, 스크립트 중단

**모범 답변 방향**:
- `redis.call()`: 에러 발생 시 즉시 클라이언트에 에러 전파 + 스크립트 중단
- `redis.pcall()`: 에러를 테이블로 반환 → 스크립트 내에서 catch하여 처리 가능
- 선택 기준: 에러 발생 시 그냥 실패해도 되면 call(), 에러에 따라 다른 로직 실행 필요하면 pcall()

**꼬리 질문 예시**:
- Lua 스크립트에서 롤백이 안 된다면 부분 실행 중 에러 처리는 어떻게 하나요?

> 출처: https://redis.io/docs/latest/develop/programmability/eval-intro/

---
