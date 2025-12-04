# 04. ECR - EKS - RDS 통합

## 📋 개요

Spring Boot Petclinic 애플리케이션을 컨테이너화하여 ECR에 푸시하고, EKS에 배포한 후 RDS MySQL과 연동합니다.

## 🐳 ECR 레지스트리 생성

```bash
# ECR 레포지토리 생성
aws ecr create-repository \
  --repository-name petclinic \
  --region ap-northeast-2

# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com
```

## 🏗️ Petclinic 애플리케이션 빌드

### 1. Spring Boot Petclinic 클론

```bash
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

### 2. Maven 빌드

```bash
./mvnw clean package -DskipTests
```

### 3. Docker 이미지 빌드

**Dockerfile**:
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# 이미지 빌드
docker build -t petclinic:latest .

# ECR에 태그
docker tag petclinic:latest <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:latest

# ECR에 푸시
docker push <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:latest
```

## 🗄️ RDS MySQL 생성

```bash
# RDS MySQL 생성
aws rds create-db-instance \
  --db-instance-identifier petclinic-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password <password> \
  --allocated-storage 20 \
  --vpc-security-group-ids <sg-id> \
  --db-subnet-group-name <subnet-group> \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --region ap-northeast-2
```

### Security Group 설정

```bash
# RDS Security Group에 EKS Worker Node에서 접근 허용
aws ec2 authorize-security-group-ingress \
  --group-id <rds-sg-id> \
  --protocol tcp \
  --port 3306 \
  --source-group <eks-worker-sg-id>
```

## 📦 Kubernetes Deployment

### ConfigMap (DB 설정)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: petclinic-config
data:
  SPRING_PROFILES_ACTIVE: "mysql"
  SPRING_DATASOURCE_URL: "jdbc:mysql://<rds-endpoint>:3306/petclinic"
```

### Secret (DB 비밀번호)

```bash
kubectl create secret generic petclinic-secret \
  --from-literal=SPRING_DATASOURCE_USERNAME=admin \
  --from-literal=SPRING_DATASOURCE_PASSWORD=<password>
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: petclinic
  template:
    metadata:
      labels:
        app: petclinic
    spec:
      containers:
        - name: petclinic
          image: <account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: petclinic-config
            - secretRef:
                name: petclinic-secret
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
```

### Service 및 Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: petclinic-svc
spec:
  type: ClusterIP
  selector:
    app: petclinic
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: petclinic-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: petclinic-svc
                port:
                  number: 8080
```

## ✅ 배포 및 검증

```bash
# 리소스 배포
kubectl apply -f petclinic-configmap.yaml
kubectl apply -f petclinic-deployment.yaml
kubectl apply -f petclinic-service.yaml
kubectl apply -f petclinic-ingress.yaml

# Pod 상태 확인
kubectl get pods -l app=petclinic

# 로그 확인
kubectl logs -f <pod-name>

# Ingress 확인
kubectl get ingress petclinic-ingress

# 애플리케이션 접속
curl http://<alb-dns>/
```

## 🎯 다음 단계

[05. EKS Monitoring Stack](./05-EKS-Monitoring-Stack.md)