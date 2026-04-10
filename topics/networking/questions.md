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

HTTP는 데이터를 평문으로 전송하기 때문에 네트워크 경로상에서 누군가 패킷을 가로채면 내용을 그대로 읽을 수 있고, 위변조도 가능합니다. HTTPS는 이 HTTP 위에 TLS 암호화 레이어를 추가한 프로토콜로, 기본 포트는 443을 사용합니다. HTTPS가 보장하는 것은 크게 세 가지입니다. 첫째는 기밀성으로, 데이터를 암호화해 제3자가 내용을 볼 수 없게 합니다. 둘째는 무결성으로, 전송 중 데이터가 변조됐는지 MAC을 통해 감지할 수 있습니다. 셋째는 인증으로, 서버 인증서를 통해 내가 접속하는 서버가 신뢰할 수 있는 서버인지 확인합니다. 실제 통신 시에는 TLS 핸드셰이크 과정에서 세션 키를 교환한 뒤, 이후 본문 통신은 빠른 대칭 암호화로 처리합니다. 공개키 암호화를 계속 쓰면 연산 비용이 크기 때문에, 키 교환에만 공개키를 쓰고 본문에는 대칭키를 사용하는 구조입니다.

**꼬리 질문 예시**:
- "TLS 핸드셰이크 과정을 단계별로 설명해주세요"
- "공개키 암호화와 대칭키 암호화를 왜 함께 사용하나요?"

> 출처: https://aws.amazon.com/compare/the-difference-between-https-and-http/

---

## TLS Handshake 과정을 단계별로 설명해주세요.

**난이도**: 중급

**핵심 키워드**: Client Hello, Server Hello, 인증서, Pre-Master Secret, Session Key, 대칭키

**모범 답변 방향**:

TLS 핸드셰이크는 클라이언트와 서버가 암호화 통신을 시작하기 전에 세션 키를 안전하게 교환하는 과정입니다. 먼저 클라이언트가 Client Hello를 보내는데, 자신이 지원하는 암호화 방식 목록과 클라이언트 랜덤값을 함께 전송합니다. 서버는 Server Hello로 응답하면서 암호화 방식을 선택하고, 서버 랜덤값과 서버 인증서를 클라이언트에게 전달합니다. 클라이언트는 이 인증서를 OS나 브라우저에 사전 설치된 CA 루트 인증서로 검증한 뒤, 인증서에 포함된 서버의 공개키로 Pre-Master Secret을 암호화해 서버에 전송합니다. 이 단계까지 공개키 암호화가 쓰이는 이유는 Pre-Master Secret을 서버 외에는 아무도 복호화하지 못하게 하기 위함입니다. 이후 양측은 클라이언트 랜덤, 서버 랜덤, Pre-Master Secret 세 값을 조합해 동일한 세션 키를 각자 독립적으로 도출하고, 이때부터는 이 대칭 세션 키로 빠르게 통신합니다. 공개키 암호화는 연산 비용이 높기 때문에 키 교환 단계에만 사용하고, 실제 데이터 전송은 대칭키로 처리하는 것이 핵심 설계 원칙입니다.

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

TLS 1.2와 1.3의 가장 큰 차이는 핸드셰이크 성능과 보안 강화입니다. TLS 1.2는 핸드셰이크에 2 RTT가 필요하고 RSA 키 교환을 지원하는데, RSA는 서버의 개인키가 노출되면 과거 통신 내용까지 복호화할 수 있다는 문제가 있습니다. TLS 1.3은 핸드셰이크를 1 RTT로 줄여 연결 수립 속도를 높이고, RSA 키 교환을 제거해 Perfect Forward Secrecy를 기본으로 적용합니다. PFS는 세션마다 임시 키를 생성하기 때문에 특정 세션의 키가 유출되더라도 다른 세션의 통신 내용은 안전하게 보호됩니다. 또한 TLS 1.3은 세션 재개 시 0-RTT를 지원해, 이전에 연결한 서버와 다시 통신할 때 핸드셰이크 없이 바로 데이터를 전송할 수 있습니다. 다만 0-RTT는 재전송 공격(Replay Attack)에 취약하다는 보안 트레이드오프가 있어, 멱등성이 보장되는 요청에만 적용하는 것이 권장됩니다. 취약한 Cipher Suite도 정리돼 보안 수준 자체가 높아졌고, 2025년 기준으로 TLS 1.3이 권장 버전이며 TLS 1.1 이하는 사용이 금지된 수준입니다.

