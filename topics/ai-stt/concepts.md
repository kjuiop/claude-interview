---
tags: [ai-stt, whisper, groq, golang, pipeline, ffmpeg]
related: [golang, distributed-systems]
---

# AI STT (Speech-to-Text) 핵심 개념

→ [[home]] | 질문: [[topics/ai-stt/questions]]

---

## 1. Whisper STT 동작 원리

### 전체 흐름

```
오디오 파일
  ↓ 전처리
로그-멜 스펙트로그램 (Log-Mel Spectrogram)
  ↓ Encoder (CNN + Transformer)
고차원 벡터 표현 (음성의 의미론적 특징)
  ↓ Decoder (Transformer + Autoregressive)
텍스트 토큰 시퀀스
  ↓ Beam Search
최종 텍스트
```

### 단계별 설명

**1. 전처리 — 로그-멜 스펙트로그램**
- 오디오를 시간(x) × 주파수(y) × 강도(z)로 표현한 2D 이미지
- 인간 청각 특성에 맞게 멜 스케일 적용 (저주파 구간 더 세밀)
- Whisper: 30초 단위 윈도우로 처리, 16kHz 샘플레이트

**2. Encoder — 음성 인식**
- CNN 프론트엔드: 스펙트로그램의 로컬 패턴 추출
- Transformer Encoder: Self-Attention으로 음성의 장거리 의존성 학습
- 출력: 각 시간 프레임에 대한 고차원 벡터

**3. Decoder — 텍스트 생성 (Autoregressive)**
- 이전에 생성한 토큰 + Encoder 출력 → 다음 토큰 확률 분포 예측
- Cross-Attention으로 음성 정보 참조
- 특수 토큰으로 태스크/언어 제어:
  - `<|transcribe|>` : 전사 태스크
  - `<|translate|>` : 번역 태스크
  - `<|ko|>`, `<|en|>` : 언어 지정
  - `<|notimestamps|>` : 타임스탬프 없음

**4. Beam Search 디코딩**
- Temperature = 0 → Greedy Decoding (가장 높은 확률의 토큰만 선택) → 결정론적
- Temperature > 0 → Sampling (다양성 추가, 자막에는 부적합)
- Beam Size = 5 → 5개의 후보 시퀀스를 동시에 탐색, 최고 확률 선택

---

## 2. Groq API 파라미터 의미

Groq는 OpenAI Whisper 모델을 LPU(Language Processing Unit)로 가속해 제공하는 서비스.

### model
| 모델 | 특징 |
|---|---|
| `whisper-large-v3` | 최고 정확도, 다국어 + 번역 지원 |
| `whisper-large-v3-turbo` | 속도 우선, 번역 미지원 |
| `distil-whisper-large-v3-en` | 영어 전용, 경량화 버전 |

### temperature (0 사용 이유)
```
temperature = 0 → Greedy Decoding → 결정론적 출력
temperature = 1 → 창의적/다양한 출력
```
자막 생성은 음성을 정확히 전사하는 것이 목적 → **0 고정**이 맞음.
높으면 없는 단어를 만들어내거나(hallucination) 표현이 달라질 수 있음.

### response_format (verbose_json 사용 이유)
| 값 | 응답 내용 |
|---|---|
| `json` | `{text: "..."}`만 반환 |
| `text` | 텍스트 문자열만 반환 |
| `verbose_json` | task, language, duration, **segments(시간 정보)** 전체 반환 |

SRT 자막에는 각 구간의 시작/끝 시간이 필요 → `verbose_json` 필수.

### timestamp_granularities[] (word + segment 동시 사용 이유)
- `segment`: 문장/구 단위 시작·끝 시간 → SRT 자막 생성에 직접 사용
- `word`: 단어 단위 시작·끝 시간 → 정밀한 자막 편집, 특정 단어 하이라이트에 활용
- **verbose_json과 함께 사용 필수** (없으면 무시됨)

