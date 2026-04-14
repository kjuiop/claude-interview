---
tags: [zookeeper, distributed-systems, coordination, event-driven]
related: [distributed-systems, kafka, rabbitmq, golang]
---

# ZooKeeper — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/zookeeper/questions]]

---

## 1. ZooKeeper란?

분산 시스템의 **코디네이션 서비스**.
여러 서버가 협력할 때 필요한 공통 상태 관리(리더 선출, 설정 공유, 서비스 디스커버리)를 담당한다.

---

## 2. ZNode (노드)

ZooKeeper의 데이터 단위. 파일 시스템처럼 트리 구조로 구성됨.

### Persistent Node vs Ephemeral Node

| 종류 | 특징 | 용도 |
|------|------|------|
| Persistent | 명시적으로 삭제하기 전까지 유지 | 설정값, 영구 데이터 |
| Ephemeral | **클라이언트 세션이 끊기면 자동 삭제** | 서비스 등록, 헬스체크 |

**Ephemeral Node의 핵심 가치:**
- 클라이언트(서버)가 정상 종료하든 crash하든 세션 타임아웃 후 자동 삭제
- 별도의 crash 감지 로직 없이 "서버가 사라졌음"을 자동으로 감지
- 실무 예시: 트랜스코더 서버가 기동 시 ephemeral node 등록 → crash 시 자동 삭제 → Watch 이벤트로 감지

---

## 3. Watch 이벤트

ZNode의 변화를 **콜백(Callback)** 으로 수신하는 메커니즘.

```
애플리케이션이 ZNode에 Watch 등록
    ↓
트랜스코더 서버 crash → 세션 끊김 → ephemeral node 자동 삭제
    ↓
Watch 이벤트 발생 → 애플리케이션 콜백 수신
    ↓
"서버 사라짐" 처리 (작업 재배분 등)
```

**Polling과의 차이:**
- Polling: 매 N초마다 질의 → 상태 변화 없어도 네트워크 비용 발생
- Watch: 변화가 있을 때만 이벤트 → 불필요한 네트워크 호출 없음

---

## 4. 실무 활용 패턴

### 서비스 디스커버리
```
서버 기동 시: /services/transcoder/{server-id} 에 ephemeral node 등록
서버 종료 시: 자동 삭제 (ephemeral)
애플리케이션: /services/transcoder/ 를 Watch → 변화 시 서버 목록 갱신
```

### Disable Node (무중단 점검)
```
/services/transcoder/disabled/{server-id} 에 persistent node 생성
→ 애플리케이션이 disabled 목록 확인 → 해당 서버에 신규 작업 미배분
→ 기존 작업 완료 후 점검 진행
```

### Feature Flag (분산 설정 전환)
```
ZooKeeper Watch로 설정값 변경 감지
→ N대 서버 전체에 동시에 전환 신호 전달
→ 일관된 시점에 모든 인스턴스 동시 전환
```
실무 예시: RDS → Aurora 무중단 마이그레이션 시 Feature Flag로 DB 커넥션 전환 (다운타임 3분 → 2초)

---

## 참고 링크
- [ZooKeeper 공식 문서](https://zookeeper.apache.org/doc/current/zookeeperOver.html)
- [ZooKeeper Recipes](https://zookeeper.apache.org/doc/current/recipes.html)
