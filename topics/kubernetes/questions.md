---
tags: [kubernetes, k8s, 면접질문, deployment, hpa]
related: [kubernetes/concepts]
---

# Kubernetes 면접 질문

→ [[home]] | [[topics/kubernetes/concepts]]

---

## StatefulSet vs Deployment

### Q. StatefulSet과 Deployment의 핵심 차이를 설명하고, Kafka/MySQL을 K8s에서 운영할 때 StatefulSet을 선택해야 하는 이유를 설명해주세요.

| | Deployment | StatefulSet |
|---|---|---|
| Pod 이름 | 랜덤(`app-7d4b9c-xkz2p`) | 고정(`app-0`, `app-1`, `app-2`) |
| DNS | 없음 (Service IP) | 고정(`app-0.app.ns.svc.cluster.local`) |
| 배포/종료 순서 | 동시 | 순서 보장 (0→1→2 배포, 2→1→0 종료) |
| 스토리지 | Pod 재시작 시 데이터 소실 | PVC 영구 바인딩, 재시작 후에도 유지 |

**PVC 바인딩 방식:**
- `volumeClaimTemplates`로 Pod마다 PVC 자동 생성
- `app-0 → pvc-app-0`, `app-1 → pvc-app-1` 독립 바인딩
- Pod 재시작해도 같은 PVC 재바인딩 → 데이터 유지

**Kafka StatefulSet 필요 이유:**
- 브로커마다 고유한 `broker.id` 필요 → Pod 이름 고정 필요
- 파티션 데이터가 특정 브로커 디스크에 저장 → PVC 고정 필요
- Pod 이름 랜덤하면 재시작 후 데이터 정합성 깨짐

**MySQL StatefulSet 필요 이유:**
- Primary는 항상 `mysql-0`으로 고정
- Replica가 `mysql-0.mysql` 고정 DNS로 binlog 수신

**면접 세션 피드백 (2026-04-12 4회차)**:
- 처음 접한 주제 — 위 구조 전체 암기 필요
- 이력서 K8s 실운영 경험 연결: Kafka StatefulSet volumeClaimTemplates 구성 방식 설명 가능해야 함

---

## Kubernetes에서 서비스 무중단 배포를 보장하기 위해 어떤 전략을 사용했나요? HPA는 어떤 기준으로 설정하나요?

**난이도**: 심화

**핵심 키워드**: Rolling Update, maxUnavailable, maxSurge, readinessProbe, PodDisruptionBudget, HPA

**모범 답변**:
무중단 배포의 핵심은 Rolling Update 전략과 readinessProbe의 조합입니다. Rolling Update를 사용할 때 `maxUnavailable: 0, maxSurge: 1`로 설정하는 이유는, 기존 Pod를 종료하기 전에 반드시 새 Pod가 먼저 준비 완료 상태가 되도록 강제하기 위해서입니다. 이 설정 없이 기본값을 쓰면 배포 중에 가용 Pod 수가 일시적으로 줄어들 수 있습니다. readinessProbe는 그 자체로 무중단 배포의 게이트 역할을 합니다. 새 Pod가 뜨더라도 readinessProbe가 성공하기 전까지는 Service의 엔드포인트에 등록되지 않기 때문에, 애플리케이션이 초기화 중인 상태에서 트래픽이 유입되는 문제를 차단할 수 있습니다. 카테노이드에서 채팅 서버를 운영할 때 초기화 시간이 있는 서비스에 readinessProbe를 적용하지 않으면 503이 간헐적으로 발생하는 것을 경험했고, 그 이후부터는 모든 서비스에 readinessProbe를 기본으로 붙이는 것을 팀 컨벤션으로 정착시켰습니다. 노드 점검이나 스케일다운 상황에서는 PodDisruptionBudget을 활용해 최소 가용 Pod 수를 보장합니다. PDB는 eviction API 수준에서 동작하는 안전장치로, `kubectl drain` 실행 시 minAvailable 조건을 위반하는 eviction은 자동으로 차단됩니다. HPA의 CPU 임계치는 이론값이 아니라 실측값으로 잡는 것이 중요합니다. k6로 부하 테스트를 수행해 트래픽이 점진적으로 증가할 때 CPU 사용률이 어떻게 변화하는지 p95/p99 기준으로 측정한 뒤, 그 값의 70~80% 수준에서 임계치를 설정했습니다. 너무 낮게 잡으면 불필요한 scale-out이 잦아지고, 너무 높게 잡으면 트래픽 급증 시 scale-out 반응 전에 서비스가 포화 상태가 됩니다.

