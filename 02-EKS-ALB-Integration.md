# 02. EKS - ALB 통합

## 📋 개요

AWS Application Load Balancer(ALB)를 EKS 클러스터와 통합하여 외부 트래픽을 Kubernetes 서비스로 라우팅합니다. ALB Ingress Controller를 사용하면 하나의 로드밸런서로 여러 서비스를 경로 기반으로 분리할 수 있습니다.

### 왜 ALB를 사용하는가?

- **Layer 7 로드밸런싱**: HTTP/HTTPS 경로 기반 라우팅 지원
- **비용 효율성**: 여러 서비스를 하나의 ALB로 처리 (서비스마다 NLB를 만들 필요 없음)
- **유연한 라우팅**: `/api/*`, `/auth/*`, `/static/*` 등 경로 기반 분리
- **AWS 통합**: ACM 인증서, WAF, CloudWatch 등과 자연스러운 연동

---

## 🔧 AWS Load Balancer Controller 설치

### 1. IAM Policy 생성

AWS Load Balancer Controller가 AWS 리소스를 관리하기 위한 권한을 생성합니다.

> **⚠️ 중요**: EKS 1.22 버전 이후부터는 추가 권한이 필요합니다. 반드시 버전을 확인하세요!

```bash
# v2.14.1 IAM Policy 다운로드
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json

# IAM Policy 생성
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

**참고 문서**:
- [AWS Load Balancer Controller 공식 문서](https://github.com/aws/eks-charts?tab=readme-ov-file#aws-load-balancer-controller)
- [EKS 버전별 요구사항](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

### 2. IRSA (IAM Roles for Service Accounts) 생성

Kubernetes Service Account에 IAM 역할을 연결합니다.

```bash
# AWS 계정 ID 가져오기
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Policy ARN 생성
POLICY_ARN="arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy"

# IRSA 생성
eksctl create iamserviceaccount \
  --cluster=petclinic-kr-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=$POLICY_ARN \
  --override-existing-serviceaccounts \
  --region=ap-northeast-2 \
  --approve
```

### 3. Helm으로 Controller 설치

```bash
# Helm 저장소 추가
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# AWS Load Balancer Controller 설치
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=petclinic-kr-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-northeast-2 \
  --set vpcId=vpc-05dbb2a501951af95
```

### 4. 설치 확인

```bash
# Deployment 확인
kubectl get deployment -n kube-system aws-load-balancer-controller

# Pod 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# 로그 확인
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

## 🏷️ Subnet 태그 설정

ALB가 올바른 서브넷에 배포되도록 태그를 설정합니다.

### Public Subnet에 태그 추가 (External ALB용)

```bash
# Public Subnet에 태그 추가
aws ec2 create-tags \
  --resources subnet-0b59cec4b30885866 subnet-07c475fa1608156d4 \
  --tags \
    Key=kubernetes.io/role/elb,Value=1 \
    Key=kubernetes.io/cluster/petclinic-kr-eks,Value=shared
```

### 태그 확인

```bash
# Subnet 태그 확인
aws ec2 describe-subnets \
  --subnet-ids subnet-0b59cec4b30885866 subnet-07c475fa1608156d4 \
  --query 'Subnets[*].{SubnetId:SubnetId,Tags:Tags[?contains(Key,`kubernetes`)]}' \
  --output table
```

**필수 태그**:
- `kubernetes.io/role/elb=1`: External ALB용 (Public Subnet)
- `kubernetes.io/role/internal-elb=1`: Internal ALB용 (Private Subnet)
- `kubernetes.io/cluster/<cluster-name>=shared`: EKS 클러스터 소유 표시

---

## 📦 Service 및 Ingress 생성

### 1. ClusterIP Service 생성

**test-service.yaml**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: test
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 2. Ingress 생성 (ALB)

**test-ingress.yaml**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: default
  annotations:
    # ALB 설정
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    
    # Health Check 설정
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/success-codes: '200'
    
    # 태그 설정
    alb.ingress.kubernetes.io/tags: Environment=production,Project=petclinic
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: test-service
                port:
                  number: 80
```

### 3. 리소스 배포

```bash
# Service 배포
kubectl apply -f test-service.yaml

