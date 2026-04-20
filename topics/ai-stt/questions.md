---
tags: [ai-stt, whisper, groq, golang, pipeline, interview-questions]
related: [golang, ai-stt/concepts]
---

# AI STT (video-ai-stt 프로젝트) — 면접 예상 질문

→ [[home]] | 개념: [[topics/ai-stt/concepts]]

---

## video-ai-stt 프로젝트를 소개해주세요.

**난이도**: 기초

**핵심 키워드**: Pipeline Pattern, Channel, Goroutine, FFmpeg, Groq Whisper API, SRT

**모범 답변 방향**:
영상 파일을 업로드하면 자동으로 자막을 생성하는 Go 서버 애플리케이션입니다. 폴더 감시 → 오디오 추출 → STT API 호출 세 단계를 각각 goroutine과 channel로 연결한 파이프라인 구조로 설계했습니다. Watcher가 N초마다 폴더를 폴링해서 새 영상 파일을 감지하면 `videoCh`로 전송합니다. Extractor는 FFmpeg으로 영상에서 오디오를 FLAC, 모노 16kHz 형식으로 추출해 `audioCh`로 전송합니다. Groq Worker는 Whisper API를 호출해 JSON 텍스트와 SRT 자막 파일을 생성합니다. 각 단계의 처리 속도가 다르기 때문에 channel이 자연스럽게 버퍼 역할을 하면서 파이프라인이 흐릅니다. 중복 처리 방지를 위해 `sync.Map` 기반 `ProcessedManager`로 파일별 8단계 처리 상태를 추적합니다.

**꼬리 질문 예시**:
- "왜 채널 두 개로 파이프라인을 만들었나요? 하나로 하면 안 되나요?"
- "파일 중복 처리 방지를 어떻게 구현했나요?"
- "서버가 재시작되면 처리 중이던 파일은 어떻게 되나요?"

---

## Groq Whisper API에서 temperature를 0으로 설정한 이유는?

**난이도**: 기초

**핵심 키워드**: temperature, greedy decoding, hallucination, deterministic

**모범 답변 방향**:
temperature는 언어 모델이 다음 토큰을 선택할 때 확률 분포의 다양성을 조절하는 파라미터입니다. `temperature = 0`으로 설정하면 Greedy Decoding이 적용되어 매번 가장 높은 확률의 토큰만 선택하기 때문에 같은 오디오를 처리하면 항상 동일한 결과가 나옵니다. `temperature > 0`이면 확률에 따라 다양한 토큰을 샘플링해서 창의적인 출력이 가능하지만, 같은 입력에 대해 매번 결과가 달라질 수 있습니다. 자막 생성의 목적은 음성을 정확하게 전사하는 것이라 창의성이 필요 없고 재현성이 중요합니다. temperature를 높이면 hallucination — 실제로 말하지 않은 단어를 모델이 생성하는 현상 — 위험이 높아져서 자막에는 부적합합니다. 따라서 `temperature = 0`으로 설정했습니다.

**꼬리 질문 예시**:
- "temperature가 높으면 자막에서 어떤 문제가 생길 수 있나요?" → 없는 단어 생성, 같은 오디오를 다시 처리하면 다른 자막이 나올 수 있음
- "AvgLogProb이 낮다는 것은 어떤 의미인가요?" → 모델이 해당 구간의 텍스트 예측을 어려워했다는 뜻 → 음질 불량 또는 인식 불가능한 언어

---

## response_format으로 verbose_json을 선택한 이유와 timestamp_granularities의 차이는?

**난이도**: 기초

**핵심 키워드**: verbose_json, timestamp_granularities, segment, word, SRT 자막