**꼬리 질문 예시**:
- readinessProbe와 livenessProbe의 차이는 무엇인가요?
- Canary 배포와 Rolling Update의 차이는 언제 선택 기준이 되나요?
- HPA가 scale-out하는 동안 트래픽 급증이 오면 어떻게 대응하나요?
- 실제로 배포 중 장애가 났을 때 rollback은 어떻게 했나요? → `kubectl rollout undo` 또는 ArgoCD 이전 리비전 sync
- 노드 점검 시 Pod 가용성은 어떻게 보장하나요? → PodDisruptionBudget

**면접 세션 피드백 (2026-03-28 4회차)**:
- 잘한 점: readinessProbe 실무 설계, HPA CPU 기준 판단 근거, k6 p95/p99 임계치 연결
- 보완: Rollback(ArgoCD sync), PDB(minAvailable), maxUnavailable 정확한 정의

**면접 세션 피드백 (2026-03-30 1회차)**:
- ArgoCD 롤백 언급은 올바름. kubectl rollout undo 명령어 누락 — 반드시 암기
- maxSurge는 rollback과 무관 (rolling update 배포 전략 설정). rollback은 이전 ReplicaSet scale up
- PDB는 eviction 게이팅 장치 — "자동으로 Pod 증설"이 아니라 "eviction API 호출 시 minAvailable 조건 위반 시 차단"

---

## Pod, Service, Deployment 역할 차이

**난이도**: 기초

**핵심 키워드**: Pod(ephemeral, 컨테이너 단위), Service(ClusterIP, Label Selector, stable endpoint), Deployment(ReplicaSet, desired state)

**모범 답변 방향**:

Pod는 Kubernetes에서 배포 가능한 가장 작은 단위로, 하나 이상의 컨테이너를 묶어 실행합니다. Pod의 가장 중요한 특성은 ephemeral, 즉 휘발성입니다. 재시작할 때마다 새로운 IP 주소가 발급되기 때문에 직접 IP로 접근하는 방식은 신뢰할 수 없습니다. 이 문제를 해결하는 것이 Service입니다. Service는 Label Selector를 통해 살아있는 Pod를 동적으로 찾아 라우팅하기 때문에, Pod가 교체되거나 IP가 바뀌어도 Service 주소는 항상 고정됩니다. Service 타입은 접근 범위에 따라 세 가지로 나뉩니다. ClusterIP는 클러스터 내부에서만 접근 가능한 기본 타입이고, NodePort는 노드의 IP와 특정 포트로 외부에서 접근할 수 있게 합니다. LoadBalancer는 클라우드 환경에서 외부 로드밸런서를 프로비저닝해 외부 트래픽을 클러스터 안으로 진입시키는 방식입니다. Deployment는 원하는 Pod 개수(replica count)를 선언하면 그 상태를 유지하는 desired state 관리자입니다. 내부적으로 ReplicaSet을 통해 Pod 수를 보장하고, Rolling Update와 Rollback도 Deployment 수준에서 처리됩니다.

**꼬리 질문 예시:**
- "ClusterIP와 LoadBalancer의 차이는?"
- "Deployment 없이 Pod를 직접 배포하면 어떤 문제가 있나요?"
- "ReplicaSet과 Deployment의 관계는?"

**면접 세션 피드백 (2026-04-01 3회차)**:
- Pod 휘발성과 Service 필요성 연결 정확
- 보완: Label Selector 메커니즘, Service 타입 3가지 구분 추가 필요
- Deployment를 "차트"로 표현 → "manifest/spec"이 정확 (Helm 차트와 혼동 주의)

