EFS → EKS
First Project 할 때처럼 정적 웹사이트를 띄우려고 했음
어찌저찌 성공은 했기 때문에 그걸 바탕으로 작성
생성
EFS
파일 생성할 때 SG 설정 잘 봐야함
어느 서브넷에 마운트 할 건지도 확인 필수!
파일 생성
aws efs create-file-system \
--performance-mode generalPurpose \
--throughput-mode bursting \
--encrypted \
--tags Key=Name,Value=petclinic-kr-efs \
--region ap-northeast-2
서브넷 확인
# 10.0.100.0/24 서브넷 ID 찾기
aws ec2 describe-subnets \
--filters "Name=cidr-block,Values=10.0.100.0/24" \
--region ap-northeast-2 \
--query 'Subnets[0].[SubnetId,AvailabilityZone]' \
--output table
# 10.0.110.0/24 서브넷 ID 찾기
aws ec2 describe-subnets \
--filters "Name=cidr-block,Values=10.0.110.0/24" \
--region ap-northeast-2 \
--query 'Subnets[0].[SubnetId,AvailabilityZone]' \
--output table
Worker Node SG 찾기
EFS  EKS 1

# EKS 클러스터의 Security Group 확인
aws eks describe-cluster \
--name petclinic-kr-eks \
--region ap-northeast-2 \
--query 'cluster.resourcesVpcConfig.ClusterSG:clusterSecurityGroupId,S
ecurityGroupIds:securityGroupIds}' \
--output json
SG에 NFS 포트 허용 추가
# Cluster Security Group에 NFS 포트 허용 (자기 자신으로부터)
aws ec2 authorize-security-group-ingress \
--group-id <nfs sg> \
--protocol tcp \
--port 2049 \
--source-group <nfs sg> \
--region ap-northeast-2
# Additional Security Group에서도 접근 허용
aws ec2 authorize-security-group-ingress \
--group-id <nfs sg> \
--protocol tcp \
--port 2049 \
--source-group <worker node sg> \
--region ap-northeast-2
EFS Mount Target 생성
# Subnet 1 10.0.100.0/24, ap-northeast-2a)에 마운트 타겟 생성
aws efs create-mount-target \
--file-system-id <nfs id> \
--subnet-id subnet-0944eac02533aa03c \
--security-groups <nfs sg> \
--region ap-northeast-2
# Subnet 2 10.0.110.0/24, ap-northeast-2c)에 마운트 타겟 생성
aws efs create-mount-target \
EFS  EKS 2

--file-system-id <nfs id> \
--subnet-id subnet-0435656f96272cc56 \
--security-groups <nfs sg> \
--region ap-northeast-2
Mount 확인
# 마운트 타겟 상태 확인
aws efs describe-mount-targets \
--file-system-id fs-0e92527d8c619a8d7 \
--region ap-northeast-2 \
--query 'MountTargets[].[MountTargetId,LifeCycleState,AvailabilityZoneN
ame]' \
--output table
EFS CSI Driver
K8s는 스토리지를 직접 인식하지 못함
EFS는 AWS 서비스라서 K8s가 EFS를 사용하려면 통신을 할 수 있는 드라이버 필요
그 역할을 EFS CSI Driver가 함
OIDC Provider 연결
eksctl utils associate-iam-oidc-provider \
--cluster petclinic-kr-eks \
--region ap-northeast-2 \
--approve
EFS CSI Driver IAM 정책 생성
# IAM 정책 JSON 다운로드
curl -o iam-policy-efs.json https://raw.githubusercontent.com/kubernetes-
sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json
# IAM 정책 생성
aws iam create-policy \
--policy-name AmazonEKS_EFS_CSI_Driver_Policy \
EFS  EKS 3

