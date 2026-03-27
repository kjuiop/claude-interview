---
tags: [obsidian, pkm, knowledge-graph, tools]
related: [claude, knowledge-graph, ontology]
---

# Obsidian — 핵심 개념 정리

→ [[home]] | 질문 모음: [[topics/obsidian/questions]]

---

## 1. Obsidian이란?

**로컬 마크다운 파일 기반 PKM(Personal Knowledge Management) 도구.**
노트들을 `[[wikilink]]`로 연결하면 자동으로 Knowledge Graph를 생성해준다.

### 핵심 철학
- **로컬 우선** — 모든 파일이 내 컴퓨터에 `.md` 파일로 저장됨. 서버 없음, 락인 없음
- **연결 중심** — 단순 폴더 구조가 아닌 노트 간 관계(링크)가 핵심
- **Plain Text** — 마크다운이라 어떤 에디터로도 열 수 있고, git으로 버전 관리 가능

### Notion과의 차이
| 항목 | Obsidian | Notion |
|------|----------|--------|
| 저장 위치 | 로컬 파일 (.md) | 클라우드 서버 |
| 오프라인 | 완전 지원 | 제한적 |
| 속도 | 매우 빠름 | 느릴 수 있음 |
| 관계 표현 | Knowledge Graph | 데이터베이스 링크 |
| 협업 | 약함 | 강함 |
| git 연동 | 완벽 | 불가 |

---

## 2. 핵심 기능

### Wikilink (`[[]]`)
노트 간 연결의 핵심. `[[노트이름]]`을 입력하면 해당 노트로 이동하는 링크가 생성된다.

```markdown
[[golang/concepts]]         # 다른 노트로 링크
[[golang/concepts#goroutine]]  # 특정 섹션으로 링크
[[golang|Go 언어]]           # 표시 텍스트 변경
```

### Backlink (역링크)
특정 노트를 참조하는 **모든 노트의 목록을 자동으로 추적**.
"이 개념이 어디서 언급되는지"를 별도 관리 없이 파악 가능.

```
goroutine.md 를 참조하는 노트들:
  ← golang/concepts.md
  ← 채팅서버 면접 질문.md
  ← whitecube 세션 기록.md
```

### Graph View
전체 노트 간 연결을 **시각화**해주는 뷰.
많이 연결된 노트일수록 노드가 크게 표시됨 → 핵심 개념 파악에 유용.

### Canvas
노트들을 자유롭게 배치하고 화살표로 연결하는 **2D 화이트보드**.
아키텍처 다이어그램, 면접 준비 로드맵 등에 활용 가능.

### Dataview 플러그인
마크다운 파일의 frontmatter를 기반으로 **SQL처럼 노트를 쿼리**.

```dataview
TABLE tags, related FROM "topics"
WHERE contains(tags, "interview-questions")
SORT file.name ASC
```

---

## 3. YAML Frontmatter

노트 상단에 메타데이터를 정의하는 구조. Obsidian이 이를 읽어 필터링/쿼리에 활용.

```yaml
---
tags: [golang, interview-questions]
related: [goroutine, channel, mutex]
difficulty: intermediate
last-reviewed: 2026-03-27
---
```

---

## 4. Vault 구조

Obsidian은 **폴더 하나를 Vault**로 지정해서 그 안의 모든 `.md` 파일을 관리한다.

```
vault/
├── .obsidian/       # Vault 설정 (테마, 플러그인 등)
├── home.md          # 중앙 허브 (MOC)
├── topics/          # 기술 지식 베이스
└── jobs/            # 공고별 준비 자료
```

### MOC (Map of Content)
Knowledge Graph의 허브 역할을 하는 인덱스 노트.
`home.md`처럼 다른 노트들로 가는 링크를 모아둔 파일.

---

## 5. Claude + Obsidian 조합

→ [[topics/obsidian/concepts#Claude와 함께 쓰면 좋은 이유]] 참고

### 왜 궁합이 좋은가
1. **같은 파일 포맷** — 둘 다 마크다운. Claude가 생성한 내용을 그대로 Obsidian에서 열면 됨
2. **로컬 파일** — Claude Code는 로컬 파일을 직접 읽고 씀 → Obsidian vault에 바로 쓰기 가능
3. **Knowledge Graph 자동 구축** — Claude가 wikilink를 포함해서 파일을 쓰면 Obsidian에서 그래프가 자동 생성됨
4. **CLAUDE.md = Vault 지침** — Claude가 어떤 노트를 어떻게 연결할지 규칙을 CLAUDE.md에 정의

> 이 구조의 이론적 배경: [[topics/knowledge-systems/concepts]]
> Obsidian이 구현하는 것 = Knowledge Graph / wikilink 규칙 = Ontology / Claude 연동 = Harness Engineering

### 실제 워크플로우
```
1. Claude가 면접 세션 진행
        ↓
2. topics/golang/concepts.md 에 wikilink 포함해서 자동 업데이트
        ↓
3. Obsidian에서 Graph View 열면 새 연결이 자동 반영
        ↓
4. "goroutine" 노트를 클릭하면 관련 면접 질문, 공고, 세션 기록이 Backlink로 연결
```

---

## 참고 링크
- [Obsidian 공식 사이트](https://obsidian.md)
- [Obsidian Help — Wikilinks](https://help.obsidian.md/Linking+notes+and+files/Internal+links)
- [Dataview 플러그인](https://blacksmithgu.github.io/obsidian-dataview/)
