# Interview Prep Vault — Agent Entry Point

> Claude Code 외 다른 AI 에이전트(Gemini CLI, Kiro, Codex 등)를 위한 범용 진입점.
> Claude Code를 사용한다면 `CLAUDE.md`를 읽을 것.

---

## Vault 목적

개발자 취업 면접 대비 지식 시스템.
지원자 프로필 + 채용 공고 + 기술 지식 베이스를 연결해 맞춤형 면접 준비를 지원한다.

---

## 핵심 파일 3개 (항상 먼저 읽을 것)

1. `_harness/status.md` — 기술별 준비 수준 + 다음 우선순위
2. `_harness/constraints.md` — 파일 형식 규칙 (파일 생성 전 체크)
3. `MEMORY.md` — 마지막 세션 요약 + 미결 사항

---

## 디렉토리 구조

```
claude-interview/
├── CLAUDE.md              # Claude Code 전용 진입점
├── AGENTS.md              # 범용 에이전트 진입점 (이 파일)
├── MEMORY.md              # 세션 간 연속성
├── _harness/              # 하네스 레이어
│   ├── status.md          # 준비 수준 동적 컨텍스트
│   ├── constraints.md     # 아키텍처 제약 (기계 판독형)
│   └── entropy.md         # 불일치/gaps 추적
├── 00-inbox/              # 빠른 캡처 (fleeting notes)
├── topics/                # 기술별 지식 베이스
├── daily/                 # 일일 면접 준비 기록
├── jobs/                  # 공고별 면접 준비 (비공개)
└── resume/                # 지원자 프로필 (비공개)
```

---

## 파일 작성 규칙 요약

- `topics/` 파일: YAML frontmatter 필수 (`tags`, `related`)
- wikilink 형식: `[[topics/기술명/파일명]]` (경로 포함 필수)
- 파일 수정 전: `_harness/constraints.md` 체크
- 세션 종료 후: `_harness/status.md` + `MEMORY.md` 갱신

---

## 접근 금지 영역

- `resume/` — 개인 정보 (gitignore)
- `jobs/` — 공고 정보 (gitignore)
- `_harness/constraints.md` — 규칙 자체를 수정하려면 먼저 사용자에게 확인
