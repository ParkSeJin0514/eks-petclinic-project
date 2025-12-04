K6
일정 기준치 이상까지 프로젝트를 완성해서 시간이 남아 K6를 기준으로 부하테스트
구성
HPA
HPA는 Kubernetes에서 가장 일반적으로 사용되는 오토 스케일링 기술로 애플리케이션
의 수평적인 replicas 수를 동적으로 조정하여 애플리케이션의 부하에 따라 자동으로 조
정하는 기능을 제공
HPA는 CPU 사용률, 메모리 사용률 또는 사용자 정의 지표를 기반으로 Pod
의 replicas 수를 조정
HPA 설정
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
name: petclinic-hpa
namespace: default
spec:
scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: petclinic # ← 현재 실제 WAS Deployment 이름
minReplicas: 2
maxReplicas: 4
metrics:
- type: Resource
resource:
name: cpu
target:
type: Utilization
averageUtilization: 50
behavior:
scaleDown:
stabilizationWindowSeconds: 300
K6 1

scaleUp:
stabilizationWindowSeconds: 0
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
name: static-web-hpa
namespace: default
spec:
scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: static-web # ← 실제 웹 NGINX Deployment 이름
minReplicas: 2
maxReplicas: 4
metrics:
- type: Resource
resource:
name: cpu
target:
type: Utilization
averageUtilization: 50
구성
K6
Ubuntu 기준으로 작성
kubectl get hpa -n petclinic 했을 때 unknown 뜨면치
# 기존 Metrics Server 완전 삭제
kubectl delete deployment metrics-server -n kube-system
kubectl delete service metrics-server -n kube-system
kubectl delete apiservice v1beta1.metrics.k8s.io
kubectl delete clusterrole system:metrics-server
kubectl delete clusterrolebinding system:metrics-server
kubectl delete rolebinding metrics-server-auth-reader -n kube-system
kubectl delete serviceaccount metrics-server -n kube-system
K6 2

# 새로운 Metrics Server 설치 (kubelet-insecure-tls 포함)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/release
s/latest/download/components.yaml
# Metrics Server Deployment 패치 (kubelet-insecure-tls 추가)
kubectl patch deployment metrics-server -n kube-system --type='json' -p
='[
{
"op": "add",
"path": "/spec/template/spec/containers/0/args/-",
"value": "--kubelet-insecure-tls"
}
]'
# Pod 재시작 확인
kubectl get pods -n kube-system -l k8s-app=metrics-server -w
K6 설치
# Ubuntu / Debian
sudo apt-get update && sudo apt-get install -y gnupg2 ca-certificates
curl -fsSL https://repo.k6.io/gpg.key | sudo gpg --dearmor -o /usr/share/ke
yrings/k6-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://
repo.k6.io/apt stable main" \
| sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install -y k6
K6 설정
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = {
scenarios: {
ramp: {
executor: 'ramping-vus',
startVUs: 0,
K6 3

stages: [
{ duration: '3m', target: 100 ,
{ duration: '5m', target: 200 ,
{ duration: '2m', target: 0 ,
],
gracefulRampDown: '30s',
},
},
thresholds: {
http_req_duration: ['p(95)800'],
http_req_failed: ['rate<0.01'],
},
};
const BASE_URL  __ENV.BASE_URL || 'https://www.psj0514.site';
const REQ_PATH  __ENV.REQ_PATH || '/petclinic/'; //  PATH 대신 REQ_PA
TH 사용
export default function () {
const res = http.get(`${BASE_URL$REQ_PATH`, {
headers: {
'User-Agent': 'k6-loadtest',
'Accept': 'text/html,application/xhtml+xml',
},
timeout: '30s',
});
check(res, {
'status 2xx/3xx': (r) ⇒ r.status  200 && r.status  400,
});
sleep(1);
}
K6 실행
k6 run test-domain.js
K6 4

테스트 결과
가볍게 부하가 걸리는지만 확인
Grafana로 확인
부하 걸기 전 Pod 수
부하 건 후 Pod
K6 5