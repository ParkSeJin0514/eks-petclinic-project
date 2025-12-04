ECR → EKS → RDS
RDS 연동까지는 이번 프로젝트에는 없었지만 어차피 Final Project 때 해야 하기 때문에
미리 해봄
Github에서 가져온 petclinic 같은 경우 버전이 달라서 수정해야 하는 게 많았음
어떤 petclinic을 쓰느냐에 따라 다르니 잘 찾아봐야함
아키텍처 구조
아키텍처
전체 트래픽 흐름
flowchart LR
%% ALB  static-web-svc → petclinic-svc  RDS
subgraph Internet
UUser Browser]
end
ECR  EKS  RDS 1

U |HTTPS 443| ALB"ALB (internet-facing)<br>Ingress: nginx-alb<br>T
arget type: ip"]:::alb
subgraph AWS_VPC"AWS VPC (petclinic-vpc)"]
subgraph EKS"petclinic-kr-eks"]
direction TB
LBC"AWS Load Balancer Controller"]:::ctrl
%%  WEB Tier ----------
subgraph WEB_Tier["WEB Tier NGINX"
direction TB
S_WEBService: static-web-svc<br>ClusterIP 80:::svc
P_WEB1Pod: nginx-web-1<br>nginx:alpine)):::pod
P_WEB2Pod: nginx-web-2<br>nginx:alpine)):::pod
EFSEFS efs-static-web<br>Mount: /usr/share/nginx/html)]:::db
S_WEB  P_WEB1
S_WEB  P_WEB2
P_WEB1  EFS
P_WEB2  EFS
W1"정적 콘텐츠 제공<br>(/  HTML/CSS/JS)<br>/petclinic/* → petclini
c-svc로 프록시"]:::note
P_WEB1  W1
end
%%  WAS Tier ----------
subgraph WAS_Tier["WAS Tier Petclinic App)"]
direction TB
S_WASService: petclinic-svc<br>ClusterIP 808080:::svc
P_WAS1Pod: petclinic-1<br>Spring Boot v5:::pod
P_WAS2Pod: petclinic-2<br>Spring Boot v5:::pod
Secret["Secret: petclinic-db-secret<br>DB 접속정보 (HOST/PORT/USE
R/PASS)"]:::dep
S_WAS  P_WAS1
ECR  EKS  RDS 2

S_WAS  P_WAS2
P_WAS1  Secret
P_WAS2  Secret
W2"/petclinic/  Liveness & Readiness Probe<br>/test.jsp → 테스트
페이지"]:::note
P_WAS1  W2
end
%%  In-cluster ----------
P_WEB1 |HTTP 80  petclinic.petclinic.svc.cluster.local| S_WAS
P_WEB2 |HTTP 80  petclinic.petclinic.svc.cluster.local| S_WAS
end
%%  Database ----------
subgraph RDS"RDS MySQL"]
DB[(petclinic-db<br>Multi-AZ:::db
end
end
%%  ALB routes ----------
ALB |HTTP 80| S_WEB
%%  WAS  RDS 
P_WAS1 |MySQL Connection| DB
P_WAS2 |MySQL Connection| DB
%%  Controller ----------
LBC . provisions . ALB
%%  Styles ----------
classDef alb fill:#ffebee,stroke:#b71c1c,color:#4a0000;
classDef svc fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
classDef dep fill:#f8f8f8,stroke:#999,color:#333;
classDef pod fill:#fff7e6,stroke:#f0a202,color:#5a3d00;
classDef ctrl fill:#e8f5e9,stroke:#2e7d32,color:#0e4d1c;
classDef db fill:#ede7f6,stroke:#5e35b1,color:#311b92;
classDef note fill:#f1f8e9,stroke:#558b2f,color:#1b5e20;
ECR  EKS  RDS 3

간소화한 트래픽 흐름
flowchart LR
%% ALB  static-web-svc → petclinic-svc
%%  ALB 
ALB"ALB (internet-facing)<br>Ingress: nginx-alb<br>Target type: ip"]:::al
b
%%  VPC / EKS 
subgraph AWS_VPC"AWS VPC (petclinic-vpc)"]
subgraph EKS"petclinic-kr-eks"]
direction LR
%%  WEB Tier ----------
subgraph WEB_Tier["WEB Tier NGINX"
direction TB
S_WEBService: static-web-svc<br>ClusterIP 80:::svc
P_WEB1Pod: nginx-web-1<br>nginx:alpine)):::pod
P_WEB2Pod: nginx-web-2<br>nginx:alpine)):::pod
EFSEFS efs-static-web<br>Mount: /usr/share/nginx/html)]:::db
S_WEB  P_WEB1
S_WEB  P_WEB2
P_WEB1  EFS
P_WEB2  EFS
W1"정적 콘텐츠 제공<br>(/  HTML/CSS/JS)<br>/petclinic/* → petclini
c-svc 로 프록시"]:::note
P_WEB1  W1
end
%%  HTTP 라우팅(라벨만 표현용) ----------
HTTP1"HTTP 80 →<br>petclinic.petclinic.svc.cluster.local"]:::dep
HTTP2"HTTP 80 →<br>petclinic.petclinic.svc.cluster.local"]:::dep
%%  WAS Tier ----------
subgraph WAS_Tier["WAS Tier Petclinic App)"]
ECR  EKS  RDS 4

