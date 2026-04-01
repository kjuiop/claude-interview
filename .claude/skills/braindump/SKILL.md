---
name: braindump
description: 빠른 아이디어/개념 캡처. 구술하면 00-inbox/ 에 fleeting note로 저장하고, topics/ 로 이동할 수 있는 내용은 즉시 분류한다.
---

생각이나 개념을 빠르게 캡처한다.

## Step 1 — 입력 받기

사용자의 입력을 받는다. 형태 불문 (키워드, 문장, 코드 조각, 링크 등).

## Step 2 — 분류 판단

입력 내용이 아래 중 어디에 해당하는지 판단한다:

- **즉시 topics/ 이동 가능**: 특정 기술에 대한 명확한 개념/질문 → topics/ 파일에 바로 추가
- **더 정리 필요**: 단편적이거나 맥락 불명확 → `00-inbox/` 에 fleeting note로 저장
- **오늘 daily/ 에 추가**: 오늘 세션과 관련된 내용 → `daily/YYYY-MM-DD.md` 에 추가

## Step 3 — 저장

### topics/ 로 바로 가는 경우
해당 `topics/{기술}/concepts.md` 또는 `questions.md` 에 내용 추가.
기존 내용에 append, 중복 병합.

### 00-inbox/ 로 가는 경우
파일명: `YYYYMMDDHHmm-{핵심키워드}.md`

```markdown
---
date: YYYY-MM-DD HH:mm
tags: [관련기술]
status: inbox
---

# {핵심 키워드}

{캡처한 내용}

---
처리 기한: {오늘로부터 7일 후}
이동 후보: topics/{기술명}/
```

## Step 4 — 확인

저장 위치와 내용을 한 줄로 요약해서 출력한다.
예: "`topics/kafka/concepts.md` 에 Consumer Group 재조정 개념 추가 완료"