**꼬리 질문 예시**:
- "Perfect Forward Secrecy란 무엇인가요?"
- "0-RTT의 보안 위험성은?"

> 출처: https://community.citrix.com/tech-zone/build/tech-papers/networking-tls-best-practices-2025/

---

## HTTPS를 사용해도 보안 위협이 있을 수 있나요?

**난이도**: 심화

**핵심 키워드**: MITM, 인증서 위조, 만료 인증서, HSTS, 혼합 콘텐츠

**모범 답변 방향**:

HTTPS는 전송 구간을 암호화하는 것이지 모든 보안 문제를 해결하는 것은 아닙니다. 서버나 클라이언트 자체의 취약점, 애플리케이션 레벨 취약점은 HTTPS와 무관하게 존재합니다. 구체적인 위협을 말씀드리면, 먼저 인증서 관련 문제가 있습니다. 인증서가 만료됐거나 도메인이 불일치하거나 신뢰할 수 없는 CA에서 발급된 경우 브라우저가 경고를 띄우는데, 사용자가 이 경고를 무시하고 진행하면 MITM 공격이 가능해집니다. 다음으로 혼합 콘텐츠 문제가 있습니다. HTTPS 페이지 안에서 HTTP로 이미지나 스크립트를 불러오면, 그 리소스는 암호화되지 않아 공격자가 중간에서 변조할 수 있습니다. 또한 HSTS가 설정되지 않은 경우, 사용자가 처음 HTTP로 접속하는 순간을 공격자가 낚아채 HTTPS 업그레이드 응답을 가로채는 다운그레이드 공격이 가능합니다. HSTS는 브라우저에 "이 도메인은 항상 HTTPS로만 접속하라"고 미리 지시해 이 문제를 방지합니다. 실무에서는 HSTS 설정, 인증서 자동 갱신, HTTP 요청의 HTTPS 강제 리다이렉트를 기본으로 적용하는 것이 중요합니다.

**꼬리 질문 예시**:
- "HSTS가 무엇이고 왜 필요한가요?"
- "인증서 유효기간이 왜 점점 짧아지는 추세인가요?"

> 출처: https://www.cloudflare.com/learning/ssl/why-is-http-not-secure/

---

## Nginx와 Tomcat의 차이를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: 웹 서버, WAS, Event-driven, Thread-per-request, 정적 파일, 리버스 프록시

**모범 답변 방향**:

Nginx와 Tomcat은 역할 자체가 다른 서버입니다. Nginx는 웹 서버로, 정적 파일 서빙, 리버스 프록시, 로드 밸런싱이 주된 역할입니다. Tomcat은 WAS, 즉 웹 애플리케이션 서버로, 서블릿 컨테이너 역할을 하며 Java 기반의 동적 요청을 처리합니다. 실무에서는 Nginx를 앞단에 두고 Tomcat을 뒷단에 두는 구조를 많이 사용합니다. Nginx가 외부 요청을 받아 TLS 종료까지 처리한 다음 평문 HTTP로 Tomcat에 넘기면, Tomcat은 TLS 처리 부하 없이 비즈니스 로직에만 집중할 수 있습니다. Nginx를 Tomcat 앞에 두는 또 다른 이유는 보안과 성능 측면에서도 있습니다. Tomcat을 외부에 직접 노출하지 않으면 공격 표면을 줄일 수 있고, 정적 파일은 Nginx가 훨씬 빠르게 처리할 수 있기 때문에 Tomcat의 스레드 자원을 동적 요청에만 집중시킬 수 있습니다. 요청 처리 모델도 근본적으로 다른데, Nginx는 이벤트 기반 Non-blocking I/O 방식이고 Tomcat은 기본적으로 요청당 스레드를 할당하는 방식이라 대규모 동시 연결 처리에서 Nginx가 훨씬 유리합니다.