---

## Kubernetes Rollback 처리와 PodDisruptionBudget(PDB)

**난이도**: 중급

**핵심 키워드**: kubectl rollout undo, ReplicaSet, ArgoCD sync, PDB, minAvailable, maxUnavailable, kubectl drain, eviction API

**모범 답변 방향**:

롤백에는 두 가지 방법이 있습니다. ArgoCD를 사용하는 환경에서는 이전 Git 리비전으로 sync를 하면 자동으로 이전 상태로 롤백됩니다. kubectl을 직접 쓰는 경우에는 `kubectl rollout undo deployment/{name}` 명령으로 즉시 이전 ReplicaSet으로 롤백할 수 있고, 특정 버전으로 되돌리고 싶다면 `--to-revision=N` 플래그를 함께 사용합니다. PodDisruptionBudget은 노드 점검이나 스케일다운 시 Pod 가용성을 보장하는 안전장치인데, 동작 방식을 정확히 이해하는 것이 중요합니다. `kubectl drain` 실행 시 내부적으로 eviction API가 호출되는데, 이때 PDB의 minAvailable 조건을 검사해서 조건 위반이 예상되면 eviction 자체를 거부합니다. 즉, PDB는 Pod를 자동으로 늘려주는 것이 아니라 eviction 요청 자체를 게이팅하는 역할을 합니다. 새 노드에서 대체 Pod가 Ready 상태가 된 이후에야 해당 Pod의 eviction이 허용됩니다. minAvailable과 maxUnavailable은 서로 대칭적인 표현으로, minAvailable: 1은 항상 최소 1개 Pod가 살아있어야 한다는 뜻이고, maxUnavailable: 1은 최대 1개까지 중단을 허용한다는 뜻입니다.

**꼬리 질문 예시**:
- "rollout undo 후 정상 여부를 어떻게 확인하나요?" → `kubectl rollout status`, `kubectl get pods`, 모니터링 지표 확인
- "PDB의 minAvailable과 maxUnavailable의 차이는?" → minAvailable: 최소 가용 Pod 수 (절대값 또는 %), maxUnavailable: 최대 중단 허용 Pod 수
- "drain 중 PDB 조건을 영원히 못 만족하면 어떻게 하나요?" → `--disable-eviction` 또는 `--force` 플래그 (위험 — 확인 필요), 또는 PDB를 임시 삭제

**면접 세션 피드백 (2026-03-30 1회차)**:
- 잘한 점: ArgoCD 롤백 언급, preStop + WebSocket active connection 폴링 패턴 (실무 경험 기반 강점)
- 보완: `kubectl rollout undo` 명령어 미언급. maxSurge와 rollback 개념 혼동. PDB가 eviction을 게이팅한다는 메커니즘 미설명

> 출처: Kubernetes 공식 문서 - https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

---

## WebSocket 서버를 배포할 때 기존 커넥션을 어떻게 보호하나요?

**난이도**: 심화

**핵심 키워드**: terminationGracePeriodSeconds, preStop hook, connection tracker, graceful shutdown, Server.Shutdown()

**모범 답변 방향**:

WebSocket 서버를 안전하게 교체하려면 두 단계로 접근합니다. 기본 패턴은 `terminationGracePeriodSeconds`와 `preStop: sleep 5`를 조합하는 것입니다. sleep 5가 필요한 이유는 Pod 종료 신호가 발생한 뒤 ingress controller와 kube-proxy가 Service endpoints에서 해당 Pod를 제거하는 데 몇 초의 전파 지연이 있기 때문입니다. 이 시간 동안 신규 요청이 종료 중인 Pod로 유입될 수 있어, preStop에서 짧게 대기해 전파가 완료될 때까지 기다려야 합니다. 심화 패턴은 애플리케이션이 WebSocket 커넥션 수를 atomic 카운터로 직접 추적하고, preStop hook이 `/status` 엔드포인트를 1초마다 폴링해서 active_connections가 0이 되는 순간 즉시 종료하는 방식입니다. 이렇게 하면 평균 세션 시간을 예측해서 무조건 기다리는 것보다 훨씬 효율적으로 Pod를 교체할 수 있습니다. 주의할 점은 Go의 `net/http Server.Shutdown()`이 WebSocket 연결을 추적하지 않는다는 것입니다. WebSocket은 HTTP Upgrade 이후 hijacked 연결이 되어 서버의 관리 범위를 벗어나기 때문에, 별도 atomic 카운터를 구현해야 합니다. `terminationGracePeriodSeconds`는 심화 패턴에서도 여전히 필요하고, 커넥션이 끝까지 끊기지 않는 비정상 클라이언트가 있을 경우의 안전망 역할을 합니다.