direction TB
S_WASService: petclinic-svc<br>ClusterIP 808080:::svc
P_WAS1Pod: petclinic-1<br>Spring Boot v5:::pod
P_WAS2Pod: petclinic-2<br>Spring Boot v5:::pod
Secret["Secret: petclinic-db-secret<br>DB 접속정보 (HOST/PORT/USE
R/PASS)"]:::dep
S_WAS  P_WAS1
S_WAS  P_WAS2
P_WAS1  Secret
P_WAS2  Secret
W2"/petclinic/  Liveness & Readiness Probe<br>/test.jsp → 테스트
페이지"]:::note
P_WAS1  W2
end
%%  Controller ----------
LBC"AWS Load Balancer<br>Controller"]:::ctrl
end
end
%%  외부 → 내부 ----------
ALB |HTTP 80| S_WEB
%%  WEB  WAS 라벨 노드 경유로 화살표 배치 제어) ----------
P_WEB1  HTTP1  S_WAS
P_WEB2  HTTP2  S_WAS
%%  Controller  ALB 점선, 프로비저닝) ----------
LBC . provisions . ALB
%%  Styles ----------
classDef alb fill:#ffebee,stroke:#b71c1c,color:#4a0000;
classDef svc fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
classDef dep fill:#f8f8f8,stroke:#999,color:#333;
classDef pod fill:#fff7e6,stroke:#f0a202,color:#5a3d00;
classDef ctrl fill:#e8f5e9,stroke:#2e7d32,color:#0e4d1c;
ECR  EKS  RDS 5

classDef db fill:#ede7f6,stroke:#5e35b1,color:#311b92;
classDef note fill:#f1f8e9,stroke:#558b2f,color:#1b5e20;
🌐 Internet HTTPS 443
│
▼
┌────────────────────────────────────┐
│ AWS ALB Ingress Controller) │
│------------------------------------│
│  Scheme: internet-facing │
│  Listener: 80  443 redirect │
│  Certificate: ACM SSL 종단) │
│  Target-type: ip │
│  HealthCheck: GET /healthz │
└──────────────────┬─────────────────┘
│
Path: 모든 요청 "/" → NGINX
│
▼
Service: static-web-svc ClusterIP80
│
▼
Deployment: static-web (nginx:alpine, replicas=3
┌──────────────────────────────────────────────
──────────────┐
│ NGINX │
│------------------------------------------------------------│
│ 📁 정적 콘텐츠 (EFS) │
│ - /usr/share/nginx/html │
│  PVC efs-static-web │
│ │
│ ⚙ proxy 설정 (/etc/nginx/conf.d/default.conf) │
│ │
│ • `/` → 정적 HTML/CSS/JS (index.html 등) │
│ • `/petclinic/` → 프록시 → petclinic-svc:80/petclinic/ │
│ • `/test.jsp` → 프록시 → petclinic-svc:80/test.jsp │
│ • `/healthz`  200 OK ALB 헬스체크용) │
│ │
ECR  EKS  RDS 6

