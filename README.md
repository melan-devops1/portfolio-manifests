# portfolio-manifests

> 📘 전체 프로젝트 개요는 [portfolio-overview](https://github.com/melan-devops1/portfolio-overview)에서 확인하세요.

K8s manifest 저장소. ArgoCD가 이 레포를 watch해서 EKS에 자동 배포.

## 구조

```
apps/                          ← 마이크로서비스 (자체 앱)
  product-service/
    base/                      Deployment + Service + HPA + kustomization
    overlays/dev/              dev 환경 차이 (replicas=2 patch)
  order-service/               동일 구조
  payment-service/             동일 구조
  all/overlays/dev/            3개 서비스 통합 overlay (ArgoCD가 watch)

argocd/                        ← ArgoCD 자체 설치 + portfolio-app Application
  values.yaml                  Helm chart values (v9.5.0, ArgoCD v3.3.6)
  application.yaml             ArgoCD Application — apps/all/overlays/dev watch
  ingress.yaml                 ArgoCD UI ALB Ingress
  README.md                    설치/검증/destroy 절차 가이드

infrastructure/                ← 클러스터 시스템
  ingress/                     앱용 ALB Ingress (path-based routing)

monitoring/                    ← Prometheus 스택
  kube-prometheus-stack/       ArgoCD Application + Helm values (chart v84.3.0)
  service-monitors/            서비스별 ServiceMonitor CRD (3개)
  prometheus-rules.yaml        PrometheusRule (HighErrorRate, HighLatencyP99, Pod*)
  alertmanager-config.yaml     AlertmanagerConfig (Slack severity 라우팅)
  grafana-ingress.yaml         Grafana UI ALB Ingress

logging/                       ← EFK 스택
  namespace.yaml               logging 네임스페이스
  elasticsearch.yaml           Elasticsearch StatefulSet (single node)
  kibana.yaml                  Kibana Deployment
  kibana-ingress.yaml          Kibana UI ALB Ingress
  fluent-bit/                  ArgoCD Application + Helm values (DaemonSet)

tracing/                       ← Jaeger + OTel
  application.yaml             ArgoCD Application — Jaeger Helm (chart v4.7.0)
  values.yaml                  all-in-one + memory storage + OTLP receiver
  jaeger-ingress.yaml          Jaeger UI ALB Ingress
```

## 배포 패턴

자체 앱은 Kustomize, 3rd party는 Helm (ADR-0018).

| 영역 | 패키징 | ArgoCD Application |
|---|---|---|
| product/order/payment-service | Kustomize | `argocd/application.yaml` (`apps/all/overlays/dev` watch) |
| kube-prometheus-stack | Helm chart | `monitoring/kube-prometheus-stack/application.yaml` |
| Jaeger | Helm chart | `tracing/application.yaml` |
| Fluent Bit | Helm chart | `logging/fluent-bit/application.yaml` |
| Elasticsearch / Kibana / Ingress / AlertmanagerConfig / PrometheusRule / ServiceMonitor | YAML 직접 | (ArgoCD 외부, `kubectl apply -f` 또는 `-k`) |

3rd party Helm 차트는 ArgoCD의 multi-source Application 패턴 — Helm repo + Git 레포 values.yaml 결합.

## Kustomize 패턴 (자체 앱)

각 서비스는 `base + overlays/<env>` 구조.
- **base**: 환경 무관 공통 (Deployment, Service, HPA, kustomization.yaml)
- **overlays/<env>**: 환경별 patch (replicas 등)

`base/kustomization.yaml`의 `images:` 필드는 CI가 자동 갱신:
```yaml
images:
  - name: portfolio/product-service
    newName: <ECR_REGISTRY>/portfolio/product-service
    newTag: <commit-sha>     # ← GitHub Actions가 main merge 시 자동 갱신
```

ArgoCD가 이 변경을 감지해 자동 sync.

## 설치 순서 (전체 사이클)

자세한 명령은 [`argocd/README.md`](./argocd/README.md) 참조.

1. EKS + ALB Controller + EBS CSI Driver IAM 준비 (portfolio-infra)
2. `helm install argocd` (값: `argocd/values.yaml`)
3. `kubectl apply -f argocd/ingress.yaml` (ArgoCD UI ALB)
4. `kubectl apply -f argocd/application.yaml` → 자체 앱 자동 sync
5. `kubectl apply -k infrastructure/ingress` (앱 ALB Ingress)
6. `kubectl apply -f monitoring/kube-prometheus-stack/application.yaml` → Prometheus/Grafana
7. `kubectl apply -f monitoring/service-monitors/` + `kubectl apply -f monitoring/prometheus-rules.yaml` + `kubectl apply -f monitoring/alertmanager-config.yaml`
8. `kubectl apply -f monitoring/grafana-ingress.yaml`
9. `kubectl apply -f logging/namespace.yaml` + `elasticsearch.yaml` + `kibana.yaml` + `kibana-ingress.yaml`
10. `kubectl apply -f logging/fluent-bit/application.yaml` → Fluent Bit DaemonSet
11. `kubectl apply -f tracing/application.yaml` + `kubectl apply -f tracing/jaeger-ingress.yaml`

## 수동 배포 (시연/디버깅)

ArgoCD를 거치지 않고 직접 배포할 때:

```bash
# 적용 전 미리보기
kubectl kustomize apps/all/overlays/dev

# 단일 서비스
kubectl apply -k apps/product-service/overlays/dev

# 3개 서비스 한 번에
kubectl apply -k apps/all/overlays/dev

# 상태 확인
kubectl get pods -l app.kubernetes.io/part-of=portfolio
kubectl logs -l app=product-service --tail=50

# 제거
kubectl delete -k apps/all/overlays/dev
```

ArgoCD가 켜져있으면 selfHeal이 작동하므로 `kubectl delete`로 지워도 자동 복구됨. 시연 후엔 Application부터 삭제.

## 도구

- **kubectl 1.33** + 빌트인 Kustomize v5
- 별도 `kustomize` CLI 설치 불필요 (`kubectl apply -k`)
- ArgoCD CLI는 옵션 (UI로 충분)

## 의사결정 (ADR)

- **ADR-0018** (Kustomize over Helm): 자체 앱은 Kustomize, 3rd party는 Helm
- **ADR-0019** (ALB Ingress): AWS Load Balancer Controller로 path-based routing
- **ADR-0021** (ArgoCD GitOps): auto-sync + selfHeal + ignoreDifferences(replicas)
- **ADR-0022** (Observability Metrics): ServiceMonitor + PrometheusRule
- **ADR-0024** (EFK): Elasticsearch + Fluent Bit + Kibana
- **ADR-0025** (Jaeger + OTel): traces only (metrics/logs는 별도 stack)
- **ADR-0026** (matcherStrategy=None): namespace prefix matcher 비활성으로 알림 라우팅 정상화
- **ADR-0027** (Manifest 적용 전략): Helm / kubectl / ArgoCD 경계

모든 ADR은 [portfolio-infra/docs/adr/](https://github.com/melan-devops1/portfolio-infra/tree/main/docs/adr) 참조.
