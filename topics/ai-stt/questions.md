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
video-ai-stt는 영상 파일을 업로드하면 자동으로 SRT 자막과 JSON 텍스트를 생성하는 Go 서버 애플리케이션입니다. 핵심 설계 원칙은 세 단계의 처리 파이프라인을 goroutine과 channel로 느슨하게 연결하는 것입니다. Watcher가 N초마다 지정 폴더를 폴링해 새 영상 파일을 감지하면 `videoCh` 채널로 파일 경로를 전송합니다. Extractor goroutine은 `videoCh`에서 파일을 받아 FFmpeg 명령어로 오디오를 FLAC 포맷, 모노, 16kHz 샘플레이트로 추출한 뒤 `audioCh`로 전달합니다. FLAC을 선택한 이유는 무손실 압축이라 Whisper 모델이 오디오 품질 손실 없이 입력을 받을 수 있고, Groq API가 지원하는 포맷이기 때문입니다. Groq Worker는 `audioCh`에서 오디오 파일을 받아 Whisper API를 호출하고, verbose_json 형식으로 세그먼트별 타임스탬프를 받아 SRT 자막 파일을 생성합니다. 이 파이프라인 구조의 핵심 장점은 각 단계의 처리 속도 차이를 channel 버퍼가 자연스럽게 흡수한다는 점입니다. FFmpeg 오디오 추출은 수 초 수준이지만 Groq API 호출은 네트워크 왕복이 포함되어 더 오래 걸리는데, channel이 중간 버퍼 역할을 하면서 Extractor와 Worker가 독립적으로 동작할 수 있습니다. 중복 처리 방지를 위해 `sync.Map` 기반 `ProcessedManager`를 구현해 파일별로 8단계 처리 상태를 추적합니다. 샵라이브에서 라이브 스트리밍 영상 처리 파이프라인을 다뤘던 경험이 이 설계의 기반이 됐고, 카테노이드에서 채팅 서버 무중단 마이그레이션을 진행하며 쌓은 Go 동시성 패턴 경험도 적극 활용했습니다.

**꼬리 질문 예시**:
- "왜 채널 두 개로 파이프라인을 만들었나요? 하나로 하면 안 되나요?"
- "파일 중복 처리 방지를 어떻게 구현했나요?"
- "서버가 재시작되면 처리 중이던 파일은 어떻게 되나요?"

---

## Groq Whisper API에서 temperature를 0으로 설정한 이유는?

**난이도**: 기초

**핵심 키워드**: temperature, greedy decoding, hallucination, deterministic

**모범 답변 방향**:
temperature는 언어 모델이 다음 토큰을 선택할 때 소프트맥스 확률 분포의 날카로움을 조절하는 파라미터입니다. 수식으로 표현하면 각 토큰의 로짓을 temperature 값으로 나눠서 확률을 재계산하는데, `temperature = 0`이 되면 가장 높은 로짓의 토큰이 확률 1.0에 수렴하여 매번 동일한 토큰이 선택되는 Greedy Decoding과 동일하게 동작합니다. 반면 `temperature > 0`이면 낮은 확률의 토큰도 선택될 가능성이 생겨 창의적이고 다양한 출력이 나오지만, 같은 오디오를 여러 번 처리했을 때 매번 다른 자막이 생성될 수 있습니다. 자막 생성의 본질적 목적은 화자가 실제로 발화한 내용을 최대한 정확하게 텍스트로 옮기는 것이기 때문에, 창의성보다 재현성과 결정론적 출력이 훨씬 중요합니다. temperature를 높이면 Hallucination — 실제로 말하지 않은 단어나 구문을 모델이 그럴듯하게 생성하는 현상 — 위험이 커집니다. 자막에 없는 단어가 삽입되면 고객이 재생할 때 실제 발화와 자막이 불일치하게 되어 신뢰성을 잃습니다. 또한 같은 영상을 재처리했을 때 결과가 달라지면 디버깅도 어려워집니다. video-ai-stt 프로젝트에서는 이 트레이드오프를 고려해 `temperature = 0`으로 고정했고, 덕분에 동일 입력에 대해 항상 일관된 자막을 생성할 수 있었습니다. 창의적인 텍스트 생성(소설, 마케팅 카피)이라면 temperature를 0.7~1.0 수준으로 높이는 것이 적합하지만, STT처럼 사실 기반 전사 작업에서는 반드시 0에 가깝게 설정해야 합니다.