# Ingress 배포
kubectl apply -f test-ingress.yaml
```

---

## ✅ 테스트 및 검증

### 1. 리소스 상태 확인

```bash
# Ingress 상태 확인
kubectl get ingress test-ingress

# ALB DNS 이름 가져오기
kubectl get ingress test-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Pod 확인
kubectl get pods -o wide

# Service 확인
kubectl get svc test-service

# Ingress 상세 정보
kubectl describe ingress test-ingress
```

### 2. ALB Controller 로그 확인

```bash
# Controller 로그
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100

# 실시간 로그 모니터링
kubectl logs -n kube-system deployment/aws-load-balancer-controller -f
```

### 3. AWS Console에서 ALB 확인

```bash
# ALB 목록 조회
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-default-testingr`)]'

# Target Group 확인
aws elbv2 describe-target-groups \
  --query 'TargetGroups[?contains(LoadBalancerArns[0], `k8s-default`)]'

# Target Health 확인
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

### 4. 엔드포인트 테스트

```bash
# ALB DNS 이름 가져오기
ALB_DNS=$(kubectl get ingress test-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# HTTP 요청 테스트
curl http://$ALB_DNS

# 응답 헤더 확인
curl -I http://$ALB_DNS
```

---

## 🔍 주요 Annotation 설명

### ALB 기본 설정

| Annotation | 설명 | 값 예시 |
|-----------|------|---------|
| `alb.ingress.kubernetes.io/scheme` | ALB 타입 | `internet-facing` / `internal` |
| `alb.ingress.kubernetes.io/target-type` | 타겟 타입 | `ip` / `instance` |
| `alb.ingress.kubernetes.io/listen-ports` | 리스너 포트 | `[{"HTTP": 80}, {"HTTPS": 443}]` |

### Health Check 설정

| Annotation | 설명 | 기본값 |
|-----------|------|--------|
| `alb.ingress.kubernetes.io/healthcheck-path` | Health Check 경로 | `/` |
| `alb.ingress.kubernetes.io/healthcheck-interval-seconds` | 체크 간격 | `15` |
| `alb.ingress.kubernetes.io/healthcheck-timeout-seconds` | 타임아웃 | `5` |
| `alb.ingress.kubernetes.io/success-codes` | 성공 HTTP 코드 | `200` |

### HTTPS/SSL 설정

```yaml
annotations:
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account-id:certificate/xxx
  alb.ingress.kubernetes.io/ssl-redirect: '443'
```

---

## 🔧 트러블슈팅

### 문제 1: ALB가 생성되지 않음

**확인사항**:
```bash
# Controller 로그 확인
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Subnet 태그 확인
aws ec2 describe-subnets \
  --subnet-ids <subnet-id> \
  --query 'Subnets[0].Tags'

# IAM 권한 확인
aws iam get-policy-version \
  --policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --version-id v1
```

### 문제 2: Target이 Unhealthy 상태

**해결 방법**:
```bash
# Pod 상태 확인
kubectl get pods -l app=test

# Pod 로그 확인
kubectl logs <pod-name>

# Service Endpoint 확인
kubectl get endpoints test-service

# Health Check 경로 테스트 (Pod 내부에서)
kubectl exec <pod-name> -- curl localhost:80/
```

### 문제 3: 503 Service Unavailable

**원인 및 해결**:
- Pod가 준비되지 않음 → Readiness Probe 확인
- Security Group 문제 → ALB → Pod 통신 허용 확인
- Target Type 불일치 → `ip` vs `instance` 확인

```bash
# Security Group 확인
aws ec2 describe-security-groups \
  --filters "Name=tag:kubernetes.io/cluster/petclinic-kr-eks,Values=owned"
```

---

## 📚 참고 자료

- [AWS Load Balancer Controller 공식 문서](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Ingress Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)

---

## 🎯 다음 단계

ALB 통합이 완료되었습니다. 다음 단계는:

1. [03. EKS - EFS Storage 연동](./03-EKS-EFS-Storage.md) - 영구 스토리지 구성
2. [04. ECR - EKS - RDS 통합](./04-ECR-EKS-RDS-Integration.md) - 전체 애플리케이션 스택 구축