**꼬리 질문 예시**:
- "Go의 `http.Server.Shutdown()`으로 충분하지 않나요?" → WebSocket은 HTTP Upgrade 이후 hijacked 연결이라 Shutdown()이 추적하지 않음. 별도 atomic 카운터 구현 필요.
- "커넥션이 하나도 없어야 종료한다면, 비정상 클라이언트가 연결을 안 끊으면 영원히 기다리나요?" → `terminationGracePeriodSeconds`가 상한선. 초과 시 SIGKILL.
- "Istio를 쓰면 이 문제가 자동으로 해결되나요?" → Istio v1.12+의 `EXIT_ON_ZERO_ACTIVE_CONNECTIONS` 기능으로 Envoy가 자동 처리.

**구현 핵심 코드:**
```go
// preStop hook이 폴링할 엔드포인트
func statusHandler(tracker *WSTracker) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]any{
            "active_connections": tracker.count.Load(),
        })
    }
}
```

```yaml
# preStop hook: 커넥션 0 될 때까지 폴링
preStop:
  exec:
    command:
    - /bin/sh
    - -c
    - |
      while [ $(curl -s localhost:8080/status | jq '.active_connections') -gt 0 ]; do
        sleep 1
      done
```

> 출처: https://oneuptime.com/blog/post/2026-02-09-prestop-hooks-zero-connection-drop/view

---

## Istio가 무엇인지 설명해주시고, Kubernetes NetworkPolicy와 어떤 차이가 있는지 말씀해 주세요.

**난이도**: 중급

**핵심 키워드**: Service Mesh, Envoy sidecar, Data Plane, Control Plane, Istiod, mTLS, L4 vs L7

**모범 답변 (1분 이상 말하기 형태)**:
> Istio는 Kubernetes 환경에서 마이크로서비스 간 통신을 코드 수정 없이 제어하는 Service Mesh 플랫폼입니다. 구조는 Data Plane과 Control Plane으로 나뉩니다. Data Plane은 각 Pod에 Envoy Proxy를 sidecar로 자동 주입해서 모든 인바운드·아웃바운드 트래픽을 투명하게 가로챕니다. Control Plane인 Istiod는 VirtualService 같은 Istio 리소스를 Envoy 설정으로 변환해 배포하고, CA 역할로 각 워크로드에 인증서를 발급합니다. Kubernetes의 NetworkPolicy는 IP와 포트 기반의 L4 레이어에서만 트래픽을 허용·차단합니다. "이 포트로 오는 트래픽은 허용" 수준입니다. 반면 Istio의 mTLS는 L7 레이어에서 서비스 신원 자체를 기반으로 인증하기 때문에, "이 인증서를 가진 서비스의 요청만 허용"이 가능하고 통신이 자동으로 암호화됩니다. 또한 Istio는 트래픽 라우팅, 서킷 브레이커, 분산 트레이싱, 가중치 기반 Canary 배포도 코드 수정 없이 리소스 선언만으로 적용할 수 있다는 점이 큰 차이입니다.

**꼬리 질문 예시**:
- Envoy sidecar는 어떻게 자동으로 주입되나요?
- mTLS의 STRICT 모드와 PERMISSIVE 모드 차이는 무엇인가요?
- Istio 없이 서킷 브레이커를 구현하려면 어떻게 해야 하나요?

> 출처: https://velog.io/@beryl/Istio-%EA%B0%9C%EB%85%90