**꼬리 질문 예시**:
- "temperature가 높으면 자막에서 어떤 문제가 생길 수 있나요?" → 없는 단어 생성, 같은 오디오를 다시 처리하면 다른 자막이 나올 수 있음
- "AvgLogProb이 낮다는 것은 어떤 의미인가요?" → 모델이 해당 구간의 텍스트 예측을 어려워했다는 뜻 → 음질 불량 또는 인식 불가능한 언어

---

## response_format으로 verbose_json을 선택한 이유와 timestamp_granularities의 차이는?

**난이도**: 기초

**핵심 키워드**: verbose_json, timestamp_granularities, segment, word, SRT 자막

**모범 답변 방향**:
Groq Whisper API는 `response_format` 파라미터로 응답 형식을 제어합니다. `json`은 전사된 텍스트만 반환하기 때문에 자막 생성에 필수적인 타임스탬프 정보가 전혀 없어 SRT 자막 파일을 만들 수 없습니다. `text`는 plain text만 반환하므로 마찬가지로 자막 생성에 사용할 수 없습니다. `verbose_json`을 선택하면 task, language, duration 같은 메타정보와 함께 segments 배열을 반환하는데, 각 segment에는 id, seek, start, end, text, tokens, avg_logprob, no_speech_prob, compression_ratio, temperature 필드가 포함됩니다. start/end 시간 정보가 있어야 SRT 형식으로 시퀀스 번호, 타임코드, 자막 텍스트를 조합할 수 있습니다. SRT는 `HH:MM:SS,mmm --> HH:MM:SS,mmm` 형식의 타임코드와 자막 텍스트를 빈 줄로 구분하는 표준 자막 포맷으로, 대부분의 영상 플레이어와 편집 도구가 지원합니다. `timestamp_granularities[]` 파라미터는 `verbose_json`과 함께 사용해야 동작하며 다른 format에서는 무시됩니다. `segment`는 문장/구 단위 타임스탬프를 반환해서 SRT 자막 생성에 직접 사용할 수 있고, `word`는 단어 단위 타임스탬프를 반환해서 특정 단어 강조, 자막 싱크 정밀 조정, 또는 후처리 분석에 활용할 수 있습니다. video-ai-stt 프로젝트에서는 segment 타임스탬프로 SRT 파일을 생성하고, word 타임스탬프는 향후 품질 분석이나 단어 레벨 편집 기능 추가를 대비해 JSON으로 함께 저장합니다. avg_logprob, no_speech_prob, compression_ratio 같은 품질 지표도 verbose_json에서만 반환되기 때문에, 저품질 세그먼트 필터링을 위해서도 verbose_json이 필수 선택이었습니다.

**꼬리 질문 예시**:
- "SRT 포맷이란 무엇인가요?" → 시퀀스번호 / 시작→끝 타임코드 / 자막 텍스트 / 빈 줄로 구성된 자막 표준 포맷
- "segment와 word를 동시에 요청하면 API 비용이 늘어나나요?" → 동일 API 호출 내에서 처리되므로 추가 비용 없음

---

## Whisper STT의 동작 원리를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Log-Mel Spectrogram, Encoder-Decoder Transformer, Beam Search, Autoregressive, 특수 토큰

**모범 답변 방향**:
Whisper는 OpenAI가 공개한 Encoder-Decoder Transformer 기반의 STT 모델입니다. 처리 흐름은 크게 전처리, 인코딩, 디코딩 세 단계로 나뉩니다. 먼저 입력 오디오를 25ms 윈도우, 10ms 스트라이드로 STFT(Short-Time Fourier Transform)를 적용한 뒤 멜 필터뱅크를 통해 로그-멜 스펙트로그램으로 변환합니다. 이 2D 표현은 시간 축과 주파수 축으로 구성되며, 멜 스케일이 인간 청각 특성을 반영해 음성 인식에 최적화된 입력을 만들어냅니다. Encoder는 두 개의 1D Conv 레이어와 GELU 활성화 함수로 스펙트로그램의 초기 특징을 추출한 뒤 여러 개의 Transformer 블록을 통과시켜 고차원 컨텍스트 벡터로 압축합니다. Decoder는 Encoder 출력을 Cross-Attention으로 참조하면서 이전에 생성한 토큰들을 Self-Attention으로 처리하여 다음 토큰의 확률 분포를 예측하는 Autoregressive 방식입니다. 디코딩 과정에서 Beam Search를 사용하면 여러 후보 시퀀스를 병렬로 유지하며 탐색해 전체적으로 가장 높은 확률의 시퀀스를 선택합니다. `<|transcribe|>`, `<|translate|>`, `<|ko|>` 같은 특수 토큰으로 태스크(전사 vs 번역)와 언어를 제어하기 때문에 단일 모델로 99개 언어 전사와 영어 번역을 처리할 수 있습니다. 학습 측면에서 CTC 손실 대신 standard negative log-likelihood로 학습했고, 인터넷에서 수집한 680K 시간 규모의 오디오-텍스트 쌍 데이터로 약지도 학습(Weak Supervision)을 진행해 다양한 마이크 환경, 배경 소음, 억양에 강건합니다. video-ai-stt에서 FLAC 16kHz 모노로 오디오를 추출한 것도 Whisper의 최적 입력 조건을 맞추기 위해서입니다.

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