**모범 답변 방향**:
Groq Whisper API는 `response_format` 파라미터로 반환 형식을 제어합니다. `json`은 텍스트만 반환하기 때문에 자막 생성에 필요한 타임스탬프 정보가 없어 SRT 자막을 만들 수 없습니다. `verbose_json`은 task, language, duration 같은 메타정보와 함께 segments 배열을 반환하는데, 각 segment에 시작/끝 시간과 `avg_logprob` 같은 품질 지표가 포함됩니다. 이 시간 정보가 있어야 SRT 형식의 자막을 만들 수 있습니다. `timestamp_granularities[]` 파라미터로 시간 단위를 제어하는데, `segment`는 문장/구 단위 타임스탬프를 반환해서 SRT 자막 생성에 직접 사용하고, `word`는 단어 단위 타임스탬프를 반환해서 특정 단어 강조나 정밀 편집에 활용합니다. 이 파라미터는 `verbose_json`과 함께 사용해야 동작하고 다른 format에서는 무시됩니다. 이 프로젝트에서는 segment로 SRT 파일을 생성하고, word 타임스탬프는 향후 활용을 위해 JSON으로 보존합니다.

**꼬리 질문 예시**:
- "SRT 포맷이란 무엇인가요?" → 시퀀스번호 / 시작→끝 타임코드 / 자막 텍스트 / 빈 줄로 구성된 자막 표준 포맷
- "segment와 word를 동시에 요청하면 API 비용이 늘어나나요?" → 동일 API 호출 내에서 처리되므로 추가 비용 없음

---

## Whisper STT의 동작 원리를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Log-Mel Spectrogram, Encoder-Decoder Transformer, Beam Search, Autoregressive, 특수 토큰

**모범 답변 방향**:
Whisper는 Encoder-Decoder Transformer 기반의 STT 모델입니다. 먼저 입력 오디오를 로그-멜 스펙트로그램으로 변환합니다. 이는 시간과 주파수 축으로 구성된 2D 표현으로, 인간 청각 특성을 반영해 음성 인식에 적합한 입력을 만듭니다. Encoder는 CNN과 Transformer를 결합해서 스펙트로그램을 고차원 벡터로 압축하며 음성의 의미론적 특징을 추출합니다. Decoder는 Encoder 출력과 이전에 생성한 토큰을 함께 참조해서 다음 토큰의 확률 분포를 예측하는 Autoregressive 방식입니다. Beam Search는 여러 후보 시퀀스를 동시에 유지하며 탐색해서 전체적으로 가장 높은 확률의 시퀀스를 선택합니다. `<|transcribe|>`, `<|ko|>` 같은 특수 토큰으로 태스크와 언어를 제어하기 때문에 하나의 모델로 다국어를 처리할 수 있습니다. CTC 손실 대신 negative log-likelihood로 학습했고, 680K 시간 규모의 웹 데이터로 약지도 학습해서 다양한 환경과 억양에 강건합니다.

**핵심 차별점:**
- CTC 손실 사용 안 함 (Seq2Seq Transformer, negative log-likelihood로 학습)
- 680K 시간의 대규모 웹 데이터로 weak supervision 학습 → 다양한 환경/억양 강건성

**꼬리 질문 예시**:
- "temperature=0과 Beam Search는 어떻게 다른가요?" → temperature=0은 각 스텝에서 greedy하게 1개 선택, Beam Search는 여러 후보를 유지하며 전체 시퀀스 최적화
- "FLAC으로 오디오를 추출한 이유는?" → 무손실 압축, Groq API 지원 포맷, Whisper는 16kHz 모노가 최적 → 오디오 품질 보존

---

## video-ai-stt의 파이프라인 설계에서 goroutine과 channel을 선택한 이유는?

**난이도**: 중급

**핵심 키워드**: Pipeline Pattern, Channel, sync.Map, Fan-out, Graceful Shutdown, Context

**모범 답변 방향**:

goroutine과 channel로 파이프라인을 구성한 이유는 Watcher(파일 감지) → Extractor(FFmpeg) → Groq(API 호출) 세 단계의 처리 속도가 명확히 다르기 때문입니다. FFmpeg 오디오 추출은 빠르지만 Groq API 호출은 네트워크 왕복이 있어 느립니다. channel을 버퍼로 사용하면 이 속도 차이를 자연스럽게 흡수해서 각 단계가 독립적으로 처리할 수 있습니다. Go의 "Do not communicate by sharing memory; share memory by communicating" 철학과도 일치합니다. 공유 메모리 대신 channel로 데이터를 전달하면 race condition 위험이 줄어듭니다. 파일 처리 상태 추적에 `sync.Map`을 선택한 이유는 여러 goroutine이 동시에 읽고 쓰는 환경에서 `map + RWMutex` 조합보다 읽기 비중이 높은 패턴에서 성능이 우수하기 때문입니다. Key는 파일 경로, Value는 8단계 처리 상태로 중복 처리를 방지합니다.

