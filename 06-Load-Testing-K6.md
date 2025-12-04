# 06. K6를 활용한 부하 테스트 및 HPA

## 📋 개요

K6 부하 테스트 도구를 사용하여 애플리케이션 성능을 테스트하고, HPA(Horizontal Pod Autoscaler)를 통한 자동 확장을 검증합니다.

## 📈 HPA 설정

### 1. Metrics Server 설치

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 설치 확인
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

### 2. HPA 생성

**petclinic-hpa.yaml**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: petclinic-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: petclinic
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
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

```bash
kubectl apply -f petclinic-hpa.yaml

# HPA 상태 확인
kubectl get hpa
```

## 🧪 K6 설치 및 테스트

### 1. K6 설치 (Ubuntu)

```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

### 2. 부하 테스트 스크립트

**load-test.js**:
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },  // 1분 동안 10명까지 증가
    { duration: '3m', target: 50 },  // 3분 동안 50명 유지
    { duration: '1m', target: 0 },   // 1분 동안 0으로 감소
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95%의 요청이 500ms 이내
    http_req_failed: ['rate<0.1'],    // 실패율 10% 미만
  },
};

export default function () {
  const res = http.get('http://<alb-dns>/');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

### 3. 테스트 실행

```bash
# 부하 테스트 실행
k6 run load-test.js

# HPA 상태 실시간 모니터링
watch kubectl get hpa
watch kubectl get pods -l app=petclinic
```

## 📊 결과 분석

### 테스트 메트릭
- **http_req_duration**: 응답 시간
- **http_req_failed**: 실패율
- **http_reqs**: 초당 요청 수

### HPA 동작 확인
```bash
# HPA 이벤트 확인
kubectl describe hpa petclinic-hpa

# Pod 스케일 히스토리
kubectl get events --sort-by='.lastTimestamp' | grep HorizontalPodAutoscaler
```

## 🎯 최적화 가이드

### Resource Requests/Limits 설정

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

### HPA 튜닝
- `averageUtilization`: CPU 목표 사용률 조정
- `stabilizationWindowSeconds`: 스케일 안정화 시간
- `scaleUp/scaleDown policies`: 확장/축소 속도 제어

## 📚 참고 자료

- [K6 Documentation](https://k6.io/docs/)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

---

## 🎉 프로젝트 완료!

모든 단계를 완료하셨습니다:
- ✅ EKS 클러스터 구성
- ✅ ALB Ingress 설정
- ✅ EFS 스토리지 연동
- ✅ ECR-EKS-RDS 통합
- ✅ 모니터링 스택 구축
- ✅ 부하 테스트 및 오토스케일링