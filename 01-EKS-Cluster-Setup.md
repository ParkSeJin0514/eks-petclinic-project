EKS Cluster
직접 VM에서 하는게 아니라서 되게 쉽게 구성
알아서 HA까지 해주고 CP의 경우 AWS에서 자동으로 관리해주기 때문에 아주 편함
기본 구성
일단 기본적으로 Bastion Server와 EKS mgmt가 될 인스턴스 생성
일단 테스트 해야해서 모든 Node와 Instance는 All Traffic Allow
기본 아키텍처
인터넷
↓
Public Subnet 10.0.10.0/24, 10.0.20.0/24
└─ Bastion Server
↓ SSH
Private Subnet 10.0.50.0/24
└─ K8s Management Instance (kubectl 실행)
↓ HTTPS443
EKS Control Plane AWS Managed)
↓
Private Subnet 10.0.100.0/24, 10.0.110.0/24
└─ Worker Nodes Auto Scaling Group)
역할 설정
IAM
IAM 역할은 테스트 용도면 AdminFull을 줘도 가능
하지만 실제 서비스면 절대 그렇게 주면 안됨
Management Instance에 IAM 역할 생성 후 부여
# IAM  Policies  Create policy  Json
# Inline policy
{
"Version": "20121017",
"Statement": [
{
"Sid": "EC2FullAccess",
EKS Cluster 1

"Effect": "Allow",
"Action": "ec2*",
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
"ec2*Vpc*",
"ec2*Subnet*",
"ec2*Gateway*",
"ec2*Vpn*",
"ec2*Route*",
"ec2*Address*",
"ec2*SecurityGroup*",
"ec2*NetworkAcl*",
"ec2*NetworkInterface*",
"ec2*CustomerGateway*",
"ec2*VpnConnection*",
"ec2*VpnGateway*",
"ec2*TransitGateway*",
"ec2Describe*"
],
"Resource": "*"
},
{
"Sid": "CloudFormationFullAccess",
"Effect": "Allow",
"Action": "cloudformation:*",
EKS Cluster 2

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
"elasticloadbalancingv2*"
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
"s3*",
"sts:*"
],
"Resource": "*"
}
]
}
구성
kubectl
일단 기본적으로 버전은 1.33을 기준으로 작성
kubectl 설치
# kubectl 1.33 다운로드
curl LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl"
EKS Cluster 3

# 체크섬 검증 (선택사항)
curl LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256 kubectl" | sha256sum --check
# 실행 권한 부여 및 설치
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
# 설치 확인
kubectl version --client
# 출력 예시: Client Version: v1.33.0
unzip 설치
# 1. unzip 설치
sudo apt update
sudo apt install unzip -y
AWS CLI v2 설치
kubectl이 EKS 클러스터와 통신하려면 인증 토큰이 필요!
이 토큰을 생성하는 것이 AWS CLI
최신 버전의 AWS CLI 설치 또는 업데이트 - AWS Command Line Interface
AWS CLI는 AWS 서비스와 상호 작용하는 명령을 제공하는 AWS SDK for
Python(Boto)를 사용하여 구축된 오픈 소스 도구입니다. 최소한의 구성으로, 원하는 터미
널 프로그램에서 AWS Management Console이 제공한 모든 기능을 사용할 수 있습니
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-in
stall.html
AWS CLI에서 명령 완성 구성 - AWS Command Line Interface
AWS CLI는 AWS 서비스와 상호 작용하는 명령을 제공하는 AWS SDK for
Python(Boto)를 사용하여 구축된 오픈 소스 도구입니다. 최소한의 구성으로,
원하는 터미널 프로그램에서 AWS Management Console이 제공한 모든
https://docs.aws.amazon.com/ko_kr/cli/v1/userguide/cli-config
ure-completion.html
# AWS CLI v2 다운로드 및 설치
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zi
p"
unzip awscliv2.zip
sudo ./aws/install
EKS Cluster 4

# 설치 확인
aws --version
# 출력 예시: aws-cli/2.x.x Python/3.x.x Linux/x.x.x
# 정리
rm -rf aws awscliv2.zip
# kubeconfig 생성 명령어 - AWS CLI 필수!
aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-kr-eks
eksctl 설치
# 아키텍처 확인
uname -m
# 출력: x86_64
# eksctl 최신 버전 다운로드
ARCH=amd64
PLATFORM$(uname -s)_$ARCH
curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl
_$PLATFORM.tar.gz"
# 압축 해제
tar -xzf eksctl_$PLATFORM.tar.gz C /tmp
# 시스템 경로로 이동
sudo mv /tmp/eksctl /usr/local/bin
# 권한 설정
sudo chmod +x /usr/local/bin/eksctl
# 설치 확인
eksctl version
# 출력: 0.xxx.x
# 정리
rm eksctl_$PLATFORM.tar.gz
Helm 설치 (패키지 관리자)
EKS Cluster 5