---

## Istio에서 mTLS가 자동으로 동작하는 원리를 설명해주세요.

**난이도**: 심화

**핵심 키워드**: Istiod, SPIFFE, X.509, PeerAuthentication, STRICT/PERMISSIVE, Zero Trust

**모범 답변 (1분 이상 말하기 형태)**:
> Istio의 mTLS는 상호 TLS 인증으로, 서비스 A와 B가 통신할 때 서로의 신원을 인증서로 검증하고 통신을 암호화하는 방식입니다. 동작 원리는 Istiod가 CA 역할을 해서 각 워크로드에 SPIFFE 표준 기반의 X.509 인증서를 발급하는 것에서 시작합니다. 각 Pod의 Envoy sidecar가 이 인증서를 받아서 다른 서비스와 통신할 때 TLS 핸드셰이크를 자동으로 수행합니다. 애플리케이션 코드는 여전히 HTTP로 localhost에 요청하는 것처럼 작성하면 되고, Envoy가 투명하게 가로채서 mTLS로 변환합니다. PeerAuthentication 리소스에서 mode를 STRICT로 설정하면 인증서가 없는 일반 HTTP 요청은 자동으로 차단됩니다. PERMISSIVE 모드는 mTLS와 HTTP를 모두 허용하는데, 기존 서비스를 Istio로 점진적으로 마이그레이션할 때 과도기적으로 사용합니다. 이를 통해 클러스터 내부 통신도 Zero Trust 보안 모델로 구성할 수 있고, 개발자는 TLS 코드를 한 줄도 작성하지 않아도 됩니다.

**꼬리 질문 예시**:
- SPIFFE가 무엇인지 아시나요?
- 인증서가 만료되면 어떻게 갱신되나요? (Istiod가 자동 갱신)
- Istio mTLS와 애플리케이션 레벨 TLS를 동시에 쓰면 이중 암호화가 되나요?

> 출처: https://oneuptime.com/blog/post/2026-01-07-istio-mtls-security/view

---

## Istio의 VirtualService와 DestinationRule을 사용해서 Canary 배포를 어떻게 구현하나요?

**난이도**: 심화

**핵심 키워드**: VirtualService, DestinationRule, subset, weight, 가중치 기반 라우팅, 헤더 기반 라우팅

**모범 답변 (1분 이상 말하기 형태)**:
> Istio에서 Canary 배포는 VirtualService와 DestinationRule을 조합해서 구현합니다. 먼저 DestinationRule로 동일한 Service를 버전별 subset으로 나눕니다. v1 라벨을 가진 Pod는 v1 subset, v2 라벨을 가진 Pod는 v2 subset으로 분류합니다. 그런 다음 VirtualService에서 가중치를 지정합니다. 처음에는 v1에 90%, v2에 10%를 보내다가, v2가 안정적으로 동작하는 것을 확인하면 점진적으로 비율을 올립니다. Kubernetes 기본 기능만으로 Canary를 하려면 v1과 v2 Deployment의 replica 수 비율을 조정해야 하는데, 이는 파드 수와 트래픽 비율이 연동되어 세밀한 제어가 어렵습니다. Istio를 쓰면 replica 수와 무관하게 정확히 10%의 트래픽만 v2로 보낼 수 있습니다. 또한 헤더 기반 라우팅도 가능해서, x-version: v2 헤더를 가진 요청만 v2로 보내는 방식으로 QA 팀이 프로덕션 환경에서 새 버전을 먼저 테스트할 수도 있습니다.

**꼬리 질문 예시**:
- Istio 없이 Kubernetes에서 Canary 배포를 구현하면 어떤 한계가 있나요?
- 트래픽 비율을 바꿀 때 서비스 재시작 없이 적용이 되나요? (VirtualService 수정만으로 즉시 적용)
- 서킷 브레이커는 어떤 리소스로 설정하나요? (DestinationRule의 outlierDetection)

