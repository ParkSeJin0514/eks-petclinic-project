# 05. EKS Monitoring Stack (Prometheus & Grafana)

## 📋 개요

Grafana Cloud와 Alloy를 사용하여 EKS 클러스터 모니터링을 구축합니다.

## ☁️ Grafana Cloud 설정

### 1. Grafana Cloud 계정 생성
- https://grafana.com 에서 무료 계정 생성
- Prometheus 토큰 생성 및 저장

### 2. Helm 저장소 추가

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 3. Alloy 설치

**values.yaml**:
```yaml
controller:
  type: 'daemonset'

alloy:
  configMap:
    content: |
      prometheus.remote_write "grafana_cloud" {
        endpoint {
          url = "<your-prometheus-endpoint>"
          basic_auth {
            username = "<your-username>"
            password = "<your-password>"
          }
        }
      }

      discovery.kubernetes "nodes" {
        role = "node"
      }

      prometheus.scrape "nodes" {
        targets = discovery.kubernetes.nodes.targets
        forward_to = [prometheus.remote_write.grafana_cloud.receiver]
      }

      discovery.kubernetes "pods" {
        role = "pod"
      }

      prometheus.scrape "pods" {
        targets = discovery.kubernetes.pods.targets
        forward_to = [prometheus.remote_write.grafana_cloud.receiver]
      }

      discovery.kubernetes "kube_state_metrics" {
        role = "service"
        namespaces {
          names = ["kube-system"]
        }
      }

      prometheus.scrape "kube_state_metrics" {
        targets = discovery.relabel.kube_state_metrics.output
        forward_to = [prometheus.remote_write.grafana_cloud.receiver]
      }
```

```bash
# Alloy 설치
helm install alloy grafana/alloy -f values.yaml -n alloy --create-namespace

# 설치 확인
kubectl get pods -n alloy
```

### 4. kube-state-metrics 설치

```bash
kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/download/v2.10.0/kube-state-metrics-v2.10.0.yaml
```

## 📊 Grafana 대시보드

Grafana Cloud에서 사전 구성된 대시보드 가져오기:
- Kubernetes Cluster Monitoring
- Kubernetes Pod Monitoring
- Node Exporter Full

## ✅ 검증

```bash
# Alloy Pod 로그 확인
kubectl logs -n alloy -l app.kubernetes.io/name=alloy

# 메트릭 수집 확인 (Grafana Cloud)
```

## 📚 참고 자료

- [Grafana Alloy](https://grafana.com/docs/alloy/)
- [Kubernetes Monitoring](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/)

## 🎯 다음 단계

[06. Load Testing](./06-Load-Testing-K6.md)