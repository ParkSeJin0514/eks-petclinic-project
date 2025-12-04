# 01. EKS Cluster 구성

## 📋 개요

AWS EKS(Elastic Kubernetes Service)를 사용하여 관리형 Kubernetes 클러스터를 구축합니다. EKS는 Control Plane을 AWS에서 자동으로 관리해주기 때문에 HA(High Availability) 구성이 쉽고, 직접 VM에서 Kubernetes를 구축하는 것보다 훨씬 편리합니다.

## 🏗️ 기본 아키텍처

```
Internet
   ↓
Public Subnet (10.0.10.0/24, 10.0.20.0/24)
└─ Bastion Server
   ↓ SSH
Private Subnet (10.0.50.0/24)
└─ K8s Management Instance (kubectl 실행)
   ↓ HTTPS:443
EKS Control Plane (AWS Managed)
   ↓
Private Subnet (10.0.100.0/24, 10.0.110.0/24)
└─ Worker Nodes (Auto Scaling Group)
```

### 구성 요소

- **Bastion Server**: Public Subnet에 배치, SSH 접근용
- **Management Instance**: Private Subnet에 배치, kubectl/eksctl 명령 실행
- **EKS Control Plane**: AWS 관리형, HA 자동 구성
- **Worker Nodes**: Private Subnet에 배치, Auto Scaling Group으로 관리

> **⚠️ 주의**: 테스트 환경이므로 모든 Node와 Instance는 All Traffic Allow로 설정. 프로덕션 환경에서는 반드시 최소 권한 원칙을 적용해야 합니다.

---

## 🔐 IAM 역할 설정

### Management Instance IAM 정책

테스트 환경에서는 AdministratorAccess를 사용할 수 있지만, 프로덕션에서는 반드시 최소 권한으로 제한해야 합니다.

