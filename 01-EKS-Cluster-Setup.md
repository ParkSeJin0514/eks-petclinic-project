# 🚀 EKS Cluster Setup - PetClinic Project

## 📋 구성 개요

PetClinic 애플리케이션을 위한 **AWS EKS 클러스터 구성 과정 정리**

- **목적**
  - 직접 VM 위에 Kubernetes를 설치하는 방식이 아닌,  
    **AWS 관리형 서비스인 EKS**를 활용한 안정적인 클러스터 구성
- **특징**
  - Control Plane은 AWS가 관리  
  - 기본적으로 고가용성(HA) 제공  
  - Node 그룹은 Auto Scaling Group 기반 확장/축소  
  - Management Instance 기반 운영

---

## 🎯 구성 목표

1. **안정적인 EKS 클러스터 구성**  
2. **운영 편의성 확보**  
3. **확장 가능한 구조 확보**  
4. **테스트 환경 기준 구성**

---

## 🏗️ 기본 아키텍처

```text
Internet
  ↓
Public Subnet (10.0.10.0/24, 10.0.20.0/24)
  └─ Bastion Server
        ↓ SSH
Private Subnet (10.0.50.0/24)
  └─ K8s Management Instance (kubectl, eksctl 실행)
        ↓ HTTPS 443
EKS Control Plane (AWS Managed)
        ↓
Private Subnet (10.0.100.0/24, 10.0.110.0/24)
  └─ Worker Nodes (ASG)
```

---

## 🧰 사전 준비 및 IAM 역할

IAM Role 구성 예시:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "EC2FullAccess", "Effect": "Allow", "Action": "ec2:*", "Resource": "*" },
    { "Sid": "EKSFullAccess", "Effect": "Allow", "Action": "eks:*", "Resource": "*" },
    { "Sid": "IAMFullAccess", "Effect": "Allow", "Action": "iam:*", "Resource": "*" },
    {
      "Sid": "VPCFullAccess",
      "Effect": "Allow",
      "Action": ["ec2:*Vpc*", "ec2:*Subnet*", "ec2:*Gateway*", "ec2:*SecurityGroup*", "ec2:Describe*"],
      "Resource": "*"
    }
  ]
}
```

---

## 🧩 관리 도구 설치

### kubectl 설치

```bash
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### AWS CLI 설치

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### eksctl 설치

```bash
ARCH=amd64
PLATFORM="$(uname -s)_$ARCH"
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf eksctl_${PLATFORM}.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### Helm 설치

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo update
```

---

## ⚙️ EKS cluster.yaml 구성

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: petclinic-kr-eks
  region: ap-northeast-2
  version: "1.33"

vpc:
  id: vpc-05dbb2a501951af95
  subnets:
    private:
      ap-northeast-2a: { id: subnet-0823227d57e749380 }
      ap-northeast-2c: { id: subnet-0f4ddfc7f69a9f621 }

iam:
  withOIDC: true

managedNodeGroups:
  - name: ng-app
    instanceTypes:
      - t3.large
    minSize: 3
    desiredCapacity: 3
    maxSize: 6
    privateNetworking: true
    ssh:
      allow: true
      publicKeyName: project
    volumeSize: 20
    volumeType: gp3
    volumeEncrypted: true
```

---

## 🚀 EKS 클러스터 생성

```bash
eksctl create cluster --config-file=cluster.yaml --verbose 4
```

---

## 🔍 EKS 상태 확인

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A
```

노드 서브넷 매핑 확인

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,INTERNAL_IP:.status.addresses[0].address,INSTANCE_ID:.spec.providerID
```

---

## 🧱 CloudFormation 상태 점검

```bash
aws cloudformation describe-stack-events   --stack-name eksctl-petclinic-kr-k8s-nodegroup-ng-app   --region ap-northeast-2   --output table
```

---

## ♻️ 클러스터 삭제 및 재생성

### 삭제

```bash
eksctl delete cluster --name petclinic-kr-eks --region ap-northeast-2 --wait
```

### 재생성

```bash
eksctl create cluster --config-file=cluster.yaml --verbose 4
```

---

## 🔑 kubeconfig 설정

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-kr-eks
kubectl config get-contexts
kubectl config current-context
```

---

## 🧪 테스트 배포 (nginx)

```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl get svc nginx
```

nginx 응답 테스트 후 클러스터 기본 동작 검증 완료