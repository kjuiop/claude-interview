---
tags: [kubernetes, k8s, deployment, hpa, devops]
related: [distributed-systems, golang]
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
