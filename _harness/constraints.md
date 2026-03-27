---
type: harness-constraints
---

# 아키텍처 제약 (Architectural Constraints)

> CLAUDE.md의 산문 규칙을 기계 판독형으로 구조화한 파일.
> Claude는 파일 생성/수정 전 이 파일을 체크리스트로 사용한다.

---

## 파일 형식 제약

### frontmatter 필수 필드

모든 `topics/` 파일:
```yaml
---
tags: [기술명]          # 필수
related: [연관기술]     # 필수 (없으면 빈 배열 [])
---
```

`daily/` 파일:
```yaml
---
date: YYYY-MM-DD
topics: [오늘 다룬 기술]
session: 회사명 또는 null
---
```

`jobs/{회사}/job.md`:
```yaml
---
company: 회사명
position: 포지션명
date: 공고 등록일
status: active | closed
---
```

### 위반 예시 (금지)
- frontmatter 없는 topics 파일
- `related: null` (빈 배열 `[]` 사용)
- tags가 배열이 아닌 문자열

---

## 파일 네이밍 제약

| 위치 | 규칙 | 예시 |
|---|---|---|
| `topics/golang/` | 주제 단어, 하이픈 구분 | `clean-architecture.md` |
| `topics/{기타}/` | `concepts.md` + `questions.md` 고정 | `concepts.md` |
| `daily/` | `YYYY-MM-DD.md` | `2026-03-27.md` |
| `jobs/{회사}/sessions/` | `YYYY-MM-DD-{주제}.md` | `2026-03-27-golang.md` |

---

## wikilink 제약

### 올바른 형식
```
[[topics/golang/goroutine]]         # 전체 경로
[[topics/golang/goroutine#Goroutine Leak]]  # 섹션 링크
[[home]]                            # 루트 파일
```

### 금지 형식
```
[[goroutine]]         # 경로 없는 링크 (Obsidian에서 깨질 수 있음)
[goroutine](...)      # 마크다운 링크 (Obsidian graph에서 추적 안 됨)
```

### 링크 방향 규칙
- `topics/` → 관련 `topics/` (동등한 개념 연결)
- `daily/` → `topics/` (오늘 다룬 개념으로)
- `jobs/{회사}/sessions/` → `topics/` (세션에서 다룬 개념으로)
- `home.md` → 모든 주요 파일 (허브 역할)

---

## 디렉토리 제약

```
topics/golang/     → 주제별 파일 (concepts.md/questions.md 금지)
topics/{기타}/     → concepts.md + questions.md 만
_harness/          → status.md, constraints.md, entropy.md 만
resume/            → profile.md 만 (gitignore)
jobs/              → gitignore
```

---

## 내용 작성 제약

### topics 파일 섹션 순서
1. frontmatter
2. `# 제목 — 부제목`
3. `→ [[home]] | [[관련파일1]] | [[관련파일2]]` (네비게이션 링크)
4. `---`
5. 개념 섹션들 (`## 개념명`)
6. `## 면접 질문` (있는 경우)
7. `## 참고 링크`

### 금지 패턴
- 개념과 질문을 하나의 파일에 섞되 섹션 없이 나열
- 참고 링크 없는 외부 정보 추가 (출처 필수)
- 중복 내용을 다른 파일에 복붙 (wikilink로 참조)

---

## Entropy 감지 기준

아래 상황은 엔트로피 신호 → `_harness/entropy.md`에 기록:

1. **Broken wikilink**: 링크 대상 파일이 없음
2. **Stale status**: `_harness/status.md`에서 마지막 확인이 7일 이상
3. **Empty file**: 파일이 존재하지만 내용이 frontmatter뿐
4. **Missing frontmatter**: topics 파일에 frontmatter 없음
5. **Orphan node**: home.md에 링크되지 않은 topics 파일
6. **Harness signal**: Claude가 세션 중 정보를 못 찾은 경우
