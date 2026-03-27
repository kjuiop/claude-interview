---
tags: [knowledge-graph, ontology, harness-engineering, interview-questions]
related: [obsidian, claude]
---

# Knowledge Systems — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/knowledge-systems/concepts]]

---

**Q. Ontology와 단순 카테고리 분류의 차이는?**
- 분류: "Go는 프로그래밍 언어다" (is-a 관계 하나)
- 온톨리지: 관계의 종류까지 명시 — uses, requires, related-to, is-a
- 온톨리지가 있어야 "Go를 요구하는 회사는?", "goroutine과 관련된 면접 질문은?" 같은 쿼리가 가능
- 참고: [[topics/knowledge-systems/concepts#1. Ontology (온톨리지)]]

---

**Q. Knowledge Graph를 실무에서 어떻게 활용할 수 있나요?**
- 추천 시스템: 사용자-상품-행동 관계 그래프로 연관 추천
- RAG: LLM이 답변 생성 전 Knowledge Graph를 검색해 컨텍스트 보강
- 검색 엔진: 키워드가 아닌 의미/관계 기반 검색
- 이 프로젝트: 기술-경험-공고를 연결해 맞춤형 면접 준비
- 참고: [[topics/knowledge-systems/concepts#2. Knowledge Graph (지식 그래프)]]

---

**Q. RAG(Retrieval-Augmented Generation)에서 Knowledge Graph의 역할은?**
- 순수 LLM: 학습 데이터에만 의존 → 최신 정보 부재, 환각 발생
- RAG: 질문 → Knowledge Graph 검색 → 관련 문서/노드 조회 → LLM 컨텍스트에 주입 → 답변
- Knowledge Graph 기반 RAG: 단순 벡터 검색보다 관계 기반 탐색이 가능해 더 풍부한 컨텍스트 제공
- 예: "golang 면접 질문" → golang 노드 → 연결된 내 경험 + 공고 요구사항까지 함께 조회

---

**Q. Harness Engineering이란 무엇이고 어떤 문제를 해결하나요?**
- 지식 구조(Knowledge Graph)를 실제 시스템에 연결해서 활용하는 엔지니어링
- 해결하는 문제: "쌓아둔 지식을 어떻게 자동으로 꺼내 쓸 것인가?"
- 4단계: Ingestion(수집) → Retrieval(조회) → Augmentation(증강) → Feedback Loop(피드백)
- 참고: [[topics/knowledge-systems/concepts#3. Harness Engineering (하네스 엔지니어링)]]

---

**Q. 이 세 개념(Ontology, Knowledge Graph, Harness Engineering)이 실제 프로젝트에서 어떻게 연결되나요?**
- Ontology = CLAUDE.md의 규칙 (어떤 개념이 어떤 관계로 연결되어야 하는지 스키마)
- Knowledge Graph = topics/ + jobs/ 파일들 + wikilinks (실제 구현된 그래프)
- Harness Engineering = Claude Code가 파일 읽기/쓰기로 그래프를 활용하는 방식
- 참고: [[topics/knowledge-systems/concepts#4. 세 개념이 이 프로젝트에서 작동하는 방식]]