│ ➕ XForwarded-* 헤더 추가로 클라이언트 IP 전달 │
└──────────────────────────────────────────────
──────────────┘
│
│ (프록시 트래픽)
▼
Service: petclinic-svc ClusterIP808080
│
▼
Deployment: petclinic Spring Boot, replicas=3
┌──────────────────────────────────────────────
───────────────┐
│ Petclinic App (v5) │
│-------------------------------------------------------------│
│ • 포트: 8080 │
│ • 헬스체크: /petclinic/ │
│ • 추가 경로: /test.jsp JSP 테스트 페이지) │
│  DB 연결: │
│ - Secret: petclinic-db-secret │
│  HOST/PORT/NAME/USERNAME/PASSWORD  RDS MySQL 연결
│
│ • Liveness/ReadinessProbe: /petclinic/ │
└──────────────────────────────────────────────
───────────────┘
│
▼
🗄 AWS RDS MySQL
기본 구성
git clone으로 가져와서 pom.xml 파일 수정해줘야됨!
git clone으로 Github에서 가져오기
git clone https://github.com/spring-petclinic/spring-framework-petclinic.git
cd spring-framework-petclinic
ECR  EKS  RDS 7

pom.xml Troubleshooting
버전이 안맞아서 MySQL 호환 문제, Connector 문제 등 여러 문제가 있음
확인하고 고쳐야함
MySQL connector 문제
grep A 20 "<id>MySQL/id>" pom.xml
# MySQL connector artifact ID 구버전으로 인한 오류
# mysql-connector-java는 더 이상 사용하지 않고 mysql-connector-j 변경
sed -i 's/<artifactId>mysql-connector-java<\/artifactId>/<artifactId>mysql-
connector-j<\/artifactId>/' pom.xml
groupID 변경
# MySQL connector의 groupID가 mysql에서 com.mysql로 변경
# pom.xml에서 MySQL 드라이버 버전도 확인
grep "mysql-driver.version" pom.xml
# groupId 수정
sed -i 's/<groupId>mysql<\/groupId>/<groupId>com.mysql<\/groupId>/' p
om.xml
# 버전을 사용 가능한 버전으로 변경
sed -i 's/<mysql-driver.version>8.1.0\/mysql-driver.version>/<mysql-driv
er.version>8.0.33\/mysql-driver.version>/' pom.xml
pom.xml URL 문제
# &amp;가 들어있는데, 이게 datasource-config.xml로 전달되면서 문제 발생
cd ~/eks-config/spring-framework-petclinic
# URL에서 &amp;를 &로 변경
sed -i 's|<jdbc.url>jdbc:mysql://petclinic-kr-db.crone748rvgl.ap-northeast-
2.rds.amazonaws.com:3306/petclinic?useUnicode=true&amp;characterEn
coding=UTF8&amp;serverTimezone=Asia/Seoul</jdbc.url>|<jdbc.url>jdb
ECR  EKS  RDS 8