**IAM → Policies → Create policy → JSON**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2FullAccess",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    },
    {
      "Sid": "EKSFullAccess",
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
    },
    {
      "Sid": "IAMFullAccess",
      "Effect": "Allow",
      "Action": "iam:*",
      "Resource": "*"
    },
    {
      "Sid": "VPCFullAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:*Vpc*",
        "ec2:*Subnet*",
        "ec2:*Gateway*",
        "ec2:*Vpn*",
        "ec2:*Route*",
        "ec2:*Address*",
        "ec2:*SecurityGroup*",
        "ec2:*NetworkAcl*",
        "ec2:*NetworkInterface*",
        "ec2:*CustomerGateway*",
        "ec2:*VpnConnection*",
        "ec2:*VpnGateway*",
        "ec2:*TransitGateway*",
        "ec2:Describe*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudFormationFullAccess",
      "Effect": "Allow",
      "Action": "cloudformation:*",
      "Resource": "*"
    },
    {
      "Sid": "AutoScalingFullAccess",
      "Effect": "Allow",
      "Action": "autoscaling:*",
      "Resource": "*"
    },
    {
      "Sid": "ElasticLoadBalancingFullAccess",
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:*",
        "elasticloadbalancingv2:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AdditionalServicesAccess",
      "Effect": "Allow",
      "Action": [
        "ssm:*",
        "kms:*",
        "logs:*",
        "s3:*",
        "sts:*"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 🛠️ Management Instance 구성

### 1. kubectl 설치 (v1.33 기준)

```bash
# kubectl 1.33 다운로드
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl"

# 체크섬 검증 (선택사항)
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# 실행 권한 부여 및 설치
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 설치 확인
kubectl version --client
# 출력 예시: Client Version: v1.33.0
```

### 2. unzip 설치

```bash
sudo apt update
sudo apt install unzip -y
```

### 3. AWS CLI v2 설치

kubectl이 EKS 클러스터와 통신하려면 인증 토큰이 필요합니다. 이 토큰을 생성하는 것이 AWS CLI의 역할입니다.

```bash
# AWS CLI v2 다운로드 및 설치
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 설치 확인
aws --version
# 출력 예시: aws-cli/2.x.x Python/3.x.x Linux/x.x.x

# 정리
rm -rf aws awscliv2.zip
```

**kubeconfig 생성 명령어 (나중에 사용)**:
```bash
aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-kr-eks
```

### 4. eksctl 설치

```bash
# 아키텍처 확인
uname -m
# 출력: x86_64

# eksctl 최신 버전 다운로드
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# 압축 해제
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp

# 시스템 경로로 이동
sudo mv /tmp/eksctl /usr/local/bin

# 권한 설정
sudo chmod +x /usr/local/bin/eksctl

# 설치 확인
eksctl version
# 출력: 0.xxx.x

# 정리
rm eksctl_$PLATFORM.tar.gz
```

### 5. Helm 설치 (Kubernetes 패키지 관리자)

```bash
# Helm 설치 스크립트 실행
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 설치 확인
helm version
# 출력 예시: version.BuildInfo{Version:"v3.x.x"...}

# Helm 레포지토리 추가 (선택사항)
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## 🚀 EKS 클러스터 배포

### 1. VPC 서브넷 확인

eksctl로 클러스터를 생성하기 전에 기존 VPC의 서브넷 정보를 확인합니다.

```bash
# VPC의 모든 서브넷 조회
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-xxxxxxxxx" \
  --region ap-northeast-2 \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

**출력 예시**:
```
-------------------------------------------------------------
|                     DescribeSubnets                       |
+-------------------------+----------------+-----------------+
|  subnet-xxxxx1         |  10.0.10.0/24  |  ap-northeast-2a|
|  subnet-xxxxx2         |  10.0.20.0/24  |  ap-northeast-2c|
|  subnet-xxxxx3         |  10.0.50.0/24  |  ap-northeast-2a|
|  subnet-xxxxx4         |  10.0.100.0/24 |  ap-northeast-2a|
|  subnet-xxxxx5         |  10.0.110.0/24 |  ap-northeast-2c|
+-------------------------+----------------+-----------------+
```

### 2. cluster.yaml 파일 작성

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: petclinic-kr-eks
  region: ap-northeast-2
  version: "1.31"

vpc:
  id: "vpc-05dbb2a501951af95"
  subnets:
    private:
      ap-northeast-2a:
        id: subnet-xxxxx4  # 10.0.100.0/24
      ap-northeast-2c:
        id: subnet-xxxxx5  # 10.0.110.0/24

managedNodeGroups:
  - name: petclinic-kr-ng
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 4
    volumeSize: 20
    privateNetworking: true
    subnets:
      - subnet-xxxxx4  # ap-northeast-2a
      - subnet-xxxxx5  # ap-northeast-2c
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        externalDNS: true
        certManager: true
        appMesh: true
        ebs: true
        fsx: true
        efs: true
        albIngress: true
        xRay: true
        cloudWatch: true

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
```

### 3. EKS 클러스터 생성

```bash
# cluster.yaml 파일로 클러스터 생성
eksctl create cluster -f cluster.yaml

# 생성 확인 (약 15-20분 소요)
kubectl get nodes
```

**출력 예시**:
```
NAME                                               STATUS   ROLES    AGE   VERSION
ip-10-0-100-xxx.ap-northeast-2.compute.internal   Ready    <none>   5m    v1.31.x
ip-10-0-110-xxx.ap-northeast-2.compute.internal   Ready    <none>   5m    v1.31.x
```

### 4. kubectl 설정 확인

```bash
# 현재 컨텍스트 확인
kubectl config current-context

# 클러스터 정보 확인
kubectl cluster-info

# 네임스페이스 확인
kubectl get namespaces
```

---

## ✅ 검증 및 테스트

### 클러스터 상태 확인

```bash
# 노드 상세 정보
kubectl get nodes -o wide

# 시스템 Pod 확인
kubectl get pods -n kube-system

# EKS 클러스터 정보 (AWS CLI)
aws eks describe-cluster --name petclinic-kr-eks --region ap-northeast-2
```

### 간단한 테스트 Pod 배포

```bash
# nginx 테스트 배포
kubectl create deployment nginx-test --image=nginx

# Pod 확인
kubectl get pods

# 배포 확인
kubectl get deployments

# 정리
kubectl delete deployment nginx-test
```

---

## 🔧 트러블슈팅

### 문제 1: kubectl 명령어 실행 시 인증 오류

```bash
# kubeconfig 재생성
aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-kr-eks

# aws-iam-authenticator 설치 (필요시)
curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator
chmod +x ./aws-iam-authenticator
sudo mv ./aws-iam-authenticator /usr/local/bin
```

### 문제 2: 노드가 NotReady 상태

```bash
# 노드 상세 로그 확인
kubectl describe node <node-name>

# 시스템 Pod 상태 확인
kubectl get pods -n kube-system

# AWS 콘솔에서 Auto Scaling Group 및 인스턴스 상태 확인
```

### 문제 3: eksctl 생성 중 오류

```bash
# CloudFormation 스택 확인
aws cloudformation describe-stacks --region ap-northeast-2

# 실패한 스택 삭제
eksctl delete cluster --name petclinic-kr-eks --region ap-northeast-2
```

---

## 📚 참고 자료

- [Amazon EKS 공식 문서](https://docs.aws.amazon.com/eks/)
- [eksctl 공식 문서](https://eksctl.io/)
- [kubectl 치트 시트](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [AWS CLI 설치 가이드](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

---

## 🎯 다음 단계

EKS 클러스터 구성이 완료되었습니다. 다음 단계는:

1. [02. EKS - ALB 통합](./02-EKS-ALB-Integration.md) - Application Load Balancer 설정
2. [03. EKS - EFS Storage 연동](./03-EKS-EFS-Storage.md) - 영구 스토리지 구성