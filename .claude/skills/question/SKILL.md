---
name: question
description: 특정 주제를 웹 검색해서 해당 topics/ 디렉터리에 개념 및 면접 질문으로 정리한다.
argument-hint: "<정리할 주제 또는 기술명>"
---

`$ARGUMENTS` 에 입력된 주제를 웹 검색해서 `topics/` 에 정리한다.
`$ARGUMENTS` 가 없으면 어떤 주제를 정리할지 물어보고 중단한다.

## Step 1 — 주제 분석 및 대상 topics 디렉터리 결정

`$ARGUMENTS` 를 분석해 아래 매핑 중 가장 가까운 디렉터리를 선택한다.
없으면 적절한 이름으로 새 디렉터리를 생성한다.

| 키워드 | 디렉터리 |
|---|---|
| go, golang, goroutine, channel, interface, GC | golang |
| java, kotlin, spring, jpa, 동시성 | java-kotlin |
| mysql, 인덱스, 실행계획, 트랜잭션 | mysql |
| postgresql, mvcc, pg | postgresql |
| redis, 캐시, 캐싱, pub/sub, 분산락 | redis |
| mongodb, 도큐먼트, 집계 | mongodb |
| kafka, rabbitmq, 메시징, 큐 | kafka-rabbitmq |
| zookeeper, 분산 코디네이션 | zookeeper |
| kubernetes, k8s, hpa, 배포 | kubernetes |
| 분산 시스템, cap, 일관성, 분산 트랜잭션 | distributed-systems |
| 시스템 디자인, 대용량, api 설계 | system-design |
| elasticsearch, 역인덱스, 검색 | elasticsearch |
| python, fastapi, asyncio, 비동기 | python-fastapi |

## Step 2 — 로컬 Knowledge Graph 조회 (병렬)

아래 파일들을 동시에 읽는다. 파일이 없으면 건너뛴다.
- `topics/{결정된디렉터리}/concepts.md`
- `topics/{결정된디렉터리}/questions.md`

이미 정리된 내용을 파악해 중복 작성을 방지한다.

## Step 3 — 웹 검색으로 정보 수집

WebSearch 로 아래 쿼리를 순서대로 검색한다:
1. `{$ARGUMENTS} 개념 정리 동작 원리`
2. `{$ARGUMENTS} 백엔드 개발자 면접 질문 2025`
3. `{$ARGUMENTS} best practices 실무 2025`

각 검색 결과에서 핵심 내용만 추출하고 출처 URL을 기록한다.

## Step 4 — concepts.md 업데이트

`topics/{디렉터리}/concepts.md` 에 **로컬에 없는 새 내용만** append 한다.

파일이 없으면 아래 형식으로 새로 생성한다:

```markdown
---
tags: [{$ARGUMENTS}, backend]
related: []
---

# {$ARGUMENTS} 핵심 개념
```

추가할 내용 형식:
```markdown
## {개념명}

{개념 설명 — 동작 원리, 내부 구조, 주의할 점 포함. 표/코드블록/bullet 등 구조적 설명}

### 💬 면접 답변 형태로 읽기

{위 개념을 실제 면접에서 3분 이상 말할 수 있는 완성된 줄글로 작성.
 개략식 나열 ❌ → 연속된 문장 ✅
 구조: 핵심 주장 → 원리 설명 → 트레이드오프/주의점 → 실무 경험 연결
 분량: 600자 이상, 읽는 데 3분 이상}

> 출처: {URL}
```

규칙:
- 이미 있는 섹션은 건드리지 않는다
- 새로운 섹션만 파일 하단에 append
- [[wikilink]] 로 관련 개념 연결
- **💬 면접 답변 형태로 읽기** 섹션은 모든 새 개념에 반드시 포함한다

## Step 5 — questions.md 업데이트

`topics/{디렉터리}/questions.md` 에 **로컬에 없는 새 질문만** append 한다.

파일이 없으면 아래 형식으로 새로 생성한다:

```markdown
---
tags: [{$ARGUMENTS}, 면접질문]
related: [{디렉터리}/concepts]
---

# {$ARGUMENTS} 면접 질문
```

추가할 내용 형식:
```markdown
## {질문 본문}

**난이도**: 기초 / 중급 / 심화

**핵심 키워드**: {키워드1}, {키워드2}

**모범 답변 (3분 이상 말하기 형태)**:
> {실제 면접에서 말할 수 있는 완성된 연속 문장으로 작성.
>  개략식 포인트 나열 ❌ → "~입니다. ~때문에 ~를 사용합니다." 형식 ✅
>  분량: 600자 이상, 읽는 데 3분 이상
>  구조: 핵심 주장 먼저 → 원리 설명 → 실무/경험 연결 → 트레이드오프}

**꼬리 질문 예시**:
- {꼬리 질문 1}
- {꼬리 질문 2}

> 출처: {URL}

---
```

규칙:
- 이미 있는 질문과 중복되지 않도록 기존 파일을 확인하고 append
- 난이도 순서로 배치: 기초 → 중급 → 심화

## Step 6 — home.md 업데이트

`home.md` 를 읽어 `{디렉터리}` 항목이 없으면 해당 섹션에 링크를 추가한다.

## Step 7 — 완료 보고

아래 형식으로 결과를 출력한다:

```
✅ {$ARGUMENTS} 정리 완료

📁 저장 위치: topics/{디렉터리}/
  - concepts.md: {추가된 섹션 수}개 섹션 추가
  - questions.md: {추가된 질문 수}개 질문 추가

📌 주요 개념: {개념1}, {개념2}, {개념3}
❓ 추가된 질문 미리보기:
  1. {질문1}
  2. {질문2}
  3. {질문3}
```
