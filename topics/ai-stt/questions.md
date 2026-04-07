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
> *"영상 파일을 업로드하면 자동으로 자막을 생성하는 Go 서버 애플리케이션입니다. 폴더 감시 → 오디오 추출 → STT API 호출 세 단계를 각각 goroutine + channel로 연결한 파이프라인 구조로 설계했습니다."*

- 구조: Watcher → Extractor → Groq의 3단계 파이프라인
- Watcher: N초마다 폴더 폴링, 새 영상 파일 감지 → `videoCh` 전송
- Extractor: FFmpeg으로 영상에서 오디오 추출(FLAC, 모노 16kHz) → `audioCh` 전송
- Groq: Whisper API 호출 → JSON + SRT 자막 파일 생성
- 상태 관리: `sync.Map` 기반 `ProcessedManager`로 8단계 처리 상태 추적, 중복 처리 방지

**꼬리 질문 예시**:
- "왜 채널 두 개로 파이프라인을 만들었나요? 하나로 하면 안 되나요?"
- "파일 중복 처리 방지를 어떻게 구현했나요?"
- "서버가 재시작되면 처리 중이던 파일은 어떻게 되나요?"

---

## Groq Whisper API에서 temperature를 0으로 설정한 이유는?

**난이도**: 기초

**핵심 키워드**: temperature, greedy decoding, hallucination, deterministic

**모범 답변 방향**:
- temperature는 다음 토큰 선택 시 확률 분포의 다양성을 조절하는 파라미터
- `temperature = 0` → Greedy Decoding: 매번 가장 높은 확률의 토큰만 선택 → 결정론적 출력
- `temperature > 0` → Sampling: 확률에 따라 다양한 토큰 선택 → 창의적이지만 불일치 발생 가능
- 자막 생성은 음성을 **정확히 전사**하는 것이 목적 → 창의성 불필요, 재현성이 중요
- 높은 temperature는 hallucination(없는 단어 생성) 위험이 있어 자막에 부적합

**꼬리 질문 예시**:
- "temperature가 높으면 자막에서 어떤 문제가 생길 수 있나요?" → 없는 단어 생성, 같은 오디오를 다시 처리하면 다른 자막이 나올 수 있음
- "AvgLogProb이 낮다는 것은 어떤 의미인가요?" → 모델이 해당 구간의 텍스트 예측을 어려워했다는 뜻 → 음질 불량 또는 인식 불가능한 언어

---

## response_format으로 verbose_json을 선택한 이유와 timestamp_granularities의 차이는?

**난이도**: 기초

**핵심 키워드**: verbose_json, timestamp_granularities, segment, word, SRT 자막

**모범 답변 방향**:
- `json`: 텍스트만 반환 → 자막 생성 불가 (시간 정보 없음)
- `verbose_json`: task, language, duration + **segments**(각 구간의 start/end 시간, avg_logprob 등) 반환 → SRT 자막 생성 가능
- `timestamp_granularities[]`:
  - `segment`: 문장/구 단위 시작·끝 시간 → **SRT 자막 직접 사용**
  - `word`: 단어 단위 시작·끝 시간 → 정밀 편집, 특정 단어 강조에 활용
  - `verbose_json`과 함께 사용해야 동작 (다른 format이면 무시)
- 이 프로젝트에서는 segment로 SRT 파일 생성, word는 JSON으로 보존

**꼬리 질문 예시**:
- "SRT 포맷이란 무엇인가요?" → 시퀀스번호 / 시작→끝 타임코드 / 자막 텍스트 / 빈 줄로 구성된 자막 표준 포맷
- "segment와 word를 동시에 요청하면 API 비용이 늘어나나요?" → 동일 API 호출 내에서 처리되므로 추가 비용 없음

---

## Whisper STT의 동작 원리를 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Log-Mel Spectrogram, Encoder-Decoder Transformer, Beam Search, Autoregressive, 특수 토큰

**모범 답변 방향**:
1. **전처리**: 오디오 → 로그-멜 스펙트로그램 (시간×주파수 2D 표현, 인간 청각 특성 반영)
2. **Encoder**: CNN + Transformer로 스펙트로그램 → 고차원 벡터 (음성 의미론적 특징 추출)
3. **Decoder**: Transformer가 Encoder 출력 + 이전 토큰을 보고 다음 토큰 확률 분포 예측 (Autoregressive)
4. **Beam Search**: 여러 후보 시퀀스를 동시 탐색 → 최고 확률 시퀀스 선택
5. **특수 토큰**: `<|transcribe|>`, `<|ko|>` 등으로 태스크/언어 제어 → 다국어 단일 모델 가능

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

**Channel 파이프라인 선택 이유:**
- Watcher(감지) → Extractor(FFmpeg) → Groq(API) 단계가 명확히 분리됨
- 각 단계의 처리 속도가 다름 (FFmpeg는 빠름, Groq API는 느림) → channel로 자연스럽게 속도 차이 흡수
- goroutine + channel = Go의 "Do not communicate by sharing memory; share memory by communicating" 원칙

**sync.Map 선택 이유:**
- 파일 처리 상태를 Key(경로):Value(단계) 로 추적
- 여러 goroutine이 동시에 읽고 쓰는 환경 → `sync.Map`이 `map + RWMutex`보다 읽기 많은 패턴에서 성능 우위

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
- **AvgLogProb** (평균 로그 확률): 모델이 해당 구간 텍스트를 얼마나 자신 있게 예측했는지
  - `-0.2 이상` → 양호
  - `-1.0 이하` → 음질 불량 or 알 수 없는 언어 → 자막 품질 낮음 경고
- **NoSpeechProb** (무음 확률): `0.6 이상`이면 실제 발화 없는 배경음 구간 → 자막에서 제외 가능
- **CompressionRatio** (압축 비율): `2.4 초과`이면 모델이 같은 텍스트를 반복 생성 (hallucination 의심)
- **실무 활용**: 이 필드들로 저품질 세그먼트를 자동 필터링 → 자막 후처리 품질 향상

**꼬리 질문 예시**:
- "hallucination을 줄이는 다른 방법은?" → language 파라미터로 명시적 언어 지정, prompt로 전문 용어 미리 제공
- "이 프로젝트에서 현재 이 필드들을 활용하고 있나요?" → JSON 파일로 보존하지만 필터링 로직은 미구현 → 개선 포인트로 제시 가능

> 출처: [Groq Speech-to-Text Docs](https://console.groq.com/docs/speech-to-text) | [Whisper 구조](https://velog.io/@judy_choi/Whisper-Transformer-base-STT-Model)