# Helm 설치 스크립트 실행
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# 설치 확인
helm version
# 출력 예시: version.BuildInfo{Version:"v3.x.x"...}
# Helm 레포지토리 추가 (선택사항)
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
배포
EKS
Management Instance에서 eksctl을 사용하여 cluster.yaml 파일을 실행
VPC의 모든 서브넷 조회해서 정확한 서브넷 확인
# VPC의 모든 서브넷 조회
aws ec2 describe-subnets \
--filters "Name=vpc-id,Values=vpc-05dbb2a501951af95" \
--region ap-northeast-2 \
--query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' \
--output table
**출력 예시**
------------------------------------------------------------------------------------
| DescribeSubnets |
+---------------------------+----------------+------------------+-----------------
--+
| subnet-004fea8a0253ba05c | 10.0.50.0/24 | ap-northeast-2a | private3-eks-
mgmt |
| subnet-0f4ddf6ff63a9fe21 | 10.0.60.0/24 | ap-northeast-2c | private2-eks-mg
mt |
| subnet-XXXXXXXXXXXX | 10.0.100.0/24 | ap-northeast-2a | private-worker
|
| subnet-YYYYYYYYYYYY | 10.0.110.0/24 | ap-northeast-2c | private-worker
|
EKS Cluster 6

+---------------------------+----------------+------------------+-----------------
--+
cluster.yaml 생성
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
name: petclinic-kr-eks
region: ap-northeast-2
version: "1.33"
vpc:
id: vpc-05dbb2a501951af95
subnets:
private: # EKS Cluster 범위 지정
ap-northeast-2a: # 기존에는 Instance Management 서브넷만 지정
id: subnet-0823227d57e749380
ap-northeast-2c:
id: subnet-0f4ddfc7f69a9f621
iam:
withOIDC true
managedNodeGroups: # EKS WorerNode 범위 지정
- name: ng-app
instanceTypes:
- t3.large
minSize: 3
desiredCapacity: 3
maxSize: 6
privateNetworking: true
ssh:
allow: true # SSH 접속 허용 (디버깅용)
publicKeyName: project # 실제 키 이름
subnets:
- subnet-0944eac02533aa03c
- subnet-0435656f96272cc56
volumeSize: 20
volumeType: gp3
volumeEncrypted: true
iam:
withAddonPolicies:
autoScaler: true
EKS Cluster 7

ebs: true
efs: true
albIngress: true
cloudWatch: true
externalDNS true
tags:
Environment: production
ManagedBy: eksctl
Project: petclinic
labels:
role: application
environment: production
# 파일 내용 확인
cat cluster.yaml
# 파일 크기 확인
ls -lh cluster.yaml
# YAML 문법 검증 (Python의 yaml 모듈 사용)
python3 -c "import yaml; yaml.safe_load(open('cluster.yaml'))" && echo "✅ YAM
L 문법 정상"
# eksctl로 설정 검증 (실제 생성 안 함)
eksctl create cluster --config-file=cluster.yaml --dry-run
# 클러스터 생성 시작
eksctl create cluster --config-file=cluster.yaml --verbose 4
생성 확인
# kubeconfig는 자동 생성됨
# ~/.kube/config 에 추가됨
# 클러스터 정보 확인
kubectl cluster-info
# 노드 확인 (중요!)
kubectl get nodes -o wide
# 노드가 올바른 서브넷에 있는지 확인
EKS Cluster 8

kubectl get nodes -o custom-columns=\
NAME.metadata.name,\
INTERNALIP.status.addresses[0].address,\
INSTANCEID.spec.providerID
# 네임스페이스 확인
kubectl get namespaces
# 모든 Pod 확인
kubectl get pods A
CloudFormation Error 확인
# Node Group 스택의 실패 이유 확인
aws cloudformation describe-stack-events \
--stack-name eksctl-petclinic-kr-k8s-nodegroup-ng-app \
--region ap-northeast-2 \
--query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].LogicalResourceId,
ResourceStatusReason]' \
--output table
# 최근 이벤트 50개 확인
aws cloudformation describe-stack-events \
--stack-name eksctl-petclinic-kr-k8s-nodegroup-ng-app \
--region ap-northeast-2 \
--max-items 50 \
--query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalReso
urceId,ResourceStatusReason]' \
--output table
aws eks describe-cluster \
--name petclinic-kr-k8s \
--region ap-northeast-2 \
--query 'cluster.Status:status,Endpoint:endpoint,Version:version}'
클러스터 완전 삭제 후 재생성
# 1. 완전히 삭제
eksctl delete cluster --name petclinic-kr-eks --region ap-northeast-2 --wait
# 2. CloudFormation 스택 완전 삭제 확인
EKS Cluster 9

