---
tags: [kubernetes, k8s, 면접질문, deployment, hpa]
related: [kubernetes/concepts]
---

# Kubernetes 면접 질문

→ [[home]] | [[topics/kubernetes/concepts]]

---

## Kubernetes에서 서비스 무중단 배포를 보장하기 위해 어떤 전략을 사용했나요? HPA는 어떤 기준으로 설정하나요?

**난이도**: 심화

**핵심 키워드**: Rolling Update, maxUnavailable, maxSurge, readinessProbe, PodDisruptionBudget, HPA

**모범 답변 방향**:
- Rolling Update 전략 + `maxUnavailable: 0, maxSurge: 1` 설정 이유 설명
- readinessProbe가 무중단 배포의 핵심인 이유 (없으면 초기화 중 트래픽 유입)
- PDB로 노드 점검 중 최소 가용 Pod 보장
- HPA: k6 부하 테스트로 CPU 임계치 측정 후 70~80% 기준 설정 (이력서 k6 경험 연결)

**꼬리 질문 예시**:
- readinessProbe와 livenessProbe의 차이는 무엇인가요?
- Canary 배포와 Rolling Update의 차이는 언제 선택 기준이 되나요?
- HPA가 scale-out하는 동안 트래픽 급증이 오면 어떻게 대응하나요?
- 실제로 배포 중 장애가 났을 때 rollback은 어떻게 했나요? → `kubectl rollout undo` 또는 ArgoCD 이전 리비전 sync
- 노드 점검 시 Pod 가용성은 어떻게 보장하나요? → PodDisruptionBudget

**면접 세션 피드백 (2026-03-28 4회차)**:
- 잘한 점: readinessProbe 실무 설계, HPA CPU 기준 판단 근거, k6 p95/p99 임계치 연결
- 보완: Rollback(ArgoCD sync), PDB(minAvailable), maxUnavailable 정확한 정의

> 출처: Kubernetes 공식 문서 - https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

---

## WebSocket 서버를 배포할 때 기존 커넥션을 어떻게 보호하나요?

**난이도**: 심화

**핵심 키워드**: terminationGracePeriodSeconds, preStop hook, connection tracker, graceful shutdown, Server.Shutdown()

**모범 답변 방향**:
- 기본 패턴: `terminationGracePeriodSeconds` + `preStop: sleep 5` (endpoints 제거 전파 대기)
- 심화 패턴: 애플리케이션이 커넥션 수를 atomic 카운터로 직접 추적 → `/status` 엔드포인트로 노출 → preStop hook이 0이 될 때까지 폴링 → 즉시 종료
- Go에서 `net/http Server.Shutdown()`은 **WebSocket(hijacked 연결) 추적 안 함** → 별도 트래커 필수
- `terminationGracePeriodSeconds`는 여전히 필요 (안전망 역할)

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
