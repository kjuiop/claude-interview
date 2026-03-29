---
tags: [networking, http, https, tls, ssl, security, nginx, tomcat, web-server]
related: [distributed-systems, system-design, kubernetes]
---

# Networking — 핵심 개념

→ [[home]] | 질문 모음: [[topics/networking/questions]]

---

## HTTP vs HTTPS 핵심 차이

| 항목 | HTTP | HTTPS |
|---|---|---|
| 암호화 | 평문 전송 (없음) | TLS로 암호화 |
| 포트 | 80 | 443 |
| 보안 | 도청·위변조 가능 | 기밀성·무결성·인증 보장 |
| 성능 | 오버헤드 없음 | TLS 핸드셰이크 오버헤드 |

**HTTPS가 보장하는 세 가지:**
- **기밀성(Confidentiality)**: 제3자가 내용을 읽을 수 없음
- **무결성(Integrity)**: 전송 중 데이터 변조 감지
- **인증(Authentication)**: 서버가 실제 그 서버임을 증명 (인증서)

---

## SSL / TLS 개념

- **SSL**: Netscape가 개발한 초기 프로토콜 (SSL 2.0, 3.0)
- **TLS**: SSL 3.0을 계승한 IETF 국제 표준 (TLS 1.0 → 1.1 → 1.2 → 1.3)
- 현재는 TLS가 표준이지만 관례상 SSL/TLS로 혼용
- TLS 1.0, 1.1, SSL 2.0, 3.0은 보안 취약점으로 사용 금지
- **2025년 기준 권장**: TLS 1.3 (TLS 1.2는 허용, 이하 금지)

---

## TLS Handshake 과정

### TLS 1.2 (4-Way Handshake, 2 RTT)

```
Client                          Server
  │──── Client Hello ──────────→│  (지원 암호화 방식 목록 + 랜덤값)
  │←─── Server Hello ───────────│  (선택 암호화 방식 + 서버 랜덤값 + 인증서)
  │──── Key Exchange ──────────→│  (Pre-Master Secret, 공개키로 암호화)
  │──── Change Cipher Spec ────→│
  │←─── Change Cipher Spec ─────│
  │     [암호화 통신 시작]        │
```

1. **Client Hello**: 지원 가능 암호화 방식(Cipher Suite) + 클라이언트 랜덤값
2. **Server Hello**: 선택한 Cipher Suite + 서버 랜덤값 + **서버 인증서**
3. **Key Exchange**: 클라이언트가 인증서의 공개키로 Pre-Master Secret 암호화 전송
4. **Session Key 생성**: 클라이언트 랜덤 + 서버 랜덤 + Pre-Master Secret → **대칭키(Session Key)** 생성
5. 이후 Session Key로 대칭 암호화 통신

### TLS 1.3 (1 RTT, 더 빠르고 안전)

- 핸드셰이크 왕복 횟수 감소 (2 RTT → 1 RTT)
- 취약한 암호화 방식(RSA 키 교환 등) 지원 제거
- **0-RTT 재연결** 지원 (세션 재개 시 handshake 없이 바로 통신)
- Perfect Forward Secrecy 기본 적용

---

## 암호화 방식 조합

TLS는 **공개키 + 대칭키** 두 방식을 조합해 사용:

| 방식 | 장점 | 단점 | TLS에서의 역할 |
|---|---|---|---|
| 대칭키 | 빠름 | 키 배송 문제 | 실제 데이터 암호화 |
| 공개키 | 안전한 키 교환 | 느림 | Session Key 교환 단계 |

→ 공개키로 안전하게 Session Key를 교환한 뒤, 이후 통신은 빠른 대칭키로 처리

---

## 서버 인증서 (X.509)

