EKS → Prometheus → Grafana
민호님이 서포트해주셔서 편하게 Alloy 연결 완료
Final Project 때 이대로 진행하면 될 것 같음
Grafana Cloud와 Grafana Local 따로 설명
Grafana Cloud
기본 설정
여기서는 Grafana Cloud로 진
Prometheus 토큰이 필요하니 꼭 어딘가에 저장
공개된 곳에 올리면 악용 대상이니 참고!
Helm Chart 설치
# Grafana Helm 저장소 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
# 네임스페이스 생성
kubectl create namespace alloy
Secret으로 변수를 설정해야 하지만 오류가 계속 떠서 하드 코딩으로 대체
controller:
type: 'daemonset'
alloy:
configMap:
content: |
// kube-state-metrics 스크래핑
discovery.kubernetes "kube_state_metrics" {
role = "service"
namespaces {
EKS  Prometheus  Grafana 1

names = ["kube-system"]
}
}
discovery.relabel "kube_state_metrics" {
targets = discovery.kubernetes.kube_state_metrics.targets
rule {
source_labels = ["__meta_kubernetes_service_name"]
regex = "kube-state-metrics"
action = "keep"
}
}
prometheus.scrape "kube_state_metrics" {
targets = discovery.relabel.kube_state_metrics.output
scrape_interval = "60s"
forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}
// Kubelet/cAdvisor 메트릭 (Container 메트릭)
discovery.kubernetes "nodes" {
role = "node"
}
discovery.relabel "kubelet" {
targets = discovery.kubernetes.nodes.targets
rule {
target_label = "__address__"
replacement = "kubernetes.default.svc:443"
}
rule {
source_labels = ["__meta_kubernetes_node_name"]
regex = "(.+)"
target_label = "__metrics_path__"
replacement = "/api/v1/nodes/$1/proxy/metrics/cadvisor"
EKS  Prometheus  Grafana 2

}
}
prometheus.scrape "kubelet" {
targets = discovery.relabel.kubelet.output
scheme = "https"
bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/to
ken"
tls_config {
insecure_skip_verify = true
}
scrape_interval = "60s"
forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}
// 모든 Pod 발견 (추가 모니터링용)
discovery.kubernetes "pods" {
role = "pod"
}
prometheus.scrape "pods" {
targets = discovery.kubernetes.pods.targets
scrape_interval = "60s"
forward_to = [prometheus.remote_write.grafana_cloud.receiver]
}
prometheus.remote_write "grafana_cloud" {
endpoint {
url = "https://prometheus-prod-49-prod-ap-northeast-0.grafana.net/
api/prom/push"
basic_auth {
username = "2637655"
password = "password"
}
}
}
EKS  Prometheus  Grafana 3

# Helm 차트 설치
helm install alloy grafana/alloy -n alloy -f values.yaml
수정하고 난 후 업데이트
helm upgrade alloy grafana/alloy -n alloy -f values.yaml
kubectl rollout restart daemonset/alloy -n alloy
상태 확인
# Pod 상태 확인
kubectl get pods -n alloy
# DaemonSet 확인
kubectl get daemonset -n alloy
# 로그 확인
kubectl logs -n alloy -l app.kubernetes.io/name=alloy
다양한 매트릭 수집
기본만 깔면 매트릭이 수집되는게 한정적
Node Exporter를 설치해서 더 많은 매트릭이 수집되게 변경
Helm으로 kube-state-metrics 설치
# Prometheus Community Helm 저장소 추가
helm repo add prometheus-community https://prometheus-community.gith
ub.io/helm-charts
helm repo update
# kube-state-metrics 설치
helm install kube-state-metrics prometheus-community/kube-state-metric
s -n kube-system
Node Exporter 설치
EKS  Prometheus  Grafana 4

# node-exporter.yaml 생성
vi node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
name: node-exporter
namespace: kube-system
spec:
selector:
matchLabels:
app: node-exporter
template:
metadata:
labels:
app: node-exporter
spec:
hostNetwork: true
hostPID true
containers:
- name: node-exporter
image: prom/node-exporter:latest
ports:
- containerPort: 9100
name: metrics
args:
- --path.procfs=/host/proc
- --path.sysfs=/host/sys
- --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|et
c)($$|/)
volumeMounts:
- name: proc
mountPath: /host/proc
readOnly: true
- name: sys
mountPath: /host/sys
readOnly: true
volumes:
EKS  Prometheus  Grafana 5

- name: proc
hostPath:
path: /proc
- name: sys
hostPath:
path: /sys
수정했으니 업그레이드
helm upgrade alloy grafana/alloy -n alloy -f values.yaml
kubectl rollout restart daemonset/alloy -n alloy
상태 확인
# Pod 재시작 확인
kubectl get pods -n alloy
# 로그 확인
kubectl logs -n alloy -l app.kubernetes.io/name=alloy --tail=50
Grafana Local
기본 설정
여기서는 Grafana Local 사용
매트릭이 감당이 안되서 Cloud로는 진행 불가
Prometheus  Grafana 설치
# Helm repo 등록
helm repo add prometheus-community https://prometheus-community.gith
ub.io/helm-charts
helm repo update
# 네임스페이스 생성
kubectl create namespace monitoring
EKS  Prometheus  Grafana 6

values-local.yaml 파일 생성
prometheus:
prometheusSpec:
additionalArgs:
- name: --web.enable-remote-write-receiver
scrapeInterval: 60s
retention: 2d
grafana:
enabled: true
adminUser: admin
adminPassword: <password>
service:
type: ClusterIP
설치 및 업데이트
# 설치
helm install kube-prometheus-stack prometheus-community/kube-promet
heus-stack \
-n monitoring -f values-local.yaml
# 업데이트
helm upgrade --install kube-prometheus-stack prometheus-community/ku
be-prometheus-stack \
-n monitoring -f values-local.yaml
포트포워딩
EKS는 Private Subnet에 있기 때문에 Bastion을 통해서 포트포워딩 실행
내 PC의 3000번 포트로 들어온 트래픽을, Bastion 서버를 통해 원격 서버의
localhost:3000으로 전달
구조
EKS  Prometheus  Grafana 7

[로컬 PC 브라우저] → localhost:3000
│
▼
SSH 터널 (-L 옵션)
│
▼
[Bastion 서버: 13.124.4.87]
│
▼
[EKS 내부 Pod Private Subnet)]
├─ Grafana 3000
└─ Prometheus 9090
로컬 PC에서 접속
ssh -i project.pem L 3000:localhost:3000 ubuntu@<Bastion IP
ssh -i project.pem L 9090:localhost:9090 ubuntu@<Bastion IP
터미널에서 포트포워딩 실행
# Prometheus UI
kubectl -n monitoring port-forward svc/kube-prometheus-stack-promethe
us 90909090
# Grafana UI
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana
300080
# Prometheus UI 백그라운드 실행)
nohup kubectl -n monitoring port-forward svc/kube-prometheus-stack-pro
metheus 90909090 /dev/null 2&1 &
# Grafana UI 백그라운드 실행)
nohup kubectl -n monitoring port-forward svc/kube-prometheus-stack-gra
fana 300080 /dev/null 2&1 &
EKS  Prometheus  Grafana 8

lsof -i 3000 # Grafana 포트 확인
lsof -i 9090 # Prometheus 포트 확인
접속
# Prometheus 접속
http://localhost:9090
# Grafana 접속
http://localhost:3000
EKS  Prometheus  Grafana 9