--policy-document file://iam-policy-efs.json \
--region ap-northeast-2
EFS CSI Driver Service Account 생성
# ServiceAccount 생성 (IAM 역할 자동 연결)
eksctl create iamserviceaccount \
--cluster petclinic-kr-eks \
--namespace kube-system \
--name efs-csi-controller-sa \
--attach-policy-arn arn:aws:iam::946775837287:policy/AmazonEKS_EFS_
CSI_Driver_Policy \
--approve \
--region ap-northeast-2
EFS CSI Driver 설치
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/k
ubernetes/overlays/stable/?ref=release-2.0"
kubectl get pods -n kube-system | grep efs
StorageClass
EFS를 쓰려면 직접 PV를 생성할 수 있지만 그렇게 하면 정적 프로비저닝으로 고정된 EFS
만 사용
동적 프로비저닝을 위해 StorageClass를 사용
PVC가 생성될 때 자동으로 EFS에 PV를 프로비저닝
storageclass.yaml 파일 생성
cat > efs-storageclass.yaml EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
name: efs-sc
provisioner: efs.csi.aws.com
EFS  EKS 4

parameters:
provisioningMode: efs-ap
fileSystemId: <nfs id>
directoryPerms: "777"
EOF
kubectl apply -f efs-storageclass.yaml
PersistentVolumeClaim PVC 생성
cat > efs-pvc.yaml EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: efs-static-web
namespace: default
spec:
accessModes:
 ReadWriteMany
storageClassName: efs-sc
resources:
requests:
storage: 5Gi
EOF
kubectl apply -f efs-pvc.yaml
kubectl get pvc efs-static-web
권한 설정
EFS
권한 설정을 안하고 하니깐 Permission Denied 발생
읽을 수 있게 권한 부여 필요!
EFS 마운트 확인
# Management Instance에서 EFS 구조 확인
ls -la /mnt/efs/
EFS  EKS 5

# PVC 디렉토리 내부 확인
ls -la /mnt/efs/<pvc volume>/
EFS에 index.html 업로드
# 1. EFS 유틸리티 설치
sudo apt-get update
sudo apt-get install -y nfs-common
# 2. 마운트 포인트 생성
sudo mkdir -p /mnt/efs
# 3. EFS 마운트 (NFS v4.1 사용)
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,tim
eo=600,retrans=2,noresvport fs-07d347f81cc925211.efs.ap-northeast-2.a
mazonaws.com:/ /mnt/efs
# 4. 마운트 확인
df -h | grep efs
ls -la /mnt/efs
Directory 권한 문제 해결
NGINX가
/var/cache/nginx/client_temp
디렉토리를 생성하려고 하는데 권한이 없어서 실패
하는 현상 발생
# 디렉토리 권한 (모두 읽기/실행 가능)
sudo chmod 755 /mnt/efs/pvc-a355ac6f-d51d-41c794c1-c50247e25d76/
# index.html 권한 (모두 읽기 가능)
sudo chmod 644 /mnt/efs/pvc-a355ac6f-d51d-41c794c1-c50247e25d76/i
ndex.html
# 소유자 설정 (fsGroup과 일치)
sudo chown R 5000050000 /mnt/efs/pvc-a355ac6f-d51d-41c794c1-c50
247e25d76/
EFS  EKS 6