goroutine과 channel로 파이프라인을 구성한 이유는 Watcher(파일 감지) → Extractor(FFmpeg 오디오 추출) → Groq Worker(API 호출) 세 단계의 처리 속도가 명확히 다르기 때문입니다. Watcher는 폴더 폴링이라 수 밀리초 단위로 빠르고, FFmpeg 오디오 추출은 수 초 수준이지만 Groq API 호출은 네트워크 왕복과 모델 추론 시간이 포함되어 훨씬 느립니다. 이런 속도 불균형을 공유 큐나 뮤텍스 기반 구조로 해결하면 코드가 복잡해지고 레이스 컨디션 위험이 생깁니다. Go의 channel은 이 속도 차이를 자연스럽게 흡수하면서 각 단계를 독립적으로 동작시킵니다. Go 설계 철학인 "Do not communicate by sharing memory; share memory by communicating"과도 일치하는 선택입니다. 카테노이드에서 채팅 서버 무중단 마이그레이션을 진행할 때도 goroutine 기반 파이프라인으로 메시지 처리를 구성했고, 이 경험이 video-ai-stt 설계에 직접 적용됐습니다. 파일 처리 상태 추적에는 `sync.Map`을 선택했는데, 여러 goroutine이 동시에 같은 파일 상태를 읽고 쓰는 환경에서 `map + RWMutex` 조합보다 읽기 비중이 높은 패턴에서 성능이 우수하기 때문입니다. Key는 파일 경로, Value는 8단계 처리 상태(감지, 오디오 추출, API 호출 등)로 중복 처리를 방지합니다. Graceful Shutdown은 `context.WithCancel`로 최상위 컨텍스트를 만들어 SIGINT/SIGTERM 수신 시 cancel을 호출하면 각 goroutine의 `<-ctx.Done()` 구문이 감지해 현재 처리 중인 작업을 완료한 뒤 종료하도록 구현했습니다. `sync.WaitGroup`으로 모든 goroutine이 종료될 때까지 메인 프로세스가 기다립니다.

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
Whisper의 verbose_json 응답에는 세그먼트별 품질을 자동으로 판단할 수 있는 세 가지 필드가 있습니다. 이 필드들을 활용하면 수동 검수 없이도 저품질 세그먼트를 필터링해 자막 전체 품질을 높일 수 있습니다. AvgLogProb(평균 로그 확률)은 모델이 해당 구간의 각 토큰을 예측할 때의 로그 확률 평균값으로, 값이 클수록(0에 가까울수록) 모델이 높은 확신으로 예측했다는 의미입니다. 일반적으로 `-0.2 이상`이면 양호한 품질로 보고, `-1.0 이하`이면 음질이 불량하거나 모델이 인식하기 어려운 언어/소음이 포함된 구간으로 판단합니다. NoSpeechProb(무음 확률)은 해당 세그먼트에 실제 발화 대신 배경음, 침묵, 또는 비언어 소리가 있을 확률입니다. `0.6 이상`이면 배경음 구간으로 판단해서 자막에서 제외하거나 `[배경음]` 같은 표시로 대체할 수 있습니다. CompressionRatio(압축 비율)는 생성된 텍스트를 zlib 알고리즘으로 압축했을 때의 원본 대비 비율입니다. 텍스트에 반복이 많을수록 압축률이 높아지는 원리를 이용하는데, `2.4 초과`이면 모델이 "라라라라…" 같은 식으로 동일 토큰을 반복 생성하는 Hallucination이 발생했다는 신호입니다. video-ai-stt에서는 verbose_json으로 이 세 필드를 JSON 파일로 보존하고 있지만 현재 필터링 로직은 미구현 상태입니다. 개선 방향으로는 AvgLogProb < -0.8이거나 NoSpeechProb > 0.6이거나 CompressionRatio > 2.4인 세그먼트를 자동으로 경고 표시하는 후처리 레이어를 추가할 수 있습니다. 샵라이브에서 라이브 영상 품질 모니터링을 담당했던 경험이 이런 품질 지표 설계 아이디어의 배경이 됐습니다.

