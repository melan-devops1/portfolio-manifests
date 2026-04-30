# ArgoCD GitOps 설치 절차

> Phase 3 Epic 6 결과물. 이 디렉토리는 ArgoCD 자체의 설치 자료.
> ArgoCD가 watch하는 앱 manifest는 `apps/all/overlays/dev`에 있음.

## 사전 조건

- EKS 클러스터 (1.33+) 동작 중
- kubectl, helm 설치됨
- AWS Load Balancer Controller 설치됨 (Phase 3.4.6)
- portfolio-manifests Git 레포가 GitHub에 있음 (public 또는 ArgoCD가 접근 가능)

## 설치 순서

### Step 1: Helm 저장소 추가

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo
```

### Step 2: ArgoCD 설치

```bash
cd ~/projects/portfolio-manifests/infrastructure/argocd

# values.yaml 기준 설치
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.5.0 \
  --values values.yaml \
  --wait
```

`--wait`로 모든 Pod이 Ready될 때까지 대기 (~3-5분).

### Step 3: 설치 검증

```bash
# Pod 상태 확인 (5개 정도 보여야 정상)
kubectl get pods -n argocd

# 기대:
#   argocd-application-controller-0          1/1   Running
#   argocd-redis-xxx                         1/1   Running
#   argocd-repo-server-xxx                   1/1   Running
#   argocd-server-xxx                        1/1   Running
```

### Step 4: Ingress 적용 (별도 ALB 생성)

```bash
kubectl apply -f ingress.yaml

# ALB 생성 대기 (~2-3분)
kubectl get ingress argocd-ingress -n argocd -w

# ADDRESS 컬럼에 ALB DNS 보이면 완료:
#   k8s-argocd-argocdin-xxxxxx.ap-northeast-2.elb.amazonaws.com
# Ctrl+C로 빠져나옴
```

### Step 5: 초기 admin 비밀번호 추출

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
echo
```

이 비밀번호를 메모. **첫 로그인 후 변경 권장** (운영 표준).

### Step 6: UI 접근

```bash
# ALB URL 가져오기
ARGOCD_URL=$(kubectl get ingress argocd-ingress -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD UI: http://$ARGOCD_URL"
```

브라우저에서 출력된 URL 열고:
- Username: `admin`
- Password: Step 5에서 추출한 값

### Step 7: Application 등록 (portfolio-app watch 시작)

```bash
kubectl apply -f application.yaml
```

ArgoCD UI에서 `portfolio-app` Application 카드가 나타남.

### Step 8: 자동 sync 검증

Application 정책이 Auto + Self-heal이라 즉시 sync 시작:
- Status: `Syncing` → `Synced + Healthy`
- 자원 6개 자동 배포:
  - 3개 Deployment (product, order, payment)
  - 3개 Service (ClusterIP)
  - 3개 HPA

```bash
# CLI로 확인
kubectl get application portfolio-app -n argocd
# 기대 출력:
#   NAME            SYNC STATUS   HEALTH STATUS
#   portfolio-app   Synced        Healthy

# Pod 자동 배포 확인
kubectl get pods -l app.kubernetes.io/part-of=portfolio
```

### Step 9: 운영 검증 — Self-heal 동작 확인 (선택)

ArgoCD가 진짜 GitOps 패턴으로 작동하는지 검증:

```bash
# 1) 의도적으로 deployment 망가뜨리기
kubectl scale deployment product-service --replicas=0

# 2) 1~2분 기다리면 ArgoCD가 자동 복구
kubectl get deployment product-service -w
# replicas가 다시 2로 돌아옴 (HPA가 minReplicas=2이라)
```

이게 진짜 GitOps의 가치 — Git이 진실의 소스, 수동 변경은 자동 복구.

## 정리 순서 (destroy 전 필수)

⚠️ **순서 중요** — 안 지키면 VPC destroy 실패:

```bash
# 1) Application 먼저 삭제 (자원 정리 트리거)
kubectl delete -f application.yaml
sleep 30   # 자원 정리 대기

# 2) ArgoCD Ingress 삭제 (별도 ALB 정리)
kubectl delete -f ingress.yaml
sleep 60   # ENI deregister 대기

# 3) ArgoCD Helm 삭제
helm uninstall argocd -n argocd

# 4) 앱 자체 정리 (apps Ingress + 앱)
kubectl delete -k ../ingress
sleep 60
kubectl delete -k ../../apps/all/overlays/dev

# 5) ALB Controller, metrics-server, Terraform destroy
# (기존 destroy 절차)
```

## 트러블슈팅

### "Repository unauthorized" 에러
portfolio-manifests가 private 레포면 ArgoCD에 credential 등록 필요.
public 레포라면 무관.

### Application이 Out-of-Sync 그대로
- Git의 manifest 변경 후 ArgoCD가 즉시 감지 안 할 수 있음 (default polling 3분)
- UI에서 "Refresh" 버튼 또는 webhook 설정 (Phase 5+)

### Self-heal이 무한 복구 시도
ignoreDifferences 설정 확인 — HPA가 갱신하는 deployment.replicas는 무시되어야 함.

## 면접 답변용 포인트

> "Phase 3 Epic 6에서 ArgoCD를 Helm으로 설치하고 portfolio-manifests 레포를 GitOps 소스로 등록했습니다. Application CRD에 Auto sync + Self-heal + prune 정책을 적용해 진짜 GitOps 패턴(Git이 진실의 소스, 수동 변경 자동 복구)을 구현했습니다.
>
> ALB Ingress 설정 시 흔한 함정인 'redirect loop'를 미리 방지하기 위해 ArgoCD server에 `--insecure` 플래그를 박았고(ALB가 TLS termination, ArgoCD는 HTTP backend), 별도 ALB(group.name=argocd)로 앱용 ALB와 분리해 운영 책임 영역을 명확히 했습니다.
>
> 또 ignoreDifferences로 deployment.replicas를 selfHeal 대상에서 제외해 HPA의 자동 스케일링과 ArgoCD의 self-heal이 무한 충돌하는 것을 방지했습니다."