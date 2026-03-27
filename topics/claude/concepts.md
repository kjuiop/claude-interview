# Claude — 핵심 개념 정리

## 1. 모델 종류 (2025 기준)

| 모델 | 특징 | 컨텍스트 | 적합한 용도 |
|------|------|----------|------------|
| Claude Haiku 4.5 | 가장 빠르고 저렴 | 200k | 요약, 데이터 추출, 간단한 작업 |
| Claude Sonnet 4.6 | 성능/비용 균형 (기본 추천) | 1M | 대부분의 실무 작업, 코딩 |
| Claude Opus 4.6 | 가장 강력 | 1M | 복잡한 추론, 멀티스텝 에이전트 |

- 모델 선택 기준: 빠른 응답 → Haiku, 일반 작업 → Sonnet, 복잡한 추론/코딩 → Opus
- 세 모델 모두 텍스트 + 이미지 멀티모달 지원

---

## 2. Claude Code

터미널/IDE에서 동작하는 **자율형 에이전트 코딩 도구**.
단순 코드 생성이 아니라 파일 읽기·편집·명령 실행·도구 통합까지 수행한다.

### 주요 기능
- **Deep Codebase Awareness** — 프로젝트 전체를 인덱싱해 파일 간 연관성 파악
- **Plan Mode** — 실행 전 단계별 계획을 작성하고 사용자 승인 대기
- **Subagent** — 특정 작업을 전문화된 서브 에이전트에게 위임
- **MCP 통합** — 외부 서비스(Slack, DB, GitHub 등)와 연결
- **Background Agent** — 독립적인 서브 작업을 병렬로 실행

### CLAUDE.md
- Claude Code가 세션 시작 시 **자동으로 로드**하는 프로젝트 지침 파일
- 반복 설명 없이 프로젝트 컨텍스트를 영구적으로 제공
- 위치별 우선순위: `~/.claude/CLAUDE.md` (전역) → 프로젝트 루트 → 서브디렉토리
- 포함할 내용: 기술 스택, 코딩 규칙, 주요 명령어, 파일 구조, 작동 방식

---

## 3. Claude API

### Context Window
- 한 번의 요청에서 처리할 수 있는 최대 토큰 수
- Sonnet 4.6 / Opus 4.6: **1M 토큰** (약 75만 단어)
- 대규모 코드베이스, 긴 문서 전체를 한 번에 처리 가능

### Tool Use (Function Calling)
- Claude가 외부 함수/API를 호출할 수 있는 기능
- 흐름: 사용자 메시지 → Claude가 tool 호출 결정 → 앱이 실행 → 결과를 Claude에 전달 → 최종 응답
- Claude 4.x는 **병렬 도구 호출** 지원 — 여러 도구를 동시에 실행

### Multimodal
- 텍스트 + 이미지 + PDF 입력 지원
- 단일 요청당 최대 600개 이미지 또는 PDF 페이지 처리

---

## 4. MCP (Model Context Protocol)

AI 도구 통합을 위한 **오픈 소스 표준 프로토콜**.
USB-C처럼 AI 앱이 다양한 외부 서비스와 표준화된 방식으로 연결할 수 있게 한다.

### 핵심 구성 요소
- **Tools** — 실행 가능한 액션 (예: DB 쿼리, 파일 읽기)
- **Resources** — 데이터 소스 접근 (예: API, 파일 시스템)
- **Prompts** — 재사용 가능한 프롬프트 템플릿

### 활용 사례
- Slack 메시지 전송/읽기
- GitHub 이슈/PR 관리
- DB 직접 쿼리
- 웹 검색

---

## 5. Agent / Subagent

### Subagent란
특정 유형의 작업을 전담하는 **특화된 에이전트**.
- 자체 컨텍스트 윈도우에서 독립적으로 실행
- 사용 가능한 도구 제한 가능
- 작업 완료 후 결과를 메인 에이전트에 반환

### 사용 이점
- **컨텍스트 보호** — 탐색/리서치 작업이 메인 대화 컨텍스트를 오염시키지 않음
- **병렬 실행** — 독립적인 여러 작업을 동시에 처리
- **재사용** — 프로젝트 간 에이전트 구성 재사용

### Foreground vs Background
- **Foreground**: 결과가 다음 단계에 필요한 경우 (순차 처리)
- **Background**: 독립적인 작업을 병렬로 실행할 경우

---

## 6. Prompt Engineering (Claude 최적화)

### 핵심 기법
1. **역할 지정 (Role Prompting)** — 시스템 프롬프트에 구체적인 역할 부여
2. **프롬프트 체이닝** — 복잡한 작업을 하위 작업으로 분해, XML 태그로 출력 전달
3. **Few-shot Learning** — 3~5개 예시로 원하는 출력 형식 명시
4. **Extended Thinking** — 복잡한 문제를 단계적으로 처리 (추론 강화)
5. **병렬 도구 호출** — 독립적인 도구 호출은 한 번에 묶어서 요청

### Claude에 최적화된 팁
- 긴 문서는 **앞에**, 질문은 **뒤에** 배치
- XML 태그로 입력 구조화 (`<document>`, `<question>`)
- 모호한 지시보다 **구체적이고 제약 조건이 명확한** 지시가 효과적
- "하지 마라"보다 "이렇게 해라" 형태가 효과적

---

## 7. Claude vs 타 LLM 비교

| 항목 | Claude | ChatGPT (GPT-4) |
|------|--------|-----------------|
| 컨텍스트 윈도우 | 1M 토큰 | 128k 토큰 |
| 코딩 능력 | 멀티파일 추론, 디버깅 강점 | 빠른 코드 생성 |
| 한국어 처리 | 우수 | 보통 |
| 안전성/윤리 | Constitutional AI 기반 | RLHF 기반 |
| 멀티모달 | 텍스트 + 이미지 + PDF | 텍스트 + 이미지 + 오디오 + 비디오 |
| 가격 (Sonnet급) | $3/$15 per 1M tokens | 비슷한 수준 |

---

## 참고 링크
- [Claude 모델 개요 (Anthropic 공식)](https://docs.anthropic.com/ko/docs/about-claude/models/overview)
- [Claude Code 공식 문서](https://code.claude.com/docs/ko/overview)
- [Prompt Engineering 가이드](https://platform.claude.com/docs/ko/build-with-claude/prompt-engineering/overview)
- [MCP 공식 문서](https://modelcontextprotocol.io/docs/getting-started/intro)
