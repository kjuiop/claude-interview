---
tags: [obsidian, pkm, knowledge-graph, interview-questions]
related: [claude, knowledge-graph]
---

# Obsidian — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/obsidian/concepts]]

---

**Q. Obsidian을 왜 사용하나요? Notion과 비교해서 선택한 이유는?**
- 로컬 파일 기반 → git으로 버전 관리 가능, 락인 없음
- Knowledge Graph로 개념 간 관계를 시각화
- Claude Code와 같은 마크다운 기반 도구와 자연스럽게 연동
- 오프라인 완전 지원, 속도 빠름

---

**Q. Knowledge Graph와 단순 폴더/태그 구조의 차이는?**
- 폴더/태그: 단방향 분류 (하나의 카테고리에 속함)
- Knowledge Graph: 양방향 관계 (개념 A가 B와 어떤 관계인지 명시)
- Backlink로 "이 개념이 어디서 쓰이는가"를 자동 추적
- 예: `goroutine` 노트를 열면 채팅 서버 프로젝트, 면접 질문, 공고까지 연결됨
- 참고: [[topics/obsidian/concepts#Backlink (역링크)]]

---

**Q. Obsidian을 개발 학습에 어떻게 활용하나요?**
- 기술 개념마다 노트 생성 + wikilink로 관련 개념 연결
- 면접 질문 노트에서 개념 노트로 링크 → 공부하다 보면 자연스럽게 연결됨
- Dataview로 "아직 정리 안 된 topics" 쿼리해서 우선순위 파악
- 세션 기록에서 다룬 기술로 링크 → 어떤 주제를 몇 번 공부했는지 추적

---

**Q. Claude와 Obsidian을 같이 쓰면 어떤 장점이 있나요?**
- Claude가 직접 vault 파일을 읽고 써서 노트 자동 생성
- wikilink 규칙을 CLAUDE.md에 정의하면 Claude가 파일 작성 시 자동으로 관계 연결
- 면접 세션 → Claude가 topics 파일 업데이트 → Obsidian Graph View 자동 반영
- 결과적으로 공부할수록 Knowledge Graph가 풍부해지는 구조
- 참고: [[topics/obsidian/concepts#Claude + Obsidian 조합]]
