---
tags: [networking, http, https, tls, ssl, nginx, tomcat, 면접질문]
related: [networking/concepts]
---

# Networking — 면접 질문

→ [[home]] | 개념 정리: [[topics/networking/concepts]]

---

## HTTP와 HTTPS의 차이를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: TLS, 암호화, 443포트, 기밀성, 무결성, 인증

**모범 답변 방향**:
- HTTP는 평문 전송 → 도청·위변조 가능
- HTTPS = HTTP + TLS 암호화 레이어 (포트 443)
- HTTPS가 보장하는 세 가지: 기밀성(암호화), 무결성(위변조 감지), 인증(서버 신원 확인)
- TLS 핸드셰이크로 세션 키 교환 후 대칭 암호화로 통신

**꼬리 질문 예시**:
- "TLS 핸드셰이크 과정을 단계별로 설명해주세요"
- "공개키 암호화와 대칭키 암호화를 왜 함께 사용하나요?"

> 출처: https://aws.amazon.com/compare/the-difference-between-https-and-http/

---

## TLS Handshake 과정을 단계별로 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Client Hello, Server Hello, 인증서, Pre-Master Secret, Session Key, 대칭키

**모범 답변 방향**:
- Client Hello: 지원 암호화 방식 목록 + 클라이언트 랜덤값 전송
- Server Hello: 선택 암호화 방식 + 서버 랜덤값 + 서버 인증서 전송
- 클라이언트가 인증서의 공개키로 Pre-Master Secret 암호화 전송
- 양측이 클라이언트 랜덤 + 서버 랜덤 + Pre-Master Secret으로 Session Key(대칭키) 생성
- 이후 Session Key로 빠른 대칭 암호화 통신
- 공개키는 키 교환 단계에만 사용 → 이후는 빠른 대칭키로 처리

**꼬리 질문 예시**:
- "왜 공개키로 계속 통신하지 않고 대칭키로 전환하나요?"
- "TLS 1.2와 TLS 1.3의 차이는 무엇인가요?"
- "인증서는 어떻게 신뢰하나요? CA 체인이란?"

**자주 나오는 오답 패턴** (2026-04-01 세션):
- ❌ "클라이언트가 인증서와 공개키를 서버에게 보낸다" → ✅ 서버가 클라이언트에게 전송
- ❌ "서버가 대칭키로 검증한다" → ✅ 대칭키(세션 키)는 Handshake 마지막에 도출. 키 교환 단계에서는 클라이언트가 서버의 공개키로 암호화
- ❌ "클라이언트가 인증서를 인증 회사에서 직접 다운로드" → ✅ 서버가 Handshake 중 인증서를 전송. 클라이언트는 OS/브라우저에 사전 설치된 CA 루트 인증서로 신뢰 여부 검증

---

## HTTP 프로토콜 버전과 HOL Blocking

**난이도**: 기초

**핵심 키워드**: HOL Blocking, HTTP/1.1 Pipelining, HTTP/2 Multiplexing, Stream, 인터리브, TCP 레벨 HOL Blocking, QUIC, UDP, 0-RTT, Connection Migration

**HTTP/1.1 — HOL Blocking**:
- 하나의 TCP 연결에서 요청을 **순서대로(serial)** 처리 → 앞 요청 응답이 느리면 뒤 요청이 전부 대기
- Pipelining 지원하지만 응답은 반드시 요청 순서 그대로여야 함 → 사실상 HOL Blocking 해결 안 됨
- 브라우저 우회: 도메인당 TCP 연결 6개 병렬 오픈 → 연결 비용 증가
- **주의**: 스레드 할당 문제가 아닌 **TCP 연결 내 프로토콜 순서 문제**

**HTTP/2 — Application 레벨 HOL Blocking 해결**:
- 하나의 TCP 연결에서 여러 **Stream** 병렬 처리
- 각 Stream은 독립적인 요청/응답 쌍 → 메시지를 **프레임 단위로 쪼개 인터리브(interleave) 전송**
- 콜백 방식이 아님 — 프레임 수준의 멀티플렉싱

**HTTP/2의 남은 한계 — TCP 레벨 HOL Blocking**:
- TCP 패킷이 하나 유실 → OS가 재전송을 기다리는 동안 해당 TCP 연결 위 **모든 Stream이 함께 멈춤**
- Stream이 10개여도 하나의 TCP 패킷 손실에 전부 대기

**HTTP/3 (QUIC) — TCP 레벨 HOL Blocking 해결**:
- **UDP 기반 QUIC** 프로토콜 사용
- 각 Stream이 독립적인 흐름 → 하나의 Stream 패킷 유실이 다른 Stream에 영향 없음
- **0-RTT/1-RTT 연결**: TCP 3-way handshake + TLS를 합산해 연결 수립 시간 단축
- **Connection Migration**: IP 변경(Wi-Fi → 4G)에도 Connection ID 기반으로 연결 유지

**성능 요약**:
| | Application HOL | TCP HOL | 연결 비용 |
|---|---|---|---|
| HTTP/1.1 | ❌ 있음 | ❌ 있음 | 도메인당 6 TCP 연결 |
| HTTP/2 | ✅ 해결 | ❌ 여전함 | 1 TCP 연결 |
| HTTP/3 | ✅ 해결 | ✅ 해결 | 1 QUIC 연결 + 0-RTT |