# 확인
ls -la /mnt/efs/<pvc volume>/
Nginx Deploy
index.html 파일 생성
# PVC 디렉토리로 이동
cd /mnt/efs/<pvc name>
# index.html 생성
sudo tee index.html > /dev/null <<'EOF'
!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>psj0514.site</title>
<style>
body {
margin: 0;
min-height: 100vh;
display: flex;
justify-content: center;
align-items: center;
font-family: 'Pretendard', system-ui, sans-serif;
background: radial-gradient(circle at 20% 30%, #1e293b 0%, #0f17
2a 80%;
color: #e2e8f0;
overflow: hidden;
}
.card {
background: rgba(17, 24, 39, 0.9;
padding: 40px 48px;
border-radius: 20px;
box-shadow: 0 10px 35px rgba(0,0,0,0.5;
width: min(500px, 90vw);
text-align: center;
EFS  EKS 7

backdrop-filter: blur(12px);
animation: fadeIn 1s ease-in-out;
}
h1 
margin: 0 0 20px;
font-size: 28px;
background: linear-gradient(90deg, #3b82f6, #10b981;
-webkit-background-clip: text;
-webkit-text-fill-color: transparent;
}
p {
opacity: 0.85;
margin: 0 0 28px;
line-height: 1.6;
}
.btns {
display: flex;
gap: 14px;
flex-wrap: wrap;
justify-content: center;
}
a.btn {
flex: 1 1 180px;
display: inline-block;
text-align: center;
padding: 14px 18px;
border-radius: 12px;
text-decoration: none;
font-weight: 600;
background: linear-gradient(90deg, #2563eb, #1d4ed8;
color: #fff;
transition: all 0.3s ease;
box-shadow: 0 3px 10px rgba(37, 99, 235, 0.3;
}
a.btn:hover {
transform: translateY3px);
box-shadow: 0 8px 18px rgba(37, 99, 235, 0.4;
}
EFS  EKS 8

a.secondary {
background: linear-gradient(90deg, #10b981, #059669;
box-shadow: 0 3px 10px rgba(16, 185, 129, 0.3;
}
a.secondary:hover {
transform: translateY3px);
box-shadow: 0 8px 18px rgba(16, 185, 129, 0.4;
}
@keyframes fadeIn {
from {
opacity: 0;
transform: translateY10px);
}
to {
opacity: 1;
transform: translateY0;
}
}
</style>
</head>
<body>
<div class="card">
<h1Welcome to A lots of Injung Team</h1
<p>아래 버튼으로 Petclinic과 Test로 이동할 수 있습니다.</p>
<div class="btns">
<a class="btn" href="/petclinic"Move to Petclinic</a>
<a class="btn secondary" href="/test.jsp"Move to Test</a>
</div>
</div>
</body>
</html>
EOF
# 파일 확인
ls -lh index.html
cat index.html | head 10
# 예상 출력
EFS  EKS 9

-rw-r--r-- 1 root root 2.1K Nov 9 0451 index.html
!doctype html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>psj0514.site</title>
<style>
body {
margin: 0;
Nginx Deployment .yaml 파일 생성
cat > nginx-static.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
name: nginx-static
namespace: default
spec:
replicas: 2
selector:
matchLabels:
app: nginx-static
template:
metadata:
labels:
app: nginx-static
spec:
containers:
- name: nginx
image: nginx:alpine
ports:
- containerPort: 80
volumeMounts:
- name: efs-volume
mountPath: /usr/share/nginx/html
EFS  EKS 10

volumes:
- name: efs-volume
persistentVolumeClaim:
claimName: efs-static-web
---
apiVersion: v1
kind: Service
metadata:
name: nginx-static-service
namespace: default
spec:
type: ClusterIP
selector:
app: nginx-static
ports:
- port: 80
targetPort: 80
EOF
kubectl apply -f nginx-static.yaml
Nginx Pod에서 파일 확인
# Nginx Pod 내부에서 파일 확인
kubectl exec -it $(kubectl get pod -l app=nginx-static -o jsonpath='{.items
0.metadata.name}') -- ls -lh /usr/share/nginx/html/
# index.html 내용 일부 확인
kubectl exec -it $(kubectl get pod -l app=nginx-static -o jsonpath='{.items
0.metadata.name}') -- head 5 /usr/share/nginx/html/index.html
Ingress 수정 (ALB 라우팅 설정)
cat > ingress-updated.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: petclinic-ing
EFS  EKS 11

namespace: default
annotations:
kubernetes.io/ingress.class: alb
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
alb.ingress.kubernetes.io/load-balancer-name: petclinic-kr-alb
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-29
46775837287:certificate/9af46a5b-a1084eb4-a464-b2a7145d5ff8
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP" 80, {"HTTPS" 443'
alb.ingress.kubernetes.io/ssl-redirect: '443'
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/healthcheck-path: /petclinic/actuator/health
alb.ingress.kubernetes.io/healthcheck-port: traffic-port
alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
alb.ingress.kubernetes.io/success-codes: '200'
alb.ingress.kubernetes.io/healthy-threshold-count: '2'
alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
alb.ingress.kubernetes.io/tags: Environment=production,Project=petclini
c
spec:
ingressClassName: alb
rules:
- http:
paths:
- path: /petclinic
pathType: Prefix
backend:
service:
name: petclinic-svc
port:
number: 80
- path: /
pathType: Prefix
backend:
service:
name: nginx-static-service
port:
EFS  EKS 12

number: 80
EOF
kubectl apply -f ingress-updated.yaml
kubectl get ingress petclinic-ing
kubectl describe ingress petclinic-ing
EFS  EKS 13