- **CA(Certificate Authority)**: 인증서를 발급·서명하는 신뢰 기관 (DigiCert, Let's Encrypt 등)
- 인증서에 포함된 정보: 도메인, 공개키, 유효기간, CA 서명
- 클라이언트(브라우저)는 CA 루트 인증서를 미리 신뢰 목록에 보유 → 서버 인증서 검증
- **인증서 유효기간**: 2025년 이후 점진적 단축 (398일 → 목표 47일, 2029년)

**인증서 체인 검증 순서:**
```
서버 인증서 → 중간 CA → 루트 CA (브라우저 신뢰 저장소)
```

---

## 실무 주의사항

- TLS 1.3 사용 권장, TLS 1.2는 허용, 1.1 이하 비활성화
- Cipher Suite: ECDHE + AES-GCM 또는 ChaCha20-Poly1305 권장 (PFS 보장)
- 인증서 만료 모니터링 필수 (자동 갱신 설정 권장 — Let's Encrypt + certbot)
- 세션 재개(Session Resumption) 활성화로 핸드셰이크 오버헤드 감소
- HTTP → HTTPS 리다이렉트 설정 + HSTS(HTTP Strict Transport Security) 적용

> 출처: https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/
> 출처: https://aws.amazon.com/compare/the-difference-between-https-and-http/
> 출처: https://community.citrix.com/tech-zone/build/tech-papers/networking-tls-best-practices-2025/

---

## Nginx vs Tomcat — 역할과 아키텍처

### 역할 차이

| 항목 | Nginx | Tomcat |
|---|---|---|
| 분류 | 웹 서버 (Web Server) | 웹 애플리케이션 서버 (WAS) |
| 핵심 기능 | 정적 파일 서빙, 리버스 프록시, 로드 밸런싱 | 서블릿 컨테이너, 동적 Java 애플리케이션 실행 |
| 동적 처리 | 불가 (백엔드로 위임) | 가능 (JVM 위에서 실행) |

---

## Nginx 요청 처리 모델 — Event-Driven

### Master-Worker 구조

```
Master Process
  ├── Worker Process 1  ← 수천 개의 커넥션 동시 처리
  ├── Worker Process 2
  └── Worker Process N  (보통 CPU 코어 수와 동일)
```

- **Non-blocking I/O + epoll(Linux)**: I/O 대기 중 다른 요청 처리 가능
- Worker 1개가 1,000+ 커넥션 동시 처리 (스레드 생성 없음)
- Context Switching 최소화 → CPU 효율 극대화
- **C10K 문제** 해결을 위해 설계된 아키텍처
- 정적 콘텐츠 응답시간: 1ms 미만, 워커당 100,000 req/s 처리 가능

---

## Tomcat 요청 처리 모델 — Thread-Per-Request

- 요청마다 **스레드 할당** (Thread Pool에서 꺼내서 사용)
- I/O 대기 시 스레드 점유 → 동시 연결 수 = 스레드 수에 비례
- Context Switching 오버헤드 발생 (고부하 시 성능 저하)
- JVM GC로 인한 100~500ms 지연 가능
- NIO 커넥터로 튜닝 시 10,000~50,000 req/s

---

## 정적 파일 전송 효율 — sendfile (Zero-Copy)

### 전통 방식 (복사 발생)
```
디스크 → 커널 버퍼 → 유저 스페이스 → 소켓 버퍼 → 네트워크
         ↑ 복사 1      ↑ 복사 2       ↑ 복사 3
```

### sendfile (Zero-Copy)
```
디스크 → 커널 버퍼 ──────────────────→ 소켓 버퍼 → 네트워크
         (유저 스페이스 우회, 복사 없음)
```

- Nginx: `sendfile on` 설정으로 OS 레벨 zero-copy 활용
- Tomcat: sendfile 미지원 → 정적 파일 전송 효율 열위
- **주의**: sendfile은 정적 파일에만 적용. 프록시된 동적 콘텐츠는 해당 없음

```nginx
# Nginx 권장 설정
sendfile        on;
sendfile_max_chunk 1m;   # 워커 독점 방지
tcp_nopush      on;      # 패킷 최적화 (sendfile과 함께 사용)
tcp_nodelay     on;      # Keep-alive 연결 지연 최소화
```

---

## Nginx + Tomcat 조합 아키텍처

```
클라이언트 → Nginx (80/443)
               ├── 정적 파일 요청 → Nginx 직접 서빙 (sendfile)
               └── 동적 요청 → Tomcat (8080) 프록시
```

**조합하는 이유:**
1. **역할 분리**: 정적 파일은 Nginx(빠름), 동적 처리는 Tomcat(Java)
2. **TLS 처리**: Nginx에서 TLS 종료 → Tomcat은 평문으로 처리 (부하 감소)
3. **부하 분산**: 여러 Tomcat 인스턴스에 로드 밸런싱
4. **보안**: Tomcat 포트를 외부에 직접 노출하지 않음

> 출처: https://technicalustad.com/nginx-vs-tomcat/
> 출처: https://www.getpagespeed.com/server-setup/nginx/nginx-sendfile-tcp-nopush-tcp-nodelay
> 출처: https://engineeringatscale.substack.com/p/nginx-millions-connections-event-driven-architecture
