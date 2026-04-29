# portfolio-manifests

> 📘 전체 프로젝트 개요는 [portfolio-overview](https://github.com/melan-devops1/portfolio-overview)에서 확인하세요.

K8s manifest 저장소. ArgoCD가 이 레포를 watch해서 EKS에 자동 배포 (Phase 6 예정).

## 구조

```
apps/                     ← 마이크로서비스 manifest
  product-service/
    base/                 환경 무관 공통 정의
    overlays/
      dev/                환경별 차이 (replicas, resources 등)
infrastructure/           ← 클러스터 시스템 (Phase 4: Prometheus, Grafana 등)
argocd-apps/              ← ArgoCD Application CRD (Phase 6)
```

## Kustomize 패턴

각 서비스는 `base + overlays/<env>` 구조.
- **base**: 환경 무관 공통 (Deployment, Service)
- **overlays/<env>**: 환경별 patch (replicas, resources, configmap 등)

## 수동 배포 (Phase 3.4까지 — ArgoCD 도입 전)

```bash
# 적용 전 미리보기
kubectl kustomize apps/product-service/overlays/dev

# 적용
kubectl apply -k apps/product-service/overlays/dev

# 상태 확인
kubectl get pods -l app=product-service
kubectl logs -l app=product-service --tail=50

# 제거
kubectl delete -k apps/product-service/overlays/dev
```

## 도구

- **kubectl 1.33** + 빌트인 Kustomize v5
- 별도 `kustomize` CLI 설치 불필요 (`kubectl apply -k` 사용)

## 의사결정

- ADR-0018: Kustomize 채택 (vs Helm) — portfolio-infra 레포 참조