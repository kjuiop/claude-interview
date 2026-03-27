---
tags: [claude, ai, llm, interview-questions]
related: [prompt-engineering, mcp, agent]
---

# Claude — 면접 예상 질문

→ [[home]] | 개념 정리: [[topics/claude/concepts]]

---

## 기초 개념

**Q. Claude와 ChatGPT의 주요 차이점은 무엇인가요?**
- 컨텍스트 윈도우: Claude 1M vs GPT-4 128k → 대규모 코드베이스/문서 처리에 Claude가 유리
- 안전성 접근: Claude는 Constitutional AI 기반, GPT는 RLHF 기반
- 멀티모달: GPT는 오디오/비디오까지 지원, Claude는 텍스트+이미지+PDF
- 코딩: Claude는 멀티파일 추론과 디버깅에 강점

---

**Q. Claude 모델을 선택할 때 어떤 기준으로 결정하나요?**
- Haiku: 빠른 응답이 필요한 단순 작업 (요약, 분류, 추출)
- Sonnet: 대부분의 실무 작업 — 기본값
- Opus: 복잡한 추론, 멀티스텝 에이전트, 정확도가 중요한 작업

---

**Q. Context Window가 크면 어떤 이점과 단점이 있나요?**
- 이점: 대규모 코드베이스 전체를 한 번에 분석, 긴 대화 히스토리 유지
- 단점: 응답 속도 저하, 비용 증가
- "Lost in the middle" 문제: 긴 컨텍스트 중간부 정보를 잘 못 찾는 경향 → 중요 정보는 앞/뒤 배치
- 관련 개념: [[topics/claude/concepts#3. Claude API]]

---

## Claude Code

**Q. CLAUDE.md 파일은 왜 사용하나요?**
- Claude Code가 세션 시작 시 자동 로드 → 매번 설명 없이 컨텍스트 유지
- 프로젝트 기술 스택, 코딩 규칙, 디렉토리 구조 담아두면 일관된 동작
- 전역(`~/.claude/CLAUDE.md`)과 프로젝트별로 나눠 관리 가능
- 참고: [[topics/claude/concepts#CLAUDE.md]]

---

**Q. Subagent를 언제 사용하나요?**
- 독립적인 리서치/탐색 작업 → 메인 컨텍스트를 오염시키지 않으려 할 때
- 병렬로 처리할 수 있는 독립적인 작업이 여러 개일 때
- Foreground: 결과가 다음 단계에 필요 / Background: 완전히 독립적인 작업
- 참고: [[topics/claude/concepts#5. Agent / Subagent]]

---

**Q. MCP(Model Context Protocol)란 무엇이고 어떻게 활용하나요?**
- AI 앱이 외부 서비스와 표준 방식으로 연결하는 오픈 프로토콜
- Tools(액션), Resources(데이터), Prompts(템플릿) 세 요소로 구성
- 활용: Slack 자동화, DB 쿼리, GitHub 연동, 웹 검색
- 비유: AI를 위한 USB-C
- 참고: [[topics/claude/concepts#4. MCP (Model Context Protocol)]]

---

## Prompt Engineering

**Q. Claude에서 효과적인 프롬프트 작성 방법은?**
- XML 태그로 입력 구조화: `<document>`, `<instruction>`, `<example>`
- 역할 먼저 지정: "당신은 시니어 백엔드 개발자입니다..."
- 긴 문서는 앞에, 질문은 뒤에 배치 (Lost in the middle 방지)
- 구체적인 제약 조건 명시: "300자 이내로", "JSON 형식으로"
- Few-shot: 원하는 출력 형식 예시 2~3개 제공

---

**Q. Tool Use(Function Calling)의 동작 흐름을 설명해주세요.**
1. 사용자가 메시지 전송 + 사용 가능한 도구 목록 제공
2. Claude가 도구 호출이 필요한지 판단
3. 앱이 도구를 실제 실행하고 결과 반환
4. Claude가 결과를 바탕으로 최종 응답 생성
- 병렬 호출: 독립적인 여러 도구는 한 번에 묶어서 요청 → 응답 속도 개선

---

## 활용 시나리오

**Q. 대규모 코드베이스 리뷰에 Claude를 어떻게 활용하나요?**
- 1M 컨텍스트를 활용해 전체 파일을 한 번에 로드
- CLAUDE.md에 코드 스타일 규칙, 아키텍처 원칙 정의
- Subagent로 각 모듈별 분석을 병렬 실행
- Plan Mode로 리팩터링 계획 먼저 수립 후 실행

---

**Q. Claude Code를 팀에서 사용할 때 주의할 점은?**
- CLAUDE.md를 공유 저장소에 커밋하여 팀 전체 컨텍스트 통일
- 민감한 정보(API 키, 개인 정보)는 gitignore 처리
- Permission Mode 설정으로 자동 실행 범위 제한
- 리뷰 없이 코드가 머지되지 않도록 Git 훅 설정
