# ArgoCD 오류 해결을 위한 Metrics Addon 교체

## kubectl 버전 업그레이드
~~~
cd ~
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.36.4/2026-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
~~~

## ArgoCD 애플리케이션 동기화 실패(Unknown) 문제 해결
### 문제의 원인
 - EKS에서 자체 ArgoCD 서비스를 설치하도록 변경했는데
 - 이떄 표준 upstream metrics server가 아닌 자체 meterics server를 설치한 것으로 인해
 - Application 배포시 동기화 실패(Unknown) 에러 발생

### 해결 방법
 - 관리형 애드온  metric server를 제거하고 표준 upstream metrics-server를 설치

### 해결 단계
- 기존 관리형 애드온 metrics 서버 제거
~~~
aws eks delete-addon \
  --region ap-northeast-2 \
  --cluster-name demo00 \
  --addon-name metrics-server

aws eks wait addon-deleted \
  --region ap-northeast-2 \
  --cluster-name demo00 \
  --addon-name metrics-server
~~~

### 관리형 metric addon 삭제 확인 : 여기서는 Not Found 떠야 함
~~~
kubectl -n kube-system get deployment metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
~~~

### 표준 upstream metrics server 설치 후 ArgoCD 재시작
~~~
# 공식 upstream metrics-server 설치
kubectl apply -f  https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml

# ArgoCD 재시작
kubectl -n argocd rollout restart statefulset/argocd-application-controller
kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=180s
kubectl -n argocd get applications
~~~

### metric server의 Protobuf 비교 스크립트 이 값이 200, 200이 나와야 함
~~~
(
  # 컨트롤러의 서비스 계정과 현재 클러스터 주소
  SA=$(kubectl -n argocd get pod \
    -l app.kubernetes.io/name=argocd-application-controller \
    -o jsonpath='{.items[0].spec.serviceAccountName}') || exit 1

  SERVER=$(kubectl config view --minify \
    -o jsonpath='{.clusters[0].cluster.server}') || exit 1

  # 서비스 계정의 단기 토큰 발급
  TOKEN=$(kubectl -n argocd create token "$SA" \
    --duration=10m) || exit 1

  # 클러스터 CA 인증서 준비
  CA_FILE=$(mktemp)
  trap 'rm -f "$CA_FILE"' EXIT

  kubectl -n argocd get configmap kube-root-ca.crt \
    -o jsonpath='{.data.ca\.crt}' > "$CA_FILE" || exit 1

  curl -sS --max-time 30 --cacert "$CA_FILE" \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Accept: application/json' \
    -o /dev/null -w 'JSON HTTP=%{http_code}\n' \
    "$SERVER/openapi/v2"

  curl -sS --max-time 30 --cacert "$CA_FILE" \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Accept: application/com.github.proto-openapi.spec.v2@v1.0+protobuf' \
    -o /dev/null -w 'Protobuf HTTP=%{http_code}\n' \
    "$SERVER/openapi/v2"
)
~~~

3.4 200, 200이 나왔다면 자동으로 문제해결 되었을 것임.