**sync.Map 선택 요약:**

**Graceful Shutdown:**
```go
ctx, cancel := context.WithCancel(context.Background())
// SIGINT/SIGTERM → cancel() → 각 goroutine의 <-ctx.Done() 감지 → 현재 작업 완료 후 종료
wg.Wait() // 모든 goroutine 종료 확인
```

**꼬리 질문 예시**:
- "ProcessedManager를 sync.Map 대신 Redis로 구현한다면 어떤 장단점이 있나요?" → 분산 환경(서버 여러 대)에서는 Redis가 필요, 단일 서버면 sync.Map이 더 빠르고 단순
- "Groq API 호출 실패 시 재시도 로직은 어떻게 구현하시겠어요?" → Retry + Exponential Backoff, 최종 실패 시 Dead Letter 패턴으로 별도 큐/파일에 기록

---

## SegmentsSpec의 AvgLogProb, NoSpeechProb, CompressionRatio 필드를 어떻게 활용할 수 있나요?

**난이도**: 심화

**핵심 키워드**: AvgLogProb, NoSpeechProb, CompressionRatio, 자막 품질 필터링, hallucination

**모범 답변 방향**:
Whisper의 segments 응답에는 자막 품질을 판단할 수 있는 세 가지 필드가 있습니다. AvgLogProb(평균 로그 확률)은 모델이 해당 구간 텍스트를 얼마나 자신 있게 예측했는지를 나타냅니다. `-0.2 이상`이면 양호하고, `-1.0 이하`이면 음질이 불량하거나 인식할 수 없는 언어여서 자막 품질이 낮다는 신호입니다. NoSpeechProb(무음 확률)은 해당 구간에 실제 발화가 없을 확률인데, `0.6 이상`이면 배경음 구간으로 판단해서 자막에서 제외할 수 있습니다. CompressionRatio(압축 비율)는 생성된 텍스트의 압축 효율을 측정하는데, `2.4 초과`이면 모델이 같은 텍스트를 반복 생성하는 hallucination이 발생했다는 신호입니다. 실무에서는 이 세 필드를 조합해서 저품질 세그먼트를 자동으로 필터링하면 수동 검수 없이 자막 후처리 품질을 높일 수 있습니다.

**꼬리 질문 예시**:
- "hallucination을 줄이는 다른 방법은?" → language 파라미터로 명시적 언어 지정, prompt로 전문 용어 미리 제공
- "이 프로젝트에서 현재 이 필드들을 활용하고 있나요?" → JSON 파일로 보존하지만 필터링 로직은 미구현 → 개선 포인트로 제시 가능

