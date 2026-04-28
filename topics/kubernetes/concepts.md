---
tags: [kubernetes, k8s, deployment, hpa, devops]
related: [distributed-systems, golang, networking, system-design]
---

# Kubernetes 핵심 개념

→ [[home]] | [[topics/kubernetes/questions]]

---

## Rolling Update (무중단 배포)

기존 Pod를 하나씩 교체하는 기본 배포 전략.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # 배포 중 최소 가용 Pod 보장
    maxSurge: 1         # 동시에 추가로 띄울 수 있는 Pod 수
```

- `maxUnavailable: 0` + `maxSurge: 1`: 항상 기존 Pod가 살아있는 상태에서 새 Pod를 먼저 기동 → 무중단 보장
- 새 Pod가 Ready 상태가 되면 기존 Pod를 하나 제거

---

## readinessProbe (트래픽 제어 핵심)

새 Pod가 **실제로 트래픽 받을 준비**가 됐을 때만 Service에 연결.

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

- **없으면**: 아직 초기화 중인 Pod에 트래픽이 유입되어 오류 발생
- Rolling Update에서 무중단 배포를 실질적으로 보장하는 핵심 설정
- livenessProbe와 구분: readiness = 트래픽 수신 가능 여부, liveness = 프로세스 정상 동작 여부

---

## PodDisruptionBudget (PDB)

노드 점검/업그레이드 시 동시에 내려가는 Pod 수를 제한.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-service
```

- `minAvailable: 1`: 항상 최소 1개 Pod는 살아있도록 보장
- 노드 드레인(drain) 시 PDB를 위반하면 드레인 차단

---

## HPA (Horizontal Pod Autoscaler)

트래픽에 따라 Pod 수를 자동으로 조절.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**HPA 기준 설정 방법**:
1. k6 등 부하 테스트 도구로 실제 트래픽 패턴 재현
2. CPU/메모리 사용률 임계점 측정
3. 측정값의 70~80%를 HPA threshold로 설정 (여유 확보)
4. scale-out 후 안정화까지 걸리는 시간 고려 (`--horizontal-pod-autoscaler-sync-period`)

**주의**: CPU 기반 HPA는 지연이 있으므로 트래픽 급증 시 대응이 늦을 수 있음 → 예측 가능한 트래픽이면 Scheduled Scaling도 병행.

---

## WebSocket Graceful Shutdown

WebSocket, gRPC 스트림 등 장시간 연결이 있는 서비스의 Pod 교체 시 기존 커넥션을 안전하게 처리하는 패턴.

### 종료 시퀀스
```
1. kubectl rollout 시작
2. readinessProbe 실패 → Service endpoints에서 Pod 제거 (신규 연결 차단)
3. preStop hook 실행 (설정된 경우) ← SIGTERM과 동시에 실행
4. SIGTERM 신호 전송
5. terminationGracePeriodSeconds 카운트다운
6. 기존 WebSocket 커넥션 자연 종료 대기
7. 초과 시 SIGKILL (강제 종료)
```

### 핵심 설정

```yaml
spec:
  terminationGracePeriodSeconds: 300  # 최대 대기 시간 (WebSocket 평균 세션 길이 기준)
  containers:
  - name: ws-app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # endpoints 제거 시간 확보
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
    livenessProbe:
      httpGet:
        path: /health/live   # readiness와 분리 필수
        port: 8080
```

### 왜 preStop hook에 sleep이 필요한가
- Pod 종료 시작 → endpoints 제거 → ingress controller 업데이트까지 **수 초~수십 초 지연** 발생
- 그 사이에 신규 트래픽이 종료 중인 Pod로 유입될 수 있음
- `preStop: sleep 5` 로 endpoints 제거가 전파될 시간 확보

### readiness vs liveness 분리 이유
- DB가 다운되면 liveness probe도 실패 → Pod 무한 재시작 루프 발생
- readiness: "지금 트래픽 받을 수 있나?" (DB 연결 포함)
- liveness: "프로세스가 살아있나?" (응답 자체만)

