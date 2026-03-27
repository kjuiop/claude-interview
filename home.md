---
tags: [home, index]
---

# Interview Prep — Home

> 이 파일이 Knowledge Graph의 중앙 허브입니다.
> Obsidian에서 열면 모든 노트의 연결 관계를 그래프로 볼 수 있습니다.

---

## 지원자 프로필
- [[resume/profile]] — 기술 스택, 대표 성과, 경력

---

## 지원 공고
- [[jobs/whitecube-challengers/job]] — 화이트큐브 챌린저스 백엔드

---

## 기술 지식 베이스

### Backend
- [[topics/golang/concepts]] / [[topics/golang/questions]]
- [[topics/java-kotlin/concepts]] / [[topics/java-kotlin/questions]]
- [[topics/python-fastapi/concepts]] / [[topics/python-fastapi/questions]]

### Database
- [[topics/mysql/concepts]] / [[topics/mysql/questions]]
- [[topics/postgresql/concepts]] / [[topics/postgresql/questions]]
- [[topics/redis/concepts]] / [[topics/redis/questions]]
- [[topics/mongodb/concepts]] / [[topics/mongodb/questions]]
- [[topics/elasticsearch/concepts]] / [[topics/elasticsearch/questions]]

### Messaging & Coordination
- [[topics/kafka-rabbitmq/concepts]] / [[topics/kafka-rabbitmq/questions]]
- [[topics/zookeeper/concepts]] / [[topics/zookeeper/questions]]

### Infrastructure
- [[topics/kubernetes/concepts]] / [[topics/kubernetes/questions]]

### Architecture
- [[topics/distributed-systems/concepts]] / [[topics/distributed-systems/questions]]
- [[topics/system-design/concepts]] / [[topics/system-design/questions]]

### Tools & Methodology
- [[topics/claude/concepts]] / [[topics/claude/questions]]
- [[topics/obsidian/concepts]] / [[topics/obsidian/questions]]
- [[topics/knowledge-systems/concepts]] / [[topics/knowledge-systems/questions]]

---

## 기술 관계 맵

```
[golang] ──사용── [goroutine] ──관련── [동시성]
                 [channel]            [분산시스템]
                 [gin]

[kubernetes] ──함께사용── [argocd]
                          [docker]
                          [istio]

[mysql] ──비교── [postgresql]
[kafka] ──비교── [rabbitmq]
[redis] ──관련── [분산락] ──관련── [zookeeper]
```