**꼬리 질문 예시**:
- "Nginx의 요청 처리 모델이 Tomcat과 다른 이유는?"
- "왜 Tomcat을 외부에 직접 노출하지 않나요?"

> 출처: https://technicalustad.com/nginx-vs-tomcat/

---

## Nginx가 동시 연결을 효율적으로 처리하는 원리는 무엇인가요?

**난이도**: 중급

**핵심 키워드**: Event-driven, Non-blocking I/O, epoll, Master-Worker, C10K, Context Switching

**모범 답변 방향**:

Nginx가 동시 연결을 효율적으로 처리하는 핵심은 이벤트 기반 Non-blocking I/O 구조입니다. Tomcat 같은 Thread-per-request 방식은 요청마다 스레드를 할당하기 때문에, I/O 대기 중에도 스레드가 자원을 점유하고 있습니다. 동시 연결이 수천 개를 넘어가면 스레드 수가 폭발적으로 늘어나고, 이들 사이의 Context Switching 오버헤드가 심각해집니다. Nginx는 Master-Worker 구조로 동작하는데, Master 프로세스는 설정 관리와 Worker 관리를 담당하고, 실제 요청 처리는 Worker 프로세스가 담당합니다. Worker 하나가 epoll을 통해 수천 개 이상의 커넥션을 동시에 감시하면서, I/O가 준비된 소켓에만 반응하는 방식입니다. I/O 대기 중에는 그 커넥션을 블록하지 않고 다른 커넥션의 이벤트를 처리하러 넘어가기 때문에, 스레드를 새로 생성하지 않고도 수많은 연결을 처리할 수 있습니다. 이 구조가 C10K 문제, 즉 서버 하나에서 10,000개 동시 연결을 처리하는 것을 목표로 설계된 아키텍처입니다. Worker 프로세스 수는 보통 CPU 코어 수와 동일하게 설정해, 코어 간 Context Switching 없이 각 Worker가 한 코어를 전담하도록 구성합니다.

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

sendfile은 정적 파일을 전송할 때 불필요한 데이터 복사를 제거하는 커널 수준의 최적화 기능입니다. 일반적인 파일 전송 흐름을 보면, 디스크에서 데이터를 읽어 커널 버퍼에 올리고, 이것을 다시 유저 스페이스로 복사한 뒤, 애플리케이션이 소켓 버퍼로 다시 복사해 전송합니다. 이 과정에서 데이터가 최소 두세 번 복사되고, 유저 스페이스와 커널 스페이스 사이를 오가는 컨텍스트 전환도 발생합니다. sendfile 시스템 콜을 사용하면 커널 버퍼에서 소켓 버퍼로 데이터를 직접 전달할 수 있어, 유저 스페이스를 거치지 않는 Zero-copy 전송이 가능합니다. CPU와 메모리 오버헤드가 크게 줄어들기 때문에 이미지, CSS, JS 같은 대용량 정적 파일을 대량으로 서빙하는 환경에서 효율이 극대화됩니다. Nginx 설정에서 `sendfile on`과 `tcp_nopush on`을 함께 쓰면 패킷을 최대한 모아서 보내는 추가 최적화도 됩니다. 다만 sendfile은 디스크에서 직접 읽는 정적 파일에만 적용되고, 프록시를 통해 받아온 동적 콘텐츠에는 적용할 수 없다는 한계가 있습니다. 또한 NFS 같은 네트워크 파일시스템 환경에서는 커널이 sendfile을 지원하지 않는 경우가 있어 주의가 필요합니다.

**꼬리 질문 예시**:
- "sendfile을 NFS에서 사용하면 안 되는 이유는?"
- "tcp_nopush와 tcp_nodelay를 함께 설정하는 이유는?"

> 출처: https://www.getpagespeed.com/server-setup/nginx/nginx-sendfile-tcp-nopush-tcp-nodelay