**핵심 YAML 패턴 암기:**
```yaml
# DestinationRule — subset 정의
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
spec:
  host: my-svc
  subsets:
    - name: v1
      labels: { version: stable }
    - name: v2
      labels: { version: canary }

# VirtualService — 트래픽 비율 + 헤더 기반 라우팅
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  hosts: [my-svc]
  http:
    # 헤더 기반: 내부 QA 테스터만 canary
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination: { host: my-svc, subset: v2 }
    # 가중치 기반: 일반 트래픽 90/10
    - route:
        - destination: { host: my-svc, subset: v1 }
          weight: 90
        - destination: { host: my-svc, subset: v2 }
          weight: 10
```

**면접 세션 피드백 (2026-04-20 3회차)**:
- 완전 미지 영역 (0/10) — 아래 내용 전체 암기 필요
- 핵심 차이점: 순수 K8s = 레플리카 수로만 비율 조절 / Istio = weight 선언으로 Pod 수와 무관하게 1% 단위 조절
- DestinationRule: subset 정의(label 기반) / VirtualService: weight + headers 조건
- 헤더 기반 라우팅으로 내부 테스터만 canary 버전 접근 가능

> 출처: https://gruuuuu.hololy.org/cloud/service-mesh-istio/

---

## Kubernetes 배포 전략 3가지(Rolling Update, Blue-Green, Canary)의 차이를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: Rolling Update, Blue-Green, Canary, 다운타임, 리소스 비용, 롤백 속도, ReadinessProbe

**비교표**:

| 전략 | 다운타임 | 리소스 비용 | 롤백 속도 | 특징 |
|---|---|---|---|---|
| Rolling Update | 없음 | 최대 2배 (순간) | 느림 (pod 재시작 필요) | 기본 전략, ReadinessProbe 통과 후 구 pod 제거 |
| Blue-Green | 없음 | 2배 (동시 유지) | 빠름 (트래픽 스위치만) | 두 환경 동시 운영, 검증 후 전환 |
| Canary | 없음 | 구+신 pod 합산 | 빠름 (구 버전 유지 중) | 일부 트래픽만 신 버전으로, 점진적 검증 |

**모범 답변 (3분 이상 말하기 형태)**:
> Rolling Update는 Kubernetes의 기본 배포 전략입니다. 구 버전 pod를 한꺼번에 종료하지 않고, 신 버전 pod를 하나씩 띄우면서 ReadinessProbe를 통과하면 구 pod를 제거하는 방식으로 진행됩니다. 배포 중에 구/신 버전 pod가 일시적으로 공존하기 때문에 다운타임이 없고, 리소스는 최대 2배까지 사용됩니다. 롤백 시에는 이전 ReplicaSet으로 롤백하면 되지만, pod가 다시 ReadinessProbe를 통과해야 하므로 즉각적이진 않습니다.
>
> Blue-Green은 구 버전(Blue)과 신 버전(Green) 두 환경을 동시에 완전히 띄워두는 전략입니다. Green 환경에서 충분히 검증한 후 Service의 selector를 Blue에서 Green으로 바꿔서 트래픽을 전환합니다. 트래픽 전환이 즉각적이라 롤백도 selector만 되돌리면 되어 매우 빠릅니다. 단점은 두 환경을 동시에 유지해야 하므로 리소스 비용이 2배로 고정된다는 점입니다.
>
> Canary는 신 버전에 트래픽의 일부(예: 10%)만 보내면서 점진적으로 검증하는 전략입니다. 문제가 없으면 비율을 점차 높이고, 문제가 생기면 구 버전이 살아있으므로 즉시 트래픽을 원래대로 되돌릴 수 있습니다. 새 기능의 영향을 최소화하면서 검증하고 싶을 때 적합합니다. Istio 없이 순수 Kubernetes로 구현하면 replica 수 비율로만 트래픽을 조정해야 해서 정밀한 제어가 어렵고, Istio의 VirtualService를 쓰면 replica 수와 무관하게 정확한 비율 조정이 가능합니다.

**꼬리 질문 예시**:
- Blue-Green에서 DB 스키마 마이그레이션은 어떻게 처리하나요?
- Canary 배포 시 특정 사용자(예: 내부 직원)에게만 신 버전을 보여주려면?