> 출처: [Groq Speech-to-Text Docs](https://console.groq.com/docs/speech-to-text) | [Whisper 구조](https://velog.io/@judy_choi/Whisper-Transformer-base-STT-Model)

---

## RAG vs Fine-tuning 선택 기준과 RAG 아키텍처

**난이도**: 중급

**핵심 키워드**: RAG, Fine-tuning, Vector DB, Embedding, Retriever, Chunking, pgvector

**모범 답변 방향**:
LLM 기반 챗봇 구축 시 Fine-tuning과 RAG 선택 기준은 "지식이 얼마나 자주 바뀌는가"와 "비용"입니다. Fine-tuning은 모델 가중치를 새 데이터로 재훈련해 특정 도메인 용어나 응답 스타일을 내재화하는 방식입니다. GPU 훈련 비용이 크고 지식이 바뀌면 재훈련이 필요합니다. RAG는 모델 가중치를 건드리지 않고 외부 지식 베이스에서 관련 문서를 검색해 LLM 컨텍스트로 제공하는 방식입니다. 제품 매뉴얼·사내 FAQ처럼 자주 업데이트되는 지식 기반, 출처 추적이 필요한 경우(법률·의료·금융), 비용 제약이 있을 때 RAG가 적합합니다.

RAG 아키텍처 핵심 컴포넌트:
1. **Chunker**: 문서를 적절한 크기로 분할
2. **Embedding Model**: 청크를 벡터로 변환
3. **Vector DB**: 벡터 저장 + 유사도 검색 (Pinecone, pgvector, Chroma 등)
4. **Retriever**: 질문을 벡터화해 유사 청크 검색
5. **LLM**: 검색된 컨텍스트 + 질문 → 답변 생성

**꼬리 질문 예시**:
- "RAG에서 검색 품질이 낮으면 어떤 문제가 생기나요?" → 관련 없는 컨텍스트 → hallucination 증가, 답변 품질 저하
- "Chunking 크기는 어떻게 결정하나요?" → 너무 크면 무관한 정보 포함, 너무 작으면 컨텍스트 손실 → 도메인에 따라 128~512 토큰이 일반적

**면접 세션 피드백 (2026-04-16 2회차)**:
- 잘한 점: 해당 없음 (완전 모름)
- 보완: 인포뱅크 우대 사항에 직접 명시된 주제. AICC/챗봇 플랫폼 면접에서 반드시 나오는 질문. 위 모범 답변 암기 필수.

**면접 세션 피드백 (2026-04-20 1회차)**:
- 잘한 점: RAG vs Fine-tuning 본질적 차이 명확. 꼬리 질문에서 re-embedding vs GPU 재학습 비용 차이 정확히 짚음. 데이터 변경 빈도를 선택 기준으로 연결.
- 보완:
  - **Fine-tuning 유리한 케이스 구체화**: 응답 스타일/톤 고정, 도메인 전문 용어 내재화, 민감 데이터로 외부 API 전송 불가 시(온프레미스)
  - **RAG 파이프라인 구성 요소 미언급**: Vector DB 이름(Pinecone, pgvector, Chroma) + Chunker + Embedding Model + Retriever 흐름
  - **이력서 연결 없음**: video-ai-stt Whisper/Groq 경험 → "STT로 변환한 텍스트를 RAG 지식 베이스로 구성 가능"으로 연결 필요

**면접 세션 피드백 (2026-04-17 1회차)**:
- 잘한 점: RAG 파이프라인 전체 흐름 순서 설명. Fine-tuning vs RAG — GPU 비용 + 데이터 변경 시 재훈련 구분. 인포뱅크 AICC 고객사별 지식 베이스 적용 시나리오 연결. KNN/코사인 유사도/top-k 언급.
- 보완:
  - **"토큰으로 변경" 오개념**: 청킹(Chunking) ≠ 토큰화(Tokenization). Chunking은 문서를 적절한 크기로 분할하는 것.
  - **쿼리 임베딩 단계 누락**: 사용자 질문도 동일 Embedding 모델로 벡터화 → 쿼리 벡터 생성 → Vector DB에서 유사도 검색. "질문을 청크화"가 아님.
  - **RAG 선택 기준 "출처 추적" 미언급**: Fine-tuning은 출처 추적 불가. RAG는 검색 청크로 출처 제시 가능 — 법률/의료/금융 AICC에서 핵심
  - **STT 경험 연결 없음**: "Whisper로 변환한 텍스트를 RAG 지식 베이스로 구성 가능"으로 경험 연결 필요

---

## LLM Hallucination 감소 기법

**난이도**: 중급

**핵심 키워드**: Hallucination, RAG, 청크 크기, Re-ranking, Temperature, Chain-of-Thought, 출처 인용, 제약 지시문

**모범 답변 방향**:
Hallucination이란 LLM이 학습 데이터에 없거나 부정확한 내용을 그럴듯하게 생성하는 현상입니다. 감소 방법은 RAG 관점과 프롬프트 엔지니어링 관점 두 가지로 나뉩니다.

**RAG 관점:**
- 청크 크기·오버랩 설정: 너무 작으면 맥락이 잘리고, 너무 크면 노이즈가 섞여 오히려 환각 유발
- Re-ranking: 검색된 후보 청크 중 관련도 높은 것만 최종 컨텍스트에 포함
- Retrieval 품질이 낮으면 노이즈가 컨텍스트에 섞여 환각 증가 — 트레이드오프 명시

**프롬프트 엔지니어링 관점:**
1. **제약 지시문**: "제공된 문서에 없는 내용은 '해당 내용을 찾을 수 없습니다'라고 답하라"
2. **출처 인용 요구**: "모든 답변에는 참조한 문서 청크의 출처를 인용하라"
3. **Temperature 낮추기**: 0.0~0.3 수준 → 보수적인 토큰 선택 → 창의적 생성 억제
4. **Chain-of-Thought**: "단계적으로 추론하고 각 단계에서 근거를 제시하라" → 추론 오류 중간에 드러남

**이력서 연결**: video-ai-stt에서 Whisper로 추출한 자막 텍스트를 임베딩해 RAG 지식베이스 구성 가능

**꼬리 질문 예시**:
- "RAG로 관련 문서를 가져왔는데도 없는 내용을 지어낸다면 프롬프트로 어떻게 억제하나요?" → 제약 지시문 + 출처 인용
- "Temperature 0과 Greedy Decoding의 관계는?" → temperature=0이면 매 스텝에서 가장 높은 확률 토큰만 선택 = Greedy Decoding
- "Fine-tuning 대신 RAG를 쓰면 환각이 줄어드는 이유는?" → 컨텍스트에 정확한 정보를 주입하므로 모델이 지어낼 필요가 없어짐

**면접 세션 피드백 (2026-04-20 3회차)**:
- 잘한 점: RAG 외부 문서 주입 방향 정확. 꼬리 질문에서 출처 인용 + 출처 없으면 답변 제한 패턴 스스로 도달.
- 보완:
  - **Hallucination 정의 선제 제시** 미흡 — 첫 문장에 정의 필수
  - **RAG 구체 기법 미언급**: 청크 크기·오버랩, Re-ranking
  - **프롬프트 기법**: Temperature 낮추기(0.0~0.3), Chain-of-Thought 미언급
  - **이력서 연결 없음**: STT 프로젝트 → RAG 지식베이스 구성 가능 연결 필요

---

## STT 품질 지표와 고객 커뮤니케이션 (채널톡 AX팀 연결)

**난이도**: 기초

**핵심 키워드**: CER(Character Error Rate), 최소 검수 구조, 90%/10% 프레임, 고객 커뮤니케이션, AX팀

**모범 답변 방향**:
STT 모델은 도메인 특화 용어(영어·수학)에서 CER(Character Error Rate)이 높아 오타가 발생합니다. 완벽한 자동화 대신 "AI가 확신하지 못하는 부분만 사람이 확인한다"는 최소 검수 구조가 실용적입니다. SRT 자막 추출 시 CER 임계값 초과 단어를 빨간색으로 강조 표시해 편집자가 마킹된 부분만 확인하도록 합니다. 채널톡 AX팀에서는 "90% 자동화 + 10% 수동" 프레임으로 고객사 도입 저항을 낮출 수 있습니다.

**꼬리 질문 예시**:
- "이 경험이 채널톡 AX팀 업무와 어떻게 연결되나요?" → AI 솔루션 도입 초기 기대치 조율, 90%/10% 프레임으로 완벽함 대신 실용적 가치 제시
- "CER 임계값을 어떻게 결정했나요?" → 고객사 편집 담당자와 샘플 데이터 함께 보며 조율

**면접 세션 피드백 (2026-04-17 3회차)**:
- 잘한 점: CER 지표 + SRT 빨간색 표시 + 최소 검수 흐름 명확. 꼬리 답변에서 "90%/10%" 프레임으로 AX팀 연결.
- 보완:
  - **비즈니스 수치 없음**: "검수 시간이 X분 → Y분" 같은 수치 한 문장 추가 필요
  - **고객 커뮤니케이션 구체성 부족**: "CER 임계값을 고객사 담당자와 직접 조율했다" 같은 구체 장면 필요
  - **결론 문장 없음**: "이 경험으로 서비스 도입에 성공했고, AX팀에서도 같은 방식으로 적용 가능합니다" 마무리 필요