c:mysql://petclinic-kr-db.crone748rvgl.ap-northeast-2.rds.amazonaws.co
m:3306/petclinic</jdbc.url>|' pom.xml
Docker Build
Build할 때 태깅 붙혀주는게 가독성이 있음
Dockerfile 같은 경우 다양하게 구성이 가능해서 본인의 입맛대로 바꾸면 됨
Dockerfile 생성
# Multi-stage build Dockerfile for Spring Framework Petclinic
# Stage 1 Build stage
FROM maven:3.8.6-eclipse-temurin-17 AS build
WORKDIR /app
# Copy pom.xml first for dependency caching
COPY pom.xml .
# Download dependencies
RUN mvn dependency:go-offline B P MySQL
# Copy source code
COPY src ./src
# Build the application with MySQL profile
RUN mvn clean package DskipTests B P MySQL
# Stage 2 Runtime stage
FROM tomcat:10.1-jdk17-temurin
# Remove default Tomcat applications
RUN rm -rf /usr/local/tomcat/webapps/*
# Copy the WAR file from build stage as petclinic.war
COPY --from=build /app/target/petclinic.war /usr/local/tomcat/webapps/pe
tclinic.war
# Create ROOT directory and WEBINF structure for test.jsp
RUN mkdir -p /usr/local/tomcat/webapps/ROOT/WEBINF/lib
# Create minimal web.xml for ROOT application
RUN echo '?xml version="1.0" encoding="UTF8"?' > /usr/local/tomcat/
webapps/ROOT/WEBINF/web.xml && \
echo '<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"' >> /usr/loc
al/tomcat/webapps/ROOT/WEBINF/web.xml && \
echo ' xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"' >> /
usr/local/tomcat/webapps/ROOT/WEBINF/web.xml && \
ECR  EKS  RDS 9

echo ' xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee' >> /usr/
local/tomcat/webapps/ROOT/WEBINF/web.xml && \
echo ' http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"' >> /usr/loc
al/tomcat/webapps/ROOT/WEBINF/web.xml && \
echo ' version="4.0">' >> /usr/local/tomcat/webapps/ROOT/WEBINF/w
eb.xml && \
echo ' <display-name>Test Application</display-name>' >> /usr/local/t
omcat/webapps/ROOT/WEBINF/web.xml && \
echo '</web-app>' >> /usr/local/tomcat/webapps/ROOT/WEBINF/web.
xml
# Copy MySQL JDBC driver to ROOT/WEBINF/lib
COPY mysql-connector-j-8.0.33.jar /usr/local/tomcat/webapps/ROOT/WEB
-INF/lib/
# Copy test.jsp to ROOT
COPY test.jsp /usr/local/tomcat/webapps/ROOT/test.jsp
# Expose port
EXPOSE 8080
# Set environment variables for Tomcat
ENV CATALINA_OPTS"Xms512M Xmx1024M"
# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=
3 \
CMD curl -f http://localhost:8080/petclinic/ || exit 1
# Start Tomcat
CMD "catalina.sh", "run"]
Image Build
# 기존 이미지 확인 (선택사항)
docker images | grep petclinic
# 새 이미지 빌드
docker build -t petclinic:v2 .
로컬 테스트
# 컨테이너 실행
docker run -d -p 80808080 --name petclinic-test petclinic:v2
ECR  EKS  RDS 10

# 10초 정도 기다린 후 테스트
sleep 10
# /petclinic 경로 테스트
curl I http://localhost:8080/petclinic/
# 예상 결과: HTTP/1.1 200 또는 302
# 로그 확인
docker logs petclinic-test
# 테스트 완료 후 컨테이너 삭제
docker stop petclinic-test
docker rm petclinic-test
구성
ECR
태깅을 붙혀야 sha가 아니라 태깅으로 ECR에서 이미지를 Pull을 편하게 할 수 있음
ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
docker login --username AWS --password-stdin 946775837287.dkr.ecr.
ap-northeast-2.amazonaws.com
ECR 리포지토리 생성
aws ecr create-repository \
--repository-name petclinic \
--region ap-northeast-2 \
--image-scanning-configuration scanOnPush=true
이미지 태깅
# 이미지 태깅 (v2 태그)
docker tag petclinic:v3 \
ECR  EKS  RDS 11

946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:v3
# 이미지 태깅 (latest도 업데이트)
docker tag petclinic:v3 \
946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:latest
로
ECR Push
docker image가 쌓이면서 디스크를 엄청 잡아먹음
어느정도 쌓이면 꼭 정리 요망!
태깅된 이미지 ECR로 Push
# 1. 태깅
docker tag petclinic:latest 946775837287.dkr.ecr.ap-northeast-2.amazona
ws.com/petclinic:latest
# 2. 푸시
docker push 946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petcli
nic:latest
Docker 정리 명령어
docker system prune -a -f --volumes
설정
kubectl
계속해서 yaml 파일 수정 중
로드 밸런서 수정 완료
kubeconfig 업데이트
aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-kr-e
ks
EKS Secret 생성
ECR  EKS  RDS 12

K8s Secret은 민감한 정보를 안전하게 저장하는 객체
Password, API Key, Token, Database 정보 등 yaml 파일에 직접 쓰지 않고 etcd
에 저장하여 보안 강
kubectl create secret generic petclinic-db-secret \
--from-literal=DB_HOST=petclinic-kr-db.crone748rvgl.ap-northeast-2.rd
s.amazonaws.com \
--from-literal=DB_PORT3306 \
--from-literal=DB_NAME=petclinic \
--from-literal=DB_USERNAME=admin \
--from-literal=DB_PASSWORD=<password>
petclinic-app.yaml 파일 생성
# =========================
# 1 Petclinic Deployment
# =========================
apiVersion: apps/v1
kind: Deployment
metadata:
name: petclinic
namespace: default
labels:
app: petclinic
spec:
replicas: 3
selector:
matchLabels:
app: petclinic
template:
metadata:
labels:
app: petclinic
spec:
securityContext:
runAsUser: 0
runAsGroup: 0
fsGroup: 0
ECR  EKS  RDS 13

containers:
- name: app
image: 946775837287.dkr.ecr.ap-northeast-2.amazonaws.com/petcli
nic:v5
imagePullPolicy: Always
ports:
- containerPort: 8080
resources:
requests:
cpu: 300m
memory: 300Mi
limits:
cpu: 500m
memory: 500Mi
env:
- name: DB_HOST
valueFrom: { secretKeyRef: { name: petclinic-db-secret, key: DB_H
OST  
- name: DB_PORT
valueFrom: { secretKeyRef: { name: petclinic-db-secret, key: DB_P
ORT  
- name: DB_NAME
valueFrom: { secretKeyRef: { name: petclinic-db-secret, key: DB_N
AME  
- name: DB_USERNAME
valueFrom: { secretKeyRef: { name: petclinic-db-secret, key: DB_U
SERNAME  
- name: DB_PASSWORD
valueFrom: { secretKeyRef: { name: petclinic-db-secret, key: DB_P
ASSWORD  
readinessProbe:
httpGet:
path: /petclinic/
port: 8080
initialDelaySeconds: 30
periodSeconds: 10
timeoutSeconds: 5
failureThreshold: 3
ECR  EKS  RDS 14

livenessProbe:
httpGet:
path: /petclinic/
port: 8080
initialDelaySeconds: 60
periodSeconds: 30
timeoutSeconds: 5
failureThreshold: 5
---
# ======================
# 2 Petclinic Service
# ======================
apiVersion: v1
kind: Service
metadata:
name: petclinic-svc
namespace: default
labels:
app: petclinic
spec:
selector:
app: petclinic
ports:
- name: http
port: 80
targetPort: 8080
type: ClusterIP
---
# ==========================================
# 3 NGINX Config (프록시 + 정적 + 헬스체크)
# ==========================================
apiVersion: v1
kind: ConfigMap
metadata:
name: static-web-nginx-conf
namespace: default
data:
default.conf: |
ECR  EKS  RDS 15

server {
listen 80;
server_name _;
# 헬스체크: ALB가 여기로 체크
location /healthz {
return 200 "ok";
add_header Content-Type text/plain;
}
# 정적 파일 (EFS)
location / {
root /usr/share/nginx/html;
try_files $uri $uri/ 404;
}
# Petclinic 프록시
location /petclinic/ {
proxy_pass http://petclinic-svc.default.svc.cluster.local:80;
proxy_set_header Host $host;
proxy_set_header XReal-IP $remote_addr;
proxy_set_header XForwarded-For $proxy_add_x_forwarded_for;
proxy_set_header XForwarded-Proto $scheme;
proxy_read_timeout 60s;
proxy_connect_timeout 5s;
}
# test.jsp 프록시
location /test.jsp {
proxy_pass http://petclinic-svc.default.svc.cluster.local:80/test.jsp;
proxy_set_header Host $host;
proxy_set_header XReal-IP $remote_addr;
proxy_set_header XForwarded-For $proxy_add_x_forwarded_for;
proxy_set_header XForwarded-Proto $scheme;
proxy_read_timeout 60s;
proxy_connect_timeout 5s;
ECR  EKS  RDS 16

}
}
---
# =============================================
# 4) Static Web Deployment NGINX  EFS  Conf)
# =============================================
apiVersion: apps/v1
kind: Deployment
metadata:
name: static-web
namespace: default
labels:
app: static-web
spec:
replicas: 3
selector:
matchLabels:
app: static-web
template:
metadata:
labels:
app: static-web
spec:
securityContext:
fsGroup: 50000
containers:
- name: nginx
image: nginx:alpine
ports:
- containerPort: 80
resources:
requests:
cpu: 100m
memory: 200Mi
limits:
cpu: 300m
memory: 300Mi
volumeMounts:
ECR  EKS  RDS 17

- name: efs-storage
mountPath: /usr/share/nginx/html
- name: nginx-conf
mountPath: /etc/nginx/conf.d
readinessProbe:
httpGet:
path: /healthz
port: 80
initialDelaySeconds: 10
periodSeconds: 5
timeoutSeconds: 3
failureThreshold: 3
livenessProbe:
httpGet:
path: /healthz
port: 80
initialDelaySeconds: 20
periodSeconds: 10
timeoutSeconds: 3
failureThreshold: 3
volumes:
- name: efs-storage
persistentVolumeClaim:
claimName: efs-static-web
- name: nginx-conf
configMap:
name: static-web-nginx-conf
---
# =========================
# 5 Static Web Service
# =========================
apiVersion: v1
kind: Service
metadata:
name: static-web-svc
namespace: default
labels:
app: static-web
ECR  EKS  RDS 18

spec:
selector:
app: static-web
ports:
- name: http
port: 80
targetPort: 80
type: ClusterIP
---
# ============================
# 6 Ingress ALB  NGINX 경유)
# ============================
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: petclinic-ing
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
alb.ingress.kubernetes.io/healthcheck-path: /healthz
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
ECR  EKS  RDS 19

rules:
- http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: static-web-svc
port:
number: 80
test.jsp는 git clone 한 petclinic에 넣어놓음
넣은 상태에서 Dockerfile로 Build 해야함!
% page import = "java.net.InetAddress" %
% page import = "java.sql.*" %
<html>
<head>
<title>Hello from JSP on EKS/title>
%! String message = "Hello World. From JSP test page  Tomcat is runnin
g on EKS!"; %
% InetAddress inet= InetAddress.getLocalHost(); %
</head>
<body>
<hr color="#000000" />
<center>
<h2><font color="#3366cc"% message%/font></h2
<h3><font color="#0000ff"% new java.util.Date()%/font></h3
<hr color="#000000" />
<h3%=application.getServerInfo()%/h3
<h3Host Name : %=inet.getHostName() %/h3
<h3Host Address : %=inet.getHostAddress() %/h3
<h3Client IP  %=request.getRemoteAddr()%/h3
<h3Client IPXFORWARDEDFOR  %=request.getHeader("x-forwarde
d-for"%/h3
<hr color="#000000" />
<h3ALL HTTP HEADERS/h3
<font><%
ECR  EKS  RDS 20

java.util.Enumeration names = request.getHeaderNames();
while(names.hasMoreElements()){
String name = String) names.nextElement();
out.println(name + "BR" + request.getHeader(name) + "BRBR");
%
</font>
<hr color="#000000" />
<h3MYSQL CONNECTION TEST/h3
<font>
%
Connection conn = null;
try{
String dbHost  System.getenv("DB_HOST");
String dbPort  System.getenv("DB_PORT");
String dbName = "jdbcTest";
String dbUser  System.getenv("DB_USERNAME");
String dbPass  System.getenv("DB_PASSWORD");
String url = "jdbc:mysql://" + dbHost + ":" + dbPort + "/" + dbName + "?u
seSSL=false&serverTimezone=UTC";
Class.forName("com.mysql.cj.jdbc.Driver");
conn=DriverManager.getConnection(url, dbUser, dbPass);
out.println("<h3 style='color:green;'DB Connected</h3");
}
catch(Exception e) {
out.println("<h3 style='color:red;'DB Connection Failed</h3");
out.println("<p>" + e.getMessage() + "</p>");
}
finally {
if(conn ! null) try { conn.close(); } catch(Exception e) {}
}
%
</font>
</body>
</html>
배포
ECR  EKS  RDS 21

kubectl apply -f petclinic-app.yaml
ECR  EKS  RDS 22