### 실무 주의사항
- **경매·결제 WebSocket**: 강제 종료 시 치명적 → `terminationGracePeriodSeconds`를 충분히 설정
- WebSocket 평균 세션 길이를 Grafana로 측정 후 grace period 결정
- `terminationGracePeriodSeconds`는 preStop hook 시간 + 컨테이너 정상 종료 시간의 합을 포함해야 함

---

## Connection-Aware Graceful Shutdown (커넥션 수 기반 종료)

"평균 세션 시간만큼 무조건 기다린다"는 한계를 극복하는 패턴.
**활성 WebSocket 커넥션이 0개가 되는 즉시 Pod를 종료**하는 방식.

### 핵심 아이디어
애플리케이션이 자체적으로 WebSocket 커넥션 수를 추적하고, preStop hook에서 이를 폴링해 0이 되면 종료.

```
SIGTERM 수신
  → readinessProbe 실패 (신규 연결 차단)
  → preStop hook: /status를 1초마다 폴링
  → active_connections == 0 확인
  → 즉시 종료 (평균 세션 시간 기다릴 필요 없음)
```

### ⚠️ net/http Server.Shutdown()의 한계
Go의 `http.Server.Shutdown()`은 **hijacked 연결(WebSocket, SSE)을 추적하지 않는다**.
WebSocket은 HTTP Upgrade 이후 서버의 관리 범위를 벗어나므로, **별도 커넥션 트래커 구현 필수**.

### Go 구현 예시

```go
// 1. 커넥션 트래커
type WSTracker struct {
    count      atomic.Int32
    draining   atomic.Bool
}

func (t *WSTracker) Connect() {
    t.count.Add(1)
}

func (t *WSTracker) Disconnect() {
    t.count.Add(-1)
}

func (t *WSTracker) ActiveCount() int32 {
    return t.count.Load()
}

// 2. WebSocket 핸들러에서 트래킹
func wsHandler(tracker *WSTracker) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 드레이닝 중이면 신규 연결 거부
        if tracker.draining.Load() {
            http.Error(w, "shutting down", http.StatusServiceUnavailable)
            return
        }
        conn, _ := upgrader.Upgrade(w, r, nil)
        tracker.Connect()
        defer tracker.Disconnect()
        // ... WebSocket 처리
    }
}

// 3. 상태 엔드포인트 (preStop hook이 폴링할 대상)
func statusHandler(tracker *WSTracker) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]any{
            "active_connections": tracker.ActiveCount(),
        })
    }
}

// 4. SIGTERM 처리
func main() {
    tracker := &WSTracker{}
    // 서버 설정...

    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM)
    <-sigCh

    tracker.draining.Store(true)  // 신규 연결 거부 시작
    // preStop hook의 폴링이 완료되면 Kubernetes가 SIGKILL 전에 프로세스 종료됨
}
```

### Kubernetes 설정

```yaml
spec:
  terminationGracePeriodSeconds: 300  # 안전망 (커넥션이 오래 걸릴 경우 대비)
  containers:
  - name: ws-server
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            # 드레인 시작 신호
            curl -s -X POST localhost:8080/drain
            # 커넥션 0이 될 때까지 폴링
            while [ $(curl -s localhost:8080/status | jq '.active_connections') -gt 0 ]; do
              sleep 1
            done
```

### Istio 사용 시 — 자동 지원
Istio v1.12+는 `EXIT_ON_ZERO_ACTIVE_CONNECTIONS` 기능을 기본 지원.
Envoy 프록시가 모든 활성 연결이 완료될 때까지 자동으로 종료를 지연시킴.

### sleep 기반 vs Connection-Aware 비교

| | sleep 기반 | Connection-Aware |
|---|---|---|
| 구현 복잡도 | 낮음 | 높음 (트래커 구현 필요) |
| 종료 속도 | 항상 최대 대기 | 커넥션 없으면 즉시 종료 |
| 정확도 | 평균값 의존 | 실제 상태 기반 |
| 적합한 상황 | 단순한 서비스 | 경매·결제 등 정밀 제어 필요 시 |