aws cloudformation list-stacks \
--region ap-northeast-2 \
--query 'StackSummaries[?contains(StackName,`eksctl-petclinic-kr-k8s`)].Stac
kName,StackStatus]' \
--output table
# 모두 DELETE_COMPLETE 또는 목록에 없어야 함
# 3. 재생성
cd ~/eks-config
eksctl create cluster --config-file=cluster.yaml --verbose 4
상태 확인
CloudFormation
CloudFormation이 제대로 작동하고 있는지 여러가지 방법으로 확인
Stack Status Check
watch -n 10 'aws cloudformation describe-stacks \
--region ap-northeast-2 \
--query "Stacks[?contains(StackName,\"eksctl-petclinic\")].Name:StackName,S
tatus:StackStatus}" \
--output table'
EKS Cluster Status Check
watch -n 10 'aws eks describe-cluster \
--name petclinic-kr-k8s \
--region ap-northeast-2 \
--query "cluster.status" \
--output text 2/dev/null || echo "아직 생성 안됨"'
CloudFormation 이벤트 실시간 확인
# 최신 이벤트 10개 계속 갱신
watch -n 5 'aws cloudformation describe-stack-events \
--stack-name eksctl-petclinic-kr-eks-cluster \
--region ap-northeast-2 \
--max-items 10 \
--query "StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalReso
EKS Cluster 10

urceId]" \
--output table 2/dev/null'
기본 설정
EKS
kubeconfig 설정으로 연결
kubeconfig 설정
# AWS 자격 증명 확인
aws sts get-caller-identity
# EKS 클러스터 목록 확인
aws eks list-clusters --region ap-northeast-2
# kubeconfig 생성 (클러스터명 예: my-eks-cluster)
aws eks update-kubeconfig \
--region ap-northeast-2 \
--name petclinic-kr-eks
# 출력 예시
# Added new context arn:aws:eks:ap-northeast-2:xxxx:cluster/my-eks-cluster to
/home/ec2-user/.kube/config
연결 확인
# 클러스터 정보 확인
kubectl cluster-info
# 출력 예시:
# Kubernetes control plane is running at https://xxxxx.gr7.ap-northeast-2.eks.ama
zonaws.com
# CoreDNS is running at https://xxxxx.gr7.ap-northeast-2.eks.amazonaws.com/ap
i/v1/namespaces/kube-system/services/kube-dns:dns/proxy
# 노드 목록 확인
kubectl get nodes
# 출력 예시:
# NAME STATUS ROLES AGE VERSION
# ip-100100123.ap-northeast-2.compute.internal Ready <none> 1h v1.33.
EKS Cluster 11

0-eks-xxxxx
# ip-100110456.ap-northeast-2.compute.internal Ready <none> 1h v1.33.
0-eks-xxxxx
# 모든 네임스페이스의 Pod 확인
kubectl get pods A
# EKS 버전 확인
kubectl version --short
kubeconfig 파일 구조 확인
# kubeconfig 파일 위치
cat ~/.kube/config
# 컨텍스트 목록 확인
kubectl config get-contexts
# 현재 컨텍스트 확인
kubectl config current-context
kubectl 자동완성 설정 (선택 사항)
# bash 자동완성
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
Namespace 설정 (선택 사항)
# default 네임스페이스 대신 다른 네임스페이스를 기본으로 설정
kubectl config set-context --current --namespace=<네임스페이스명>
# 예시
kubectl config set-context --current --namespace=development
생성
Deploy
테스트용 Pod 생성
EKS Cluster 12

테스트 용이기 때문에 참고만!
# 3개 replica로 nginx 배포
kubectl create deployment nginx --image=nginx --replicas=3
# 확인
kubectl get deployment
kubectl get pods -o wide
# Service 노출
kubectl expose deployment nginx --port=80 --type=ClusterIP
# Service 확인
kubectl get svc nginx
# curl 확인
curl Pod IP
EKS Cluster 13