---
tags: [knowledge-graph, ontology, harness-engineering, ai, rag]
related: [obsidian, claude, distributed-systems]
---

# Knowledge Systems — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/knowledge-systems/questions]]

---

## 세 개념의 관계

```
Ontology          →    Knowledge Graph    →    Harness Engineering
(무엇이 존재하고        (그 관계를 실제           (그 구조를 시스템/AI에
 어떤 관계인가 정의)     데이터로 구축)            연결해서 활용)

  [설계/스키마]              [구현]                   [활용/통합]
```

이 프로젝트에 대입하면:
```
Ontology          →    Knowledge Graph    →    Harness Engineering
(기술-개념-공고 간        (topics/ + home.md        (Claude가 graph를 읽고
 관계 스키마 정의)         + wikilinks로 구축)        면접 질문 생성에 활용)
```

---

## 1. Ontology (온톨리지)

**"어떤 개념들이 존재하고, 서로 어떤 관계인가"를 공식적으로 정의한 스키마.**

단순 분류(taxonomy)와의 차이:
```
분류(Taxonomy):  Go → 프로그래밍 언어  (is-a 관계만)

온톨리지:
  Go  ──is-a──────→  프로그래밍 언어
  Go  ──uses──────→  goroutine
  Go  ──used-by───→  화이트큐브
  goroutine ──related-to──→  동시성
  동시성 ──required-for──→  백엔드 면접
```

관계의 **종류(label)**까지 명시하는 게 핵심.

### 온톨리지를 구성하는 요소
- **Class (클래스)**: 개념의 종류 — 기술, 회사, 개념, 질문
- **Property (속성)**: 클래스의 특성 — 난이도, 경험여부
- **Relation (관계)**: 클래스 간 연결 — uses, requires, related-to, is-a

### 이 프로젝트의 온톨리지 예시
```
Classes:
  - Technology (golang, redis, kubernetes ...)
  - Concept (goroutine, mutex, index ...)
  - Company (화이트큐브 ...)
  - Question (면접 질문들)
  - Experience (내 프로젝트 경험들)

Relations:
  Technology  ──has-concept──→  Concept
  Company     ──requires────→  Technology
  Question    ──tests────────→  Concept
  Experience  ──demonstrates→  Technology
```

---

## 2. Knowledge Graph (지식 그래프)

**온톨리지를 기반으로 실제 데이터를 노드(Node) + 엣지(Edge)의 그래프로 구축한 것.**

```
노드(Node) = 개념/엔티티
엣지(Edge) = 관계 (방향 + 레이블)

[golang] ──has-concept──→ [goroutine]
[goroutine] ──related-to──→ [channel]
[화이트큐브] ──requires──→ [golang]
[채팅서버 프로젝트] ──demonstrates──→ [goroutine]
[Q: goroutine leak?] ──tests──→ [goroutine]
```

### 단순 검색 vs Knowledge Graph
```
단순 검색:  "goroutine" 검색 → goroutine 설명 문서

Knowledge Graph:
  goroutine 노드 클릭
    → 관련 개념: channel, mutex, WaitGroup
    → 내 경험: 채팅서버에서 사용
    → 면접 질문: 5개
    → 요구하는 회사: 화이트큐브
    → 공부 필요도: (아직 questions.md가 비어있는 개념들)
```

**맥락 기반 탐색**이 가능해진다.

### 실세계 Knowledge Graph 사례
- **Google Knowledge Graph** — "아이폰" 검색 시 옆에 나오는 관련 정보 패널
- **위키피디아** — 문서 간 링크 전체가 하나의 거대한 Knowledge Graph
- **Obsidian** — 로컬 마크다운 파일의 wikilink가 Knowledge Graph
- **RAG (Retrieval-Augmented Generation)** — LLM이 Knowledge Graph를 검색해 답변 품질 향상

---

## 3. Harness Engineering (하네스 엔지니어링)

**구축한 Knowledge Graph를 실제 시스템/AI에 연결해서 "활용"하는 엔지니어링.**

"Harness"는 말 그대로 **"마구를 채우다"** — 강력한 것(지식 구조)을 제어해서 유용하게 쓴다는 의미.

### 핵심 질문
> "쌓아둔 지식을 어떻게 자동으로 꺼내 쓸 것인가?"

### Harness Engineering의 구성 요소

**1. Ingestion (수집)**
지식을 그래프에 넣는 파이프라인
```
이력서 PDF → 파싱 → 노드 생성 (기술, 경험, 성과)
공고 URL  → 크롤링 → 노드 생성 (요구기술, 회사)
면접 세션 → 요약 → 엣지 생성 (질문 ↔ 개념)
```

**2. Retrieval (검색/조회)**
필요한 지식을 그래프에서 꺼내는 전략
```
Vector Search   → 의미적으로 유사한 노드 찾기
Graph Traversal → 관계를 따라 연결된 노드 탐색
Hybrid          → 둘을 조합
```

**3. Augmentation (증강)**
꺼낸 지식을 AI 컨텍스트에 주입
```
"golang 면접 질문 만들어줘"
  → Knowledge Graph에서 golang 관련 노드 조회
  → 내 경험(채팅서버, 트랜스코더) + 화이트큐브 요구사항 + 기존 질문들 가져옴
  → Claude 컨텍스트에 주입
  → 맞춤형 질문 생성
```

**4. Feedback Loop (피드백)**
AI 응답으로 그래프를 더 풍부하게 만드는 루프
```
면접 세션 진행
  → 새 개념/질문 발견
  → topics 파일 업데이트 (wikilink 포함)
  → Knowledge Graph 자동 확장
  → 다음 세션에서 더 풍부한 컨텍스트 활용
```

---

## 4. 세 개념이 이 프로젝트에서 작동하는 방식

```
Ontology
  CLAUDE.md에 정의된 규칙들:
  "공고 등록 시 기술 스택과 profile 비교"
  "topics 파일에 wikilink 포함"
  = 어떤 개념들이 어떤 관계로 연결되어야 하는지의 스키마

       ↓

Knowledge Graph
  실제 파일들:
  home.md (허브) + topics/**/*.md + jobs/*/job.md
  + wikilinks로 연결된 관계들
  = Obsidian Graph View로 시각화되는 실제 그래프

       ↓

Harness Engineering
  Claude Code가 수행하는 것:
  - 공고 링크 → 크롤링 → job.md 생성 + profile 비교 (Ingestion)
  - 면접 질문 생성 시 topics/ 파일 읽기 (Retrieval)
  - 읽은 내용을 프롬프트 컨텍스트에 주입 (Augmentation)
  - 세션 후 topics/ 파일 업데이트 (Feedback Loop)
```

---

## 참고 링크
- [Knowledge Graph — Wikipedia](https://en.wikipedia.org/wiki/Knowledge_graph)
- [Ontology (information science) — Wikipedia](https://en.wikipedia.org/wiki/Ontology_(information_science))
- [RAG + Knowledge Graph](https://arxiv.org/abs/2404.16130)