**면접 세션 피드백 (2026-04-13 2회차)**:
- 잘한 점: 3가지 전략의 다운타임/리소스/롤백 비교 구조적으로 명확. ReadinessProbe 역할 자연스럽게 연결.
- 보완: Blue-Green 롤백이 빠른 이유 명시 ("pod 재시작 없이 Service selector 전환만"). Rolling Update 롤백이 느린 이유 ("pod가 다시 ReadinessProbe 통과해야 함").

---

## ReadinessProbe / LivenessProbe / StartupProbe의 역할 차이를 설명해주세요.

**난이도**: 기초

**핵심 키워드**: StartupProbe 선행 실행, ReadinessProbe → Service 엔드포인트 제거, LivenessProbe → kubelet 재시작

**실행 순서**: StartupProbe 성공 → LivenessProbe + ReadinessProbe 시작

| Probe | 실패 시 동작 | 목적 |
|---|---|---|
| StartupProbe | 없음 (대기, 성공 전까지 다른 Probe 비활성) | 느린 시작 앱 보호 |
| ReadinessProbe | Service 엔드포인트에서 제거 (트래픽 차단, pod 유지) | 트래픽 수신 준비 확인 |
| LivenessProbe | kubelet이 pod 재시작 | 런타임 장애 감지 |

**모범 답변 (3분 이상 말하기 형태)**:
> 세 Probe는 실행 시점과 실패 시 동작이 다릅니다. StartupProbe가 가장 먼저 실행되고, 성공하기 전까지 LivenessProbe와 ReadinessProbe가 동작하지 않습니다. StartupProbe는 느리게 시작하는 애플리케이션을 위한 것입니다. JVM 기반 서비스처럼 초기 클래스 로딩에 30-60초가 걸리는 경우, LivenessProbe가 응답 없음으로 판정해서 pod를 재시작하는 무한 루프를 막습니다. StartupProbe가 통과되면 이후부터 LivenessProbe와 ReadinessProbe가 함께 실행됩니다.
>
> ReadinessProbe는 트래픽을 받을 준비가 됐는지 체크합니다. 실패하면 kubelet이 해당 pod를 Service의 엔드포인트 목록에서 제거해서 트래픽을 차단합니다. 중요한 점은 pod 자체는 재시작하지 않는다는 것입니다. DB 연결이 끊겼거나 외부 의존 서비스가 응답 없을 때 일시적으로 트래픽만 차단하고, 복구되면 자동으로 엔드포인트에 다시 등록됩니다. Rolling Update에서 이 동작이 핵심적인데, 신 pod의 ReadinessProbe가 통과해야만 구 pod를 제거하기 때문에 무중단이 보장됩니다.
>
> LivenessProbe는 운영 중에 지속적으로 실행되며 메모리 누수나 데드락으로 프로세스가 응답하지 못하는 상태를 감지합니다. 설정된 연속 실패 횟수를 초과하면 kubelet이 pod를 재시작합니다. 카테노이드에서 채팅 서버 운영 중 특정 조건에서 goroutine 데드락이 발생해 프로세스가 멈추는 경우가 있었는데, LivenessProbe 덕분에 자동 재시작되어 수동 개입 없이 복구됐습니다.

**꼬리 질문 예시**:
- ReadinessProbe가 일시적으로 실패했다가 다시 성공하면 트래픽이 자동으로 다시 들어오나요? (네, 엔드포인트에 자동 재등록)
- LivenessProbe와 ReadinessProbe를 같은 엔드포인트로 설정하면 어떤 문제가 생기나요?

**면접 세션 피드백 (2026-04-13 2회차)**:
- 잘한 점: ReadinessProbe·LivenessProbe 핵심 역할 방향 맞음.
- 보완: StartupProbe = "느린 시작 앱 보호, 성공 전 다른 Probe 비활성". ReadinessProbe 실패 = pod 재시작 아닌 **Service 엔드포인트 제거**. "kubectl이 재시작" → `kubelet`이 재시작.

---