### Response 필드 해석

```go
type SegmentsSpec struct {
    AvgLogProb       float64 // 평균 로그 확률: 낮을수록 불확실 (보통 -0.2 이상이면 양호)
    NoSpeechProb     float64 // 0~1, 높을수록 무음 구간 → 필터링에 활용
    CompressionRatio float64 // 텍스트/원본 압축 비율: 매우 높으면 반복 생성 의심
    Temperature      float64 // 이 세그먼트에 실제 적용된 온도 (fallback 로직 반영)
    Tokens           []int64 // 텍스트를 토크나이저로 변환한 ID 목록
}
```

**품질 판단 기준:**
- `AvgLogProb < -1.0` → 해당 구간 음질 불량 또는 알 수 없는 언어
- `NoSpeechProb > 0.6` → 실제 발화 없는 배경음 구간
- `CompressionRatio > 2.4` → 모델이 같은 텍스트를 반복 생성하는 hallucination 의심

---

## 3. video-ai-stt 아키텍처

### Pipeline 구조

```
uploads/ 폴더
  ↓ polling (WatchInterval 주기)
[Watcher Goroutine]
  videoCh (chan *job.Job)
  ↓
[Extractor Goroutine]  ← FFmpeg: video → FLAC 오디오 추출
  audioCh (chan *job.Job)
  ↓
[Groq Goroutine]  ← Groq Whisper API 호출
  ↓
output/ 폴더
  ├── {filename}.json  (verbose_json 전체)
  └── {filename}.srt   (SRT 자막)
```

### 핵심 설계 결정

**1. Channel Pipeline 패턴**
- `videoCh`, `audioCh` 두 채널로 단계 연결
- 각 단계가 독립적 goroutine → 이전 단계 처리 중에 다음 단계 병렬 진행 가능

**2. sync.Map 기반 ProcessedManager**
- 파일 경로(key) → 처리 단계(1~8) 저장
- 서버 재시작 없이 이미 처리한 파일 중복 처리 방지
- `sync.Map`: 읽기 많고 쓰기 적은 패턴에 최적화된 Go 내장 자료구조

**3. FFmpeg FLAC 변환 이유**
- FLAC = 무손실 압축 포맷
- Groq API 지원 포맷: flac, mp3, mp4, mpeg, mpga, m4a, ogg, wav, webm
- 오디오만 추출 + 모노(1채널) 변환 → Whisper는 모노 16kHz 입력이 최적
- 샘플레이트(16kHz)를 맞춰주면 모델 내부 전처리 없이 바로 스펙트로그램 변환 가능

**4. Graceful Shutdown**
- `context.WithCancel` → SIGINT/SIGTERM 수신 시 cancel() 호출
- 각 goroutine이 `<-ctx.Done()` 감지 후 현재 처리 완료 후 종료
- `sync.WaitGroup`으로 모든 goroutine 종료 확인 후 프로세스 종료

**5. Fan-out (Groq 병렬 처리)**
```go
go func(jobs *job.Job) {
    defer wg.Done()
    g.requestSubtitle(jobs.GetAudioPath())
}(jobs)
```
여러 영상을 동시에 Groq API에 병렬 요청 → 처리량 향상

---

## 4. SRT 자막 포맷

```srt
1
00:00:00,000 --> 00:00:02,500
안녕하세요, 오늘은 Go 언어에 대해

2
00:00:02,500 --> 00:00:05,000
이야기하겠습니다.
```

- `Segments.ID + 1` → 1부터 시작하는 시퀀스 번호
- `Segments.Start/End` (float64, 초) → `HH:MM:SS,mmm` 포맷으로 변환
- `srtFormatTime()`: `seconds → 02d:02d:02d,03d`

> 출처: [Groq Speech-to-Text Docs](https://console.groq.com/docs/speech-to-text) | [Whisper 모델 구조](https://velog.io/@judy_choi/Whisper-Transformer-base-STT-Model)