**면접 세션 피드백 (2026-04-07 3회차)**:
- HOL Blocking을 스레드 할당 문제로 오해 → 프로토콜 레이어의 순서 문제가 핵심
- HTTP/2 Multiplexing을 "콜백 방식"으로 설명 → 프레임 단위 인터리브 전송이 정확한 표현
- HTTP/2의 TCP 레벨 HOL Blocking 한계 + HTTP/3 QUIC 해결 방식 미숙지

> 출처: https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/

---

## TLS 1.2와 TLS 1.3의 차이점은 무엇인가요?

**난이도**: 중급

**핵심 키워드**: RTT, 0-RTT, Perfect Forward Secrecy, Cipher Suite, 성능

**모범 답변 방향**:
- TLS 1.2: 2 RTT 핸드셰이크, RSA 키 교환 지원
- TLS 1.3: 1 RTT로 단축 (성능 향상), RSA 키 교환 제거, PFS 기본 적용
- TLS 1.3: 0-RTT 세션 재개 지원 (재연결 시 핸드셰이크 없이 통신)
- TLS 1.3: 취약한 Cipher Suite 제거로 보안 강화
- 2025년 기준 TLS 1.3 권장, TLS 1.1 이하 사용 금지

**꼬리 질문 예시**:
- "Perfect Forward Secrecy란 무엇인가요?"
- "0-RTT의 보안 위험성은?"

> 출처: https://community.citrix.com/tech-zone/build/tech-papers/networking-tls-best-practices-2025/

---

## HTTPS를 사용해도 보안 위협이 있을 수 있나요?

**난이도**: 심화

**핵심 키워드**: MITM, 인증서 위조, 만료 인증서, HSTS, 혼합 콘텐츠

**모범 답변 방향**:
- HTTPS는 전송 구간만 암호화 → 서버/클라이언트 자체 취약점은 별개
- 인증서 만료·호스트 불일치·신뢰하지 않는 CA 발급 → 경고 무시 시 MITM 가능
- 혼합 콘텐츠(Mixed Content): HTTPS 페이지에서 HTTP 리소스 로드 시 보안 무력화
- HSTS 미설정 시 첫 번째 요청에서 HTTP 다운그레이드 공격 가능
- 대응: HSTS 적용, 인증서 자동 갱신, HTTP → HTTPS 강제 리다이렉트

**꼬리 질문 예시**:
- "HSTS가 무엇이고 왜 필요한가요?"
- "인증서 유효기간이 왜 점점 짧아지는 추세인가요?"

> 출처: https://www.cloudflare.com/learning/ssl/why-is-http-not-secure/

---

## Nginx와 Tomcat의 차이를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: 웹 서버, WAS, Event-driven, Thread-per-request, 정적 파일, 리버스 프록시

**모범 답변 방향**:
- Nginx: 웹 서버 — 정적 파일 서빙, 리버스 프록시, 로드 밸런싱 담당
- Tomcat: WAS(웹 애플리케이션 서버) — 서블릿 컨테이너, Java 동적 처리 담당
- 실무에서는 Nginx(앞단) + Tomcat(뒷단) 조합으로 역할 분리
- Nginx가 TLS 종료 처리 → Tomcat은 평문으로 받아 처리 부하 감소

**꼬리 질문 예시**:
- "Nginx의 요청 처리 모델이 Tomcat과 다른 이유는?"
- "왜 Tomcat을 외부에 직접 노출하지 않나요?"

> 출처: https://technicalustad.com/nginx-vs-tomcat/

---

## Nginx가 동시 연결을 효율적으로 처리하는 원리는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: Event-driven, Non-blocking I/O, epoll, Master-Worker, C10K, Context Switching

**모범 답변 방향**:
- Tomcat: 요청마다 스레드 할당 → I/O 대기 중 스레드 점유 → 고부하 시 Context Switching 오버헤드
- Nginx: Master-Worker 구조 + Non-blocking I/O(epoll)
- Worker 1개가 1,000+ 커넥션 동시 처리 (스레드 생성 없음)
- I/O 대기 중 다른 요청 처리 → Context Switching 최소화
- C10K 문제(10,000개 동시 연결) 해결을 위해 설계된 아키텍처
- 성능: Nginx 워커당 100,000 req/s vs Tomcat 10,000~50,000 req/s

**꼬리 질문 예시**:
- "epoll이란 무엇인가요?"
- "Worker 프로세스 수는 어떻게 결정하나요?"
- "Non-blocking I/O에서 I/O 완료를 어떻게 감지하나요?"

> 출처: https://engineeringatscale.substack.com/p/nginx-millions-connections-event-driven-architecture

---

## Nginx의 sendfile이란 무엇이고, 왜 정적 파일 전송에 유리한가요?

**난이도**: 중급

**핵심 키워드**: sendfile, Zero-copy, 커널 버퍼, 유저 스페이스, tcp_nopush

**모범 답변 방향**:
- 전통 방식: 디스크 → 커널 버퍼 → 유저 스페이스 → 소켓 버퍼 (복사 3회)
- sendfile: 커널 버퍼 → 소켓 버퍼 직접 전송 (유저 스페이스 우회, 복사 없음)
- CPU·메모리 오버헤드 제거 → 대용량 정적 파일 전송 효율 극대화
- `sendfile on` + `tcp_nopush on` 조합으로 패킷 최적화
- **한계**: 프록시된 동적 콘텐츠에는 적용 불가 (정적 파일 전용)

**꼬리 질문 예시**:
- "sendfile을 NFS에서 사용하면 안 되는 이유는?"
- "tcp_nopush와 tcp_nodelay를 함께 설정하는 이유는?"

> 출처: https://www.getpagespeed.com/server-setup/nginx/nginx-sendfile-tcp-nopush-tcp-nodelay