> 출처: https://oneuptime.com/blog/post/2026-02-09-prestop-hooks-zero-connection-drop/view

---

## OS 수준 커넥션 모니터링 (ss / netstat / lsof)

애플리케이션 코드 수정 없이 Linux 명령어만으로 preStop hook에서 활성 커넥션을 확인하는 방법.

### 가능한 명령어 예시

```bash
# ss — 특정 포트의 ESTABLISHED 커넥션 수 확인
ss -tn state established '( dport = :8080 )' | wc -l

# netstat — ESTABLISHED 상태 커넥션 수
netstat -an | grep ':8080' | grep ESTABLISHED | wc -l

# lsof — 포트를 열고 있는 프로세스의 ESTABLISHED 커넥션
lsof -i :8080 -s TCP:ESTABLISHED 2>/dev/null
```

### preStop hook 적용 예시

```yaml
lifecycle:
  preStop:
    exec:
      command:
      - /bin/sh
      - -c
      - |
        # 신규 연결 차단을 위한 짧은 대기
        sleep 5
        # 포트 8080의 활성 커넥션이 0이 될 때까지 대기
        while [ $(ss -tn state established '( dport = :8080 )' | wc -l) -gt 1 ]; do
          sleep 1
        done
```

### ⚠️ 핵심 한계: WebSocket과 HTTP를 구분할 수 없다

`ss`, `netstat`, `lsof`는 **Layer 4(TCP 소켓 레이어)** 에서 동작한다.
WebSocket은 **Layer 7(애플리케이션 레이어)** 프로토콜이며, HTTP와 동일한 포트(80/443/8080)를 사용한다.

```
포트 8080에서:
  - HTTP 요청        → ESTABLISHED (일반 커넥션)
  - WebSocket 연결   → ESTABLISHED (동일하게 보임)
  → ss는 둘을 구분할 수 없다
```

따라서 **"WebSocket 커넥션만" 0이 됐는지 확인하는 것은 OS 명령어만으로 불가능**.
HTTP 요청도 포함된 전체 TCP 커넥션 수를 기준으로 판단해야 한다.

### 컨테이너 이미지별 도구 가용성

| 이미지 | netstat | ss | lsof |
|---|---|---|---|
| Ubuntu/Debian | 기본 설치 | `iproute2` 패키지 필요 | 별도 설치 |
| Alpine | `net-tools` 패키지 필요 | `iproute2` 패키지 필요 | 별도 설치 |
| Distroless | **없음** | **없음** | **없음** |

→ Distroless나 경량 이미지는 별도 빌드 단계에서 도구를 추가하거나, 방식 자체를 재고해야 함.

### OS 모니터링 vs 애플리케이션 카운터 비교

| | OS 명령어 (ss/netstat) | 앱 내 atomic 카운터 |
|---|---|---|
| 구현 복잡도 | 낮음 (스크립트만) | 높음 (코드 수정 필요) |
| WebSocket 구분 | **불가** (포트만 식별) | **가능** |
| 정확도 | 중간 (HTTP 포함) | 높음 |
| 도구 의존성 | 이미지에 설치 필요 | 없음 |
| 경량 이미지 | Distroless 불가 | 무관 |

### 실무 권장 조합
- **간단한 API 서버**: OS 모니터링으로 충분 (WebSocket 없음)
- **WebSocket 전용 서버**: 애플리케이션 카운터 방식 필요
- **하이브리드**: 앱에서 `/drain` 엔드포인트 호출 → OS 명령어로 2차 확인

> 출처: https://raymii.org/s/snippets/Get_number_of_incoming_connections_on_specific_ports_with_ss.html

---

## Istio — Service Mesh 개요

Istio는 Kubernetes 위에서 동작하는 **Service Mesh** 플랫폼이다.
마이크로서비스 간 통신을 애플리케이션 코드 수정 없이 네트워크 레이어에서 제어한다.

### 구성 요소

