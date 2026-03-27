# Claude Interview Prep

## 프로젝트 목적
개발자의 취업 면접 대비를 돕는 Claude Code 기반 프로젝트.
공고 링크를 받으면 `jobs/` 폴더에 공고별 디렉토리를 만들고, 지원자 프로필과 비교해 맞춤형 면접 준비를 진행한다.

---

## 지원자 프로필

→ `resume/profile.md` 참조 (gitignore 대상 — 본인이 직접 작성)

면접 준비 시 지원자 프로필이 필요하면 반드시 `resume/profile.md` 를 읽어서 사용한다.

---

## 공고 등록 방법
공고 링크를 주면 다음을 자동으로 수행한다:
1. WebFetch로 공고 내용 크롤링
2. `jobs/{회사명-포지션}/job.md` 파일 생성
3. 공고 기술 스택과 `resume/profile.md` 비교하여 갭 분석 작성
4. 면접 준비 주제 목록 작성

## 면접 세션 진행 방법
- "{회사명} 면접 시작" → 해당 `jobs/{폴더}/job.md` 읽고 세션 진행
- 세션 기록은 `jobs/{회사}/sessions/YYYY-MM-DD-{주제}.md` 에 저장
- 진행 방식: 질문 → 답변 → 꼬리 질문 → 피드백

## 지식 검색 프로토콜 (Knowledge Retrieval Protocol)

면접 질문 생성, 기술 정리, 면접 세션 진행 시 반드시 아래 순서로 실행한다.
이것이 로컬 Knowledge Graph + 웹 검색을 조합하는 Harness Engineering 워크플로우다.

### Step 1 — 로컬 Knowledge Graph 조회 (항상 먼저)
다음 파일들을 병렬로 읽는다:
1. `resume/profile.md` — 지원자 기술 스택 및 경험
2. `topics/{관련기술}/concepts.md` — 기존에 정리된 개념
3. `topics/{관련기술}/questions.md` — 기존에 쌓인 질문들
4. 면접 세션인 경우 `jobs/{회사}/job.md` — 공고 요구사항

→ 이미 정리된 내용은 중복 생성하지 않고 기존 내용 위에 쌓는다.

### Step 2 — 웹 검색으로 최신 정보 보강 (항상 실행)
로컬 파일을 읽은 후 반드시 WebSearch로 아래를 검색한다:
- `{기술명} 백엔드 개발자 면접 질문 {연도}` — 최신 면접 트렌드
- `{기술명} best practices {연도}` — 최신 실무 관행
- 면접 세션인 경우 `{회사명} 기술 스택 개발 문화` — 회사 특화 정보

→ 로컬에 없는 새로운 내용만 추가한다.

### Step 3 — 통합 및 생성
로컬 지식 + 웹 검색 결과를 조합해서:
- 지원자 경험과 연결된 질문 우선 생성
- 공고 기술 스택 기준으로 비중 조정
- 난이도 순: 기초 확인 → 경험 기반 심화 → 구조 설계

### Step 4 — Feedback Loop (세션/정리 후 항상 실행)
세션이나 기술 정리가 끝난 후:
1. 새로 발견된 개념/질문을 `topics/{기술}/` 파일에 추가
2. 새 wikilink 연결이 생겼으면 `home.md` 업데이트
3. 세션 기록은 `jobs/{회사}/sessions/YYYY-MM-DD-{주제}.md`에 저장
4. `daily/YYYY-MM-DD.md` 의 각 Q&A 피드백 하단에 관련 `topics/` 파일 wikilink 추가
   - 형식: `- 관련 개념 상세: [[topics/{기술}/questions#{섹션}]] | [[topics/{기술}/concepts#{섹션}]]`
   - 목적: daily 기록에서 개념 상세 페이지로 바로 이동 가능하게 연결

---

## topics/ 자동 업데이트 규칙

