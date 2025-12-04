# 03. EKS - EFS Storage 연동

## 📋 개요

Amazon EFS(Elastic File System)를 EKS 클러스터와 연동하여 여러 Pod에서 동시에 접근 가능한 영구 스토리지를 구성합니다. 정적 웹 콘텐츠 서빙과 같이 ReadWriteMany 접근이 필요한 경우에 사용합니다.

## 🗄️ EFS 파일 시스템 생성

### 1. EFS 생성

```bash
# EFS 파일 시스템 생성
aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=petclinic-kr-efs \
  --region ap-northeast-2
```

### 2. 서브넷 확인

```bash
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
```

### 3. Security Group 설정

```bash
# EKS 클러스터 Security Group 확인
aws eks describe-cluster \
  --name petclinic-kr-eks \
  --region ap-northeast-2 \
  --query 'cluster.resourcesVpcConfig.{ClusterSG:clusterSecurityGroupId,SecurityGroupIds:securityGroupIds}' \
  --output json

# NFS 포트 (2049) 허용
aws ec2 authorize-security-group-ingress \
  --group-id <nfs-sg-id> \
  --protocol tcp \
  --port 2049 \
  --source-group <nfs-sg-id> \
  --region ap-northeast-2

# Worker Node SG에서 NFS SG로 접근 허용
aws ec2 authorize-security-group-ingress \
  --group-id <nfs-sg-id> \
  --protocol tcp \
  --port 2049 \
  --source-group <worker-node-sg-id> \
  --region ap-northeast-2
```

### 4. EFS Mount Target 생성

```bash
# ap-northeast-2a에 마운트 타겟 생성
aws efs create-mount-target \
  --file-system-id <efs-id> \
  --subnet-id subnet-xxxxx \
  --security-groups <nfs-sg-id> \
  --region ap-northeast-2

# ap-northeast-2c에 마운트 타겟 생성
aws efs create-mount-target \
  --file-system-id <efs-id> \
  --subnet-id subnet-yyyyy \
  --security-groups <nfs-sg-id> \
  --region ap-northeast-2

# 마운트 타겟 상태 확인
aws efs describe-mount-targets \
  --file-system-id <efs-id> \
  --region ap-northeast-2 \
  --query 'MountTargets[].[MountTargetId,LifeCycleState,AvailabilityZoneName]' \
  --output table
```

## 🔌 EFS CSI Driver 설치

### 1. OIDC Provider 연결

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster petclinic-kr-eks \
  --region ap-northeast-2 \
  --approve
```

### 2. IAM 정책 생성

```bash
# IAM 정책 다운로드
curl -o iam-policy-efs.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json

# IAM 정책 생성
aws iam create-policy \
  --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
  --policy-document file://iam-policy-efs.json \
  --region ap-northeast-2
```

### 3. Service Account 생성

```bash
eksctl create iamserviceaccount \
  --cluster petclinic-kr-eks \
  --namespace kube-system \
  --name efs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve \
  --region ap-northeast-2
```

### 4. EFS CSI Driver 설치

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-2.0"

# 설치 확인
kubectl get pods -n kube-system | grep efs
```

## 📦 StorageClass 및 PVC 생성

### StorageClass 생성

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: <efs-id>
  directoryPerms: "777"
```

```bash
kubectl apply -f efs-storageclass.yaml
```

### PersistentVolumeClaim 생성

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-static-web
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f efs-pvc.yaml
kubectl get pvc efs-static-web
```

## 🔧 EFS 마운트 및 권한 설정

### Management Instance에서 EFS 마운트

```bash
# NFS 유틸리티 설치
sudo apt-get update
sudo apt-get install -y nfs-common

# 마운트 포인트 생성
sudo mkdir -p /mnt/efs

# EFS 마운트
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport \
  <efs-id>.efs.ap-northeast-2.amazonaws.com:/ /mnt/efs

# 마운트 확인
df -h | grep efs
ls -la /mnt/efs
```

### 디렉토리 권한 설정

```bash
# PVC 볼륨 디렉토리 찾기
ls -la /mnt/efs/

# 권한 설정
sudo chmod 755 /mnt/efs/<pvc-volume-id>/
sudo chown -R 5000:5000 /mnt/efs/<pvc-volume-id>/

# index.html 파일 생성
sudo tee /mnt/efs/<pvc-volume-id>/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>EKS EFS Test</title>
</head>
<body>
    <h1>Hello from EFS!</h1>
    <p>This static content is served from Amazon EFS.</p>
</body>
</html>
EOF

# 파일 권한 설정
sudo chmod 644 /mnt/efs/<pvc-volume-id>/index.html
```

## 🌐 NGINX Deployment

### Deployment 생성

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-web
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-web
  template:
    metadata:
      labels:
        app: static-web
    spec:
      securityContext:
        fsGroup: 5000
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: efs-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: efs-storage
          persistentVolumeClaim:
            claimName: efs-static-web
```

### Service 및 Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: static-web-svc
spec:
  type: ClusterIP
  selector:
    app: static-web
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: static-web-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /static
            pathType: Prefix
            backend:
              service:
                name: static-web-svc
                port:
                  number: 80
```

## ✅ 검증

```bash
# Pod 상태 확인
kubectl get pods -l app=static-web

# PVC 확인
kubectl get pvc efs-static-web

# Ingress 확인
kubectl get ingress static-web-ingress

# 웹 브라우저 또는 curl로 테스트
curl http://<alb-dns>/static/
```

## 📚 참고 자료

- [Amazon EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
- [EFS User Guide](https://docs.aws.amazon.com/efs/)

## 🎯 다음 단계

[04. ECR - EKS - RDS 통합](./04-ECR-EKS-RDS-Integration.md)