| 레이어 | 구성 요소 | 역할 |
|---|---|---|
| **Data Plane** | Envoy Proxy (sidecar) | 실제 트래픽 처리, 암호화, 라우팅 |
| **Control Plane** | Istiod | Envoy 설정 생성·배포, CA(인증서 발급) |

### Istiod 내부 기능
- **Pilot**: VirtualService, DestinationRule → Envoy xDS 설정으로 변환 후 배포
- **Citadel**: SPIFFE 기반 X.509 인증서 발급 및 갱신 (mTLS 재료)
- **Galley**: K8s 리소스 검증

### 💬 면접 답변 형태로 읽기

Istio는 Kubernetes 환경에서 마이크로서비스 간 통신을 코드 수정 없이 제어할 수 있는 Service Mesh 플랫폼입니다. 핵심 구조는 Data Plane과 Control Plane으로 나뉩니다. Data Plane은 각 Pod에 Envoy Proxy를 sidecar로 주입해서 모든 인바운드·아웃바운드 트래픽을 가로채어 처리합니다. Control Plane인 Istiod는 VirtualService, DestinationRule 같은 Istio 리소스를 Envoy 전용 설정으로 변환해 런타임에 배포하고, CA 역할을 해서 각 워크로드에 인증서를 발급합니다. 이를 통해 트래픽 제어, mTLS 자동 암호화, 서킷 브레이커, 분산 트레이싱 같은 기능을 애플리케이션 레이어가 아닌 네트워크 레이어에서 제공합니다.

> 출처: https://velog.io/@beryl/Istio-%EA%B0%9C%EB%85%90

---

## Istio — Envoy Sidecar 주입 및 트래픽 인터셉트

### Sidecar 주입 방식

```yaml
# 네임스페이스에 라벨만 달면 자동 주입
kubectl label namespace default istio-injection=enabled
```

Pod 생성 시 **MutatingWebhookConfiguration**이 작동해서 자동으로 2개 컨테이너 추가:
1. `istio-init` (init container): iptables 규칙 설정 → 모든 트래픽을 Envoy로 리다이렉트
2. `istio-proxy` (sidecar): Envoy 프록시 본체

### 트래픽 흐름

```
[Pod A]                          [Pod B]
App → Envoy(sidecar) → 네트워크 → Envoy(sidecar) → App
         ↑                              ↑
    mTLS 암호화                    mTLS 복호화
    라우팅 규칙 적용               라우팅 규칙 적용
    메트릭 수집                    메트릭 수집
```

애플리케이션 코드는 `localhost`로 통신하는 것처럼 동작 → Envoy가 투명하게 가로채서 처리.

### 💬 면접 답변 형태로 읽기

Envoy sidecar 주입은 네임스페이스에 `istio-injection=enabled` 라벨만 달면 자동으로 동작합니다. Pod가 생성될 때 Istio의 MutatingWebhookConfiguration이 Pod spec을 수정해서 두 가지를 추가합니다. 하나는 init container로서 iptables 규칙을 설정해 모든 트래픽을 Envoy 포트로 리다이렉트하고, 다른 하나는 istio-proxy sidecar로서 실제 Envoy 프록시입니다. 이렇게 하면 애플리케이션 코드는 마치 직접 통신하는 것처럼 작성하면 되고, 실제 네트워크 트래픽은 Envoy를 통해 흐르면서 mTLS 암호화, 트래픽 라우팅, 메트릭 수집이 투명하게 처리됩니다. 서비스 A가 서비스 B를 호출하면, A의 Envoy → 암호화 → 네트워크 → B의 Envoy → 복호화 → B 앱 순서로 흐릅니다.

> 출처: https://oneuptime.com/blog/post/2026-02-24-understand-envoy-proxy-architecture-in-istio/view

---

## Istio — mTLS 자동 적용

Istio는 서비스 간 통신에 **mTLS(mutual TLS)** 를 자동으로 적용한다.
애플리케이션 코드 수정 없이 서비스 간 암호화 + 상호 인증이 가능하다.

