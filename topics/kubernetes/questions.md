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