`topics/` 는 기술별 개념 정리 및 면접 질문을 누적하는 지식 베이스다.
면접 세션 중 또는 별도 요청 시 아래 규칙에 따라 자동으로 채운다.

### 디렉토리 구조
각 기술 폴더는 두 파일로 구성된다:
- `concepts.md` — 핵심 개념, 동작 원리, 주의할 점
- `questions.md` — 면접 예상 질문 + 모범 답변 방향

### 자동 업데이트 트리거
- 면접 세션에서 특정 기술 주제를 다룬 후 → 해당 `topics/{기술}/` 파일에 내용 추가
- "{기술} 정리해줘" 요청 시 → WebSearch로 최신 정보 검색 후 작성
- 새 공고 등록 시 → 공고의 기술 스택 중 비어있는 topics 파일 우선 채우기

### 업데이트 방식
- 기존 내용이 있으면 **덮어쓰지 않고 추가(append)**
- 중복 항목은 병합
- 출처가 있는 경우 하단에 참고 링크 기재

### Obsidian Knowledge Graph 규칙
이 프로젝트는 Obsidian Vault로 열어 Knowledge Graph로 탐색할 수 있다.
파일 작성/수정 시 반드시 아래 규칙을 따른다.

**1. YAML frontmatter 필수**
모든 topics 파일 최상단에 아래 형식으로 작성:
```yaml
---
tags: [기술명, 카테고리]
related: [연관기술1, 연관기술2]
---
```

**2. wikilink로 관계 명시**
- 다른 기술/개념을 언급할 때는 반드시 `[[기술명]]` 형식 사용
- topics 내 파일 참조: `[[golang/concepts]]`
- 예: "[[goroutine]]은 [[channel]]과 함께 사용된다"

**3. 링크 방향 기준**
- concepts.md → 관련 기술, 상위 개념, 연관 개념 링크
- questions.md → 해당 concepts.md, 관련 질문 파일 링크
- job.md → 요구하는 기술 topics 링크
- sessions/ → 해당 세션에서 다룬 topics 링크
- daily/ → 각 Q&A 피드백에서 관련 topics/concepts, topics/questions 섹션으로 링크

**4. home.md 업데이트**
새 topics가 추가되거나 job이 등록될 때마다 `home.md`의 해당 섹션에 링크 추가

### 기술 폴더 목록
- golang: Go 언어, goroutine, channel, interface, GC
- java-kotlin: Java/Kotlin, Spring Boot, JPA, 동시성
- mysql: 인덱스, 실행 계획, 트랜잭션, 복제
- postgresql: MVCC, MySQL과의 차이, 고급 기능
- redis: 자료구조, 캐싱 전략, 분산락, pub/sub
- mongodb: 도큐먼트 모델, 인덱스, 집계 파이프라인
- kafka-rabbitmq: 메시징 패턴, 파티션, 컨슈머 그룹, 재처리
- zookeeper: 분산 코디네이션, 노드 구조, Watch 이벤트
- kubernetes: 아키텍처, 배포 전략, HPA, 장애 대응
- distributed-systems: CAP 정리, 일관성 모델, 분산 트랜잭션
- system-design: 실시간 채팅, 대용량 처리, API 설계 패턴
- elasticsearch: 역인덱스, 검색 쿼리, RDB와의 차이
- python-fastapi: 비동기 처리, asyncio, FastAPI 구조

---

## 파일 구조
```
claude-interview/
├── CLAUDE.md                  # Claude 동작 지침 (git 공개)
├── README.md                  # 사용 방법 (git 공개)
├── topics/                    # 기술별 지식 베이스 (git 공개)
│   └── {기술명}/
│       ├── concepts.md
│       └── questions.md
├── resume/                    # gitignore — 본인 프로필 (비공개)
│   └── profile.md
└── jobs/                      # gitignore — 공고별 면접 준비 (비공개)
    └── {회사명-포지션}/
        ├── job.md
        └── sessions/
```