**꼬리 질문 예시**:
- "hallucination을 줄이는 다른 방법은?" → language 파라미터로 명시적 언어 지정, prompt로 전문 용어 미리 제공
- "이 프로젝트에서 현재 이 필드들을 활용하고 있나요?" → JSON 파일로 보존하지만 필터링 로직은 미구현 → 개선 포인트로 제시 가능

> 출처: [Groq Speech-to-Text Docs](https://console.groq.com/docs/speech-to-text) | [Whisper 구조](https://velog.io/@judy_choi/Whisper-Transformer-base-STT-Model)

---

## RAG vs Fine-tuning 선택 기준과 RAG 아키텍처

**난이도**: 중급

**핵심 키워드**: RAG, Fine-tuning, Vector DB, Embedding, Retriever, Chunking, pgvector

**모범 답변 방향**:
LLM 기반 챗봇 구축 시 Fine-tuning과 RAG의 선택 기준은 크게 세 가지입니다. 첫째, "지식이 얼마나 자주 바뀌는가"입니다. Fine-tuning은 모델 가중치를 새 데이터로 재훈련해서 특정 도메인 용어, 응답 스타일, 도메인 특화 지식을 모델 파라미터에 내재화하는 방식입니다. 지식이 변경될 때마다 GPU 재훈련이 필요하고 비용이 매우 크기 때문에, 변경 빈도가 낮고 응답 스타일 자체를 고정해야 하거나 민감 데이터라 외부 API로 전송할 수 없는 온프레미스 환경에서 유리합니다. 반면 RAG는 모델 가중치를 전혀 건드리지 않고 외부 지식 베이스에서 관련 문서 청크를 검색해 LLM 컨텍스트에 주입하는 방식입니다. 새 지식이 추가되면 문서를 벡터로 re-embedding해서 Vector DB에 저장하면 되기 때문에, GPU 재훈련 없이 훨씬 낮은 비용으로 지식을 업데이트할 수 있습니다. 둘째, "출처 추적이 필요한가"입니다. Fine-tuning은 지식이 가중치에 녹아들기 때문에 특정 답변의 출처를 사후에 추적할 수 없습니다. RAG는 검색된 청크를 직접 참조하므로 "이 답변은 문서 A, 3페이지를 참조했습니다"처럼 출처를 제시할 수 있어 법률, 의료, 금융 AICC처럼 근거 명시가 필수인 도메인에서 결정적 장점이 됩니다. 셋째, "비용 제약"입니다. video-ai-stt에서 Whisper로 추출한 자막 텍스트를 Chunking → Embedding → Vector DB 저장 파이프라인으로 구성하면 곧바로 RAG 지식 베이스로 활용할 수 있습니다.

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
Hallucination이란 LLM이 학습 데이터에 없거나 부정확한 내용을 마치 사실인 것처럼 그럴듯하게 생성하는 현상입니다. LLM은 다음 토큰을 확률적으로 예측하는 모델이기 때문에, 컨텍스트에 충분한 근거 정보가 없으면 학습 데이터에서 패턴을 유추해 없는 내용을 생성할 수 있습니다. Hallucination 감소 방법은 RAG 아키텍처 관점과 프롬프트 엔지니어링 관점으로 나뉩니다. RAG 관점에서는 먼저 청크 크기와 오버랩 설정이 중요합니다. 청크가 너무 작으면 문맥이 잘려 관련 없는 청크가 검색되고, 너무 크면 노이즈 정보가 컨텍스트에 섞여 오히려 Hallucination을 유발합니다. 도메인에 따라 128~512 토큰이 일반적이고 오버랩은 10~20% 수준이 적합합니다. Re-ranking은 Vector DB에서 유사도 기반으로 검색된 후보 청크들을 LLM 호출 전에 Cross-Encoder 계열 별도 모델이 관련도를 재점수화해 상위 K개만 최종 컨텍스트에 전달하는 방식입니다. 검색 품질이 낮으면 노이즈가 컨텍스트에 섞여 Hallucination이 증가하는 트레이드오프가 있으므로 Retrieval 품질 관리가 핵심입니다. video-ai-stt에서 Whisper로 추출한 자막 텍스트를 임베딩해 RAG 지식 베이스로 구성하면 이 파이프라인 전체를 직접 적용할 수 있습니다.

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

**면접 세션 피드백 (2026-04-21 4회차)**:
- 잘한 점: 청크 크기 트레이드오프(너무 작으면 맥락 단절, 너무 크면 노이즈) 설명. Temperature/CoT/출처 제한 커버. Re-ranking 개념 인지.
- 보완:
  - **Hallucination 정의 3회 연속 미제시** — "Hallucination이란 LLM이 학습 데이터에 없거나 부정확한 내용을 그럴듯하게 생성하는 현상입니다"를 첫 문장으로 암기
  - **Re-ranking 메커니즘 오류**: "LLM과 프롬프트로 추리는 것"이 아님. LLM 호출 **전에** Cross-Encoder 계열 별도 모델이 후보 청크 관련도를 재점수화 → 상위 K개만 컨텍스트에 전달
  - **이력서 연결 없음**: video-ai-stt Whisper 자막 텍스트 → 임베딩 → RAG 지식베이스 구성 가능으로 연결

---

## STT 품질 지표와 고객 커뮤니케이션 (채널톡 AX팀 연결)

**난이도**: 기초

**핵심 키워드**: CER(Character Error Rate), 최소 검수 구조, 90%/10% 프레임, 고객 커뮤니케이션, AX팀

**모범 답변 방향**:
STT 모델은 일반적인 발화에서는 높은 정확도를 보이지만, 도메인 특화 전문 용어, 영어 혼용 표현, 수학·과학 용어처럼 학습 데이터에 희소한 패턴에서는 CER(Character Error Rate)이 급격히 높아져 오타가 발생합니다. 이를 완전 자동화로 해결하려 하면 품질 보장이 어렵고, 모든 자막을 사람이 검수하면 자동화의 의미가 없어집니다. 실용적인 해법은 "AI가 확신하지 못하는 구간만 사람이 확인한다"는 최소 검수 구조입니다. 구체적으로는 Whisper의 verbose_json에서 세그먼트별 avg_logprob이 임계값 이하이거나 no_speech_prob이 임계값 이상인 구간을 SRT 자막 편집 화면에서 빨간색으로 강조 표시해, 편집자가 전체를 보는 대신 마킹된 부분만 집중해서 검수하도록 UX를 설계할 수 있습니다. 이 방식의 핵심 가치는 검수 시간을 전수 검토 대비 70~80% 수준으로 줄이면서도 품질을 보장한다는 점입니다. 고객사와 STT 솔루션 도입을 논의할 때 "100% 자동화"라는 기대치를 처음부터 조율하는 것이 중요합니다. "90% 자동화 + 10% 확인 검수" 프레임으로 커뮤니케이션하면 도입 초기 기대치 과잉으로 인한 실망을 예방하고, 실제로 달성 가능한 가치를 명확히 제시할 수 있습니다. 채널톡 AX팀에서 고객사에 AI 솔루션을 도입할 때도 같은 프레임이 적용 가능하며, CER 임계값 자체는 고객사 편집 담당자와 샘플 데이터를 함께 보면서 조율해 현장 수용도를 높이는 것이 효과적입니다. 이 경험을 통해 AI 솔루션의 품질과 고객 기대치를 맞추는 커뮤니케이션 방식을 체득했습니다.

**꼬리 질문 예시**:
- "이 경험이 채널톡 AX팀 업무와 어떻게 연결되나요?" → AI 솔루션 도입 초기 기대치 조율, 90%/10% 프레임으로 완벽함 대신 실용적 가치 제시
- "CER 임계값을 어떻게 결정했나요?" → 고객사 편집 담당자와 샘플 데이터 함께 보며 조율

**면접 세션 피드백 (2026-04-17 3회차)**:
- 잘한 점: CER 지표 + SRT 빨간색 표시 + 최소 검수 흐름 명확. 꼬리 답변에서 "90%/10%" 프레임으로 AX팀 연결.
- 보완:
  - **비즈니스 수치 없음**: "검수 시간이 X분 → Y분" 같은 수치 한 문장 추가 필요
  - **고객 커뮤니케이션 구체성 부족**: "CER 임계값을 고객사 담당자와 직접 조율했다" 같은 구체 장면 필요
  - **결론 문장 없음**: "이 경험으로 서비스 도입에 성공했고, AX팀에서도 같은 방식으로 적용 가능합니다" 마무리 필요
