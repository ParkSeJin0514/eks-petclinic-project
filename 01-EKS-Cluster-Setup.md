# 🚀 EKS Cluster Setup

## 1. 개요

AWS EKS 기반으로 Kubernetes 클러스터 구성  
Control Plane은 AWS가 관리하며 고가용성 구성 자동 지원  
테스트 환경에서는 All Traffic Allow 기반으로 구성  
실제 서비스 환경에서는 최소 권한 기반으로 구성 필요

---

## 2. 기본 구성

### ▶️ 베이스 인프라

- Bastion Server 구성  
- EKS Management Instance 구성 (kubectl, eksctl 실행용)

### ▶️ 아키텍처 구조

```
Internet
  ↓
Public Subnet (10.0.10.0/24, 10.0.20.0/24)
  └─ Bastion Server
        ↓ SSH
Private Subnet (10.0.50.0/24)
  └─ K8s Management Instance
        ↓ HTTPS 443
EKS Control Plane (AWS Managed)
        ↓
Private Subnet (10.0.100.0/24, 10.0.110.0/24)
  └─ Worker Nodes (ASG)
```

---

## 3. IAM 역할 구성

### ▶️ 목적  
Management Instance가 EC2, VPC, EKS 등을 제어하기 위한 권한 구성

### ▶️ Inline Policy 예시

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
    },
    { "Sid": "CloudFormationFullAccess", "Effect": "Allow", "Action": "cloudformation:*", "Resource": "*" },
    { "Sid": "AutoScalingFullAccess", "Effect": "Allow", "Action": "autoscaling:*", "Resource": "*" }
  ]
}
```

---

## 4. 관리 도구 설치

### ▶️ kubectl 설치

```bash
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### ▶️ unzip 설치

```bash
sudo apt update && sudo apt install unzip -y
```

### ▶️ AWS CLI v2 설치

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
rm -rf aws awscliv2.zip
```

### ▶️ eksctl 설치

```bash
ARCH=amd64
PLATFORM="$(uname -s)_$ARCH"
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf eksctl_${PLATFORM}.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
chmod +x /usr/local/bin/eksctl
```

### ▶️ Helm 설치

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## 5. EKS 클러스터 생성

### ▶️ 서브넷 조회

```bash
aws ec2 describe-subnets   --filters "Name=vpc-id,Values=vpc-05dbb2a501951af95"   --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]'   --region ap-northeast-2 --output table
```

### ▶️ cluster.yaml 생성

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
    instanceTypes: [t3.large]
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

### ▶️ 생성 명령

```bash
eksctl create cluster --config-file=cluster.yaml --verbose 4
```

---

## 6. 클러스터 확인

### ▶️ 클러스터 정보 확인

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A
```

### ▶️ 노드 상세 확인

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,INTERNAL_IP:.status.addresses[0].address,INSTANCE_ID:.spec.providerID
```

---

## 7. CloudFormation 확인

### ▶️ 실패 이벤트 조회

```bash
aws cloudformation describe-stack-events   --stack-name eksctl-petclinic-kr-k8s-nodegroup-ng-app   --query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,ResourceStatusReason]'   --region ap-northeast-2 --output table
```

### ▶️ 클러스터 상태 조회

```bash
aws eks describe-cluster   --name petclinic-kr-k8s   --region ap-northeast-2   --query 'cluster.{Status:status,Endpoint:endpoint,Version:version}'
```

---

## 8. 클러스터 삭제 및 재생성

### ▶️ 삭제

```bash
eksctl delete cluster --name petclinic-kr-eks --region ap-northeast-2 --wait
```

### ▶️ CloudFormation 스택 삭제 확인

```bash
aws cloudformation list-stacks   --region ap-northeast-2   --query 'StackSummaries[?contains(StackName,`eksctl-petclinic-kr-k8s`)].[StackName,StackStatus]'   --output table
```

### ▶️ 재생성

```bash
eksctl create cluster --config-file=cluster.yaml --verbose 4
```

---

## 9. kubeconfig 설정

### ▶️ kubeconfig 생성

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-kr-eks
```

### ▶️ Context 확인

```bash
kubectl config get-contexts
kubectl config current-context
```

### ▶️ 자동완성 설정

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
```

---

## 10. 테스트 배포

### ▶️ nginx 배포

```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl get deployment
kubectl get pods -o wide
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl get svc nginx
```

curl 테스트는 클러스터 내부 또는 Management Instance에서 수행 가능

nginx 테스트 배포 완료