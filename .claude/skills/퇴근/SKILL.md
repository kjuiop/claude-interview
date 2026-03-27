---
name: 퇴근
description: 하루 마무리. 전체 문서에 linter를 실행해 Knowledge Graph / Harness Engineering 구조에 맞게 정리한다.
---

하루 작업을 마무리하고 전체 문서를 점검한다.

## Linter 체크 항목

`topics/` 하위의 모든 `.md` 파일을 순서대로 검사한다.

### CHECK 1 — YAML frontmatter
모든 파일 최상단에 아래 항목이 있는지 확인:
```yaml
---
tags: [...]
related: [...]
---
```
없으면 파일 내용을 분석해서 자동으로 추가한다.

### CHECK 2 — home.md 링크 누락
`home.md`에 등록되지 않은 topics 폴더가 있으면 해당 섹션에 추가한다.

### CHECK 3 — wikilink 정합성
`concepts.md` 안에서 다른 기술을 언급하지만 `[[링크]]`가 없는 경우 링크를 추가한다.

### CHECK 4 — 빈 파일
내용이 없는 `concepts.md` 또는 `questions.md` 파일 목록을 출력한다.
(자동 수정하지 않고 목록만 보여준다 — 다음 날 출근 시 우선 채울 후보)

### CHECK 5 — 단방향 링크
A → B 링크가 있는데 B → A 역링크(`related`)가 없는 경우 frontmatter의 `related` 필드에 추가한다.

### CHECK 6 — daily/ 오늘 파일 확인
오늘 날짜의 `daily/YYYY-MM-DD.md`가 있으면:
- 세션 기록이 비어있으면 (`/면접`을 실행하지 않은 경우) 알림
- 오늘 다룬 topics를 `## 세션 기록` 하단에 요약

## 실행 결과 출력 형식

```
## 퇴근 Linter 결과 — YYYY-MM-DD

### 수정된 항목
- [파일명]: {수정 내용}

### 빈 파일 (다음 우선순위)
- topics/postgresql/concepts.md
- topics/elasticsearch/concepts.md
- ...

### 오늘 요약
- 진행한 면접 세션: {횟수}회
- 다룬 주제: {기술1}, {기술2}
- 내일 추천 주제: {topics에서 가장 오래 안 다룬 기술}
```

결과 출력 후 변경된 파일들을 저장한다.