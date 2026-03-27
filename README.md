# Claude Interview Prep

Claude Code를 활용한 개발자 면접 대비 프로젝트.
공고 링크와 본인 이력서를 등록하면 Claude가 맞춤형 면접 질문을 생성하고 실전처럼 세션을 진행해준다.

## 특징

- 공고 링크만 주면 자동으로 기술 스택 분석 + 갭 분석 + 준비 주제 생성
- 이력서 기반으로 실제 경험과 연결된 심화/꼬리 질문 생성
- 면접 세션 후 기술별 concepts/questions 자동 누적
- 이력서·공고·출제이력은 gitignore로 개인 정보 보호
- Obsidian Knowledge Graph로 기술 간 연결 시각화

---

## 매일 사용하는 커맨드

| 커맨드 | 타이밍 | 동작 |
|--------|--------|------|
| `/출근` | 하루 시작 | 소스 문서 + 출제 이력 확인 → 오늘의 질문 3개 선정 → `daily/` 기록 |
| `/면접` | 준비됐을 때 | 오늘의 질문 3개로 실전 면접 진행 (질문→답변→꼬리질문→피드백) → topics 자동 업데이트 → daily에서 topics 링크 연결 |
| `/면접 golang` | 특정 주제 연습 | 해당 topic의 질문으로 면접 진행 |
| `/퇴근` | 하루 마무리 | 전체 문서 linter 실행 → Knowledge Graph 구조 정비 → 내일 추천 주제 출력 |

---

## 시작하기

### 1. 저장소 클론
```bash
git clone {repo_url}
cd claude-interview
```

### 2. 프로필 작성
`resume/` 폴더를 만들고 `profile.md` 를 작성한다. (gitignore 대상이므로 git에 올라가지 않는다)

```bash
mkdir resume
```

`resume/profile.md` 를 아래 형식으로 작성:

```markdown
# 지원자 프로필

## 기본 정보
- 이름: 홍길동
- 직무: Backend Developer
- 경력: N년

## 핵심 강점
- ...

## 기술 스택
- Back-end: ...
- Database: ...
- DevOps: ...

## 대표 성과
1. ...

## 경력
- 회사명 (기간): 담당 업무
```

### 3. 공고 등록
Claude Code를 열고 공고 링크를 붙여넣으면 된다.

```
https://www.wanted.co.kr/wd/xxxxxx
```

Claude가 자동으로 아래를 수행한다:
- 공고 크롤링
- `jobs/{회사명-포지션}/job.md` 생성
- 내 프로필과 비교한 갭 분석 작성
- 면접 준비 주제 목록 작성

### 4. 면접 세션 시작
```
{회사명} 면접 시작해줘
```

진행 방식: **질문 → 내 답변 → 꼬리 질문 → 피드백**

세션이 끝나면 아래가 자동으로 수행된다:
- `daily/YYYY-MM-DD.md` 에 Q&A 전문 + 피드백 기록
- 관련 기술의 `topics/` 파일 자동 업데이트 (concepts/questions 누적)
- `daily/` 의 각 Q&A 피드백에서 해당 `topics/` 섹션으로 wikilink 연결

---

## 디렉토리 구조

```
claude-interview/
├── CLAUDE.md                  # Claude 동작 지침
├── README.md                  # 이 파일
├── topics/                    # 기술별 지식 베이스 (자동 누적)
│   ├── golang/
│   │   ├── concepts.md        # 핵심 개념 정리
│   │   └── questions.md       # 면접 예상 질문
│   ├── mysql/
│   ├── redis/
│   ├── kubernetes/
│   └── ...
│
├── resume/                    # ⛔ gitignore — 본인이 직접 생성
│   └── profile.md             # 본인 프로필 작성
│
└── jobs/                      # ⛔ gitignore — 본인이 직접 생성
    └── {회사명-포지션}/
        ├── job.md             # 공고 정보 + 갭 분석
        └── sessions/          # 면접 세션 기록
```

> `resume/` 와 `jobs/` 는 `.gitignore` 에 등록되어 있어 git에 올라가지 않는다.
> 본인의 이력서, 공고, 세션 기록은 로컬에만 저장된다.

---

## topics/ 기술 목록

| 폴더 | 다루는 내용 |
|------|------------|
| golang | goroutine, channel, interface, GC |
| java-kotlin | Spring Boot, JPA, 동시성 |
| mysql | 인덱스, 트랜잭션, 복제 |
| postgresql | MVCC, MySQL 차이점 |
| redis | 캐싱 전략, 분산락, pub/sub |
| mongodb | 도큐먼트 모델, 집계 파이프라인 |
| kafka-rabbitmq | 메시징 패턴, 컨슈머 그룹 |
| zookeeper | 분산 코디네이션, Watch 이벤트 |
| kubernetes | 배포 전략, HPA, 장애 대응 |
| distributed-systems | CAP 정리, 분산 트랜잭션 |
| system-design | 실시간 채팅, 대용량 트래픽 설계 |
| elasticsearch | 역인덱스, 검색 쿼리 |
| python-fastapi | asyncio, FastAPI 구조 |

topics/ 파일들은 면접 세션을 진행할수록 자동으로 내용이 누적된다.