### 동작 원리

```
Istiod (CA 역할)
  └─ 각 워크로드에 SPIFFE 기반 X.509 인증서 발급
       └─ Envoy가 자동으로 인증서 갱신, TLS 핸드셰이크, 암호화 처리
```

### PeerAuthentication 설정

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT   # STRICT: mTLS 강제, PERMISSIVE: HTTP도 허용
```

- **STRICT**: 모든 트래픽 mTLS 필수 — 인증서 없는 요청 차단
- **PERMISSIVE**: mTLS + 일반 HTTP 모두 허용 — 마이그레이션 기간에 사용

### K8s NetworkPolicy와의 차이

| | K8s NetworkPolicy | Istio mTLS |
|---|---|---|
| 레이어 | L4 (IP/포트 기반) | L7 (서비스 신원 기반) |
| 제어 수준 | "이 포트로 오는 트래픽 허용/차단" | "이 서비스 신원의 요청만 허용" |
| 암호화 | 없음 | TLS 자동 암호화 |
| 코드 수정 | 불필요 | 불필요 |

### 💬 면접 답변 형태로 읽기

Istio의 mTLS는 서비스 간 통신에 상호 TLS 인증을 자동으로 적용하는 기능입니다. Kubernetes의 NetworkPolicy가 IP와 포트 기반의 L4 레이어에서만 트래픽을 제어하는 반면, Istio의 mTLS는 L7 레이어에서 서비스 신원 자체를 기반으로 인증합니다. 동작 원리는 Istiod가 CA 역할을 해서 각 워크로드에 SPIFFE 표준 기반의 X.509 인증서를 발급하고, 각 Pod의 Envoy sidecar가 이 인증서로 TLS 핸드셰이크와 암호화를 자동으로 처리합니다. 개발자는 코드를 한 줄도 수정하지 않아도 됩니다. PeerAuthentication 리소스에서 mode를 STRICT로 설정하면 인증서가 없는 요청은 자동으로 차단되고, PERMISSIVE 모드는 기존 서비스를 점진적으로 마이그레이션할 때 사용합니다. 이를 통해 클러스터 내부에서도 Zero Trust 보안 모델을 달성할 수 있습니다.

> 출처: https://oneuptime.com/blog/post/2026-01-07-istio-mtls-security/view

---

## Istio — 트래픽 관리 (VirtualService / DestinationRule)

### VirtualService — 라우팅 규칙

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service
  http:
  - match:
    - headers:
        x-version:
          exact: "v2"
    route:
    - destination:
        host: my-service
        subset: v2
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 90
    - destination:
        host: my-service
        subset: v2
      weight: 10
```

- 헤더 기반 라우팅, 가중치 기반 트래픽 분할(Canary)
- 재시도, 타임아웃, Fault Injection 설정 가능

### DestinationRule — 서브셋 정의

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100   # 서킷 브레이커
```

### 💬 면접 답변 형태로 읽기

Istio의 트래픽 관리는 VirtualService와 DestinationRule 두 리소스가 짝을 이루어 동작합니다. VirtualService는 "어디로 보낼지" 라우팅 규칙을 정의하는데, 헤더 값이나 URL 패턴에 따라 다른 버전으로 보내거나, 가중치를 설정해 90%는 v1, 10%는 v2로 보내는 Canary 배포를 구현할 수 있습니다. DestinationRule은 "대상의 세부 정책"을 정의해서 Pod를 버전별 subset으로 나누고, 서킷 브레이커나 커넥션 풀 설정을 적용합니다. Kubernetes의 기본 Service는 단순히 Label Selector로 Pod를 선택해 부하를 분산하는 수준인데, Istio를 쓰면 코드 수정 없이 훨씬 세밀한 트래픽 제어가 가능합니다. 실무에서는 신규 버전 배포 시 VirtualService의 weight를 조금씩 늘려가면서 안정성을 확인하는 방식으로 사용합니다.

> 출처: https://gruuuuu.hololy.org/cloud/service-mesh-istio/
