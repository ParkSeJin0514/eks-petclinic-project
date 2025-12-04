# AWS EKS 기반 Kubernetes 실습 가이드

## 📋 프로젝트 개요

이 저장소는 AWS EKS(Elastic Kubernetes Service)를 활용한 컨테이너 오케스트레이션 및 클라우드 네이티브 애플리케이션 배포를 위한 종합 가이드입니다. Petclinic 애플리케이션을 기반으로 실제 프로덕션 환경에서 사용할 수 있는 다양한 AWS 서비스 통합 및 모니터링 구성 방법을 다룹니다.

### 🎯 학습 목표

- AWS EKS 클러스터 구축 및 관리
- Kubernetes 네이티브 서비스 배포
- AWS 관리형 서비스(ALB, EFS, RDS, ECR) 통합
- 컨테이너 이미지 관리 및 배포 자동화
- 프로메테우스/그라파나 기반 모니터링 구축
- K6를 활용한 부하 테스트 및 오토스케일링

---

## 📚 문서 구성

### [1. EKS Cluster 구성](./01-EKS-Cluster-Setup.md)

**주요 내용**
- EKS 클러스터 기본 아키텍처 설계
- VPC, 서브넷, 보안 그룹 구성
- Bastion Server 및 Management Instance 설정
- IAM 역할 및 정책 구성
- eksctl을 이용한 클러스터 생성
- Worker Node 배포 및 관리

**핵심 개념**
```
Internet → Public Subnet (Bastion) 
  → Private Subnet (Management Instance) 
    → EKS Control Plane (AWS Managed) 
      → Worker Nodes (Auto Scaling Group)
```

---

### [2. EKS와 ALB 통합](./02-EKS-ALB-Integration.md)

**주요 내용**
- AWS Load Balancer Controller 설치
- ALB Ingress Controller 구성
- Path 기반 라우팅 설정
- 서비스별 트래픽 분산 구현
- HTTPS/SSL 인증서 관리

**왜 ALB인가?**
- 마이크로서비스 환경에서 효율적인 트래픽 관리
- 경로 기반 라우팅으로 단일 로드밸런서로 여러 서비스 처리
- NLB 대비 비용 효율적
- Layer 7 로드밸런싱 기능

---

### [3. EKS와 EFS 스토리지 통합](./03-EKS-EFS-Storage.md)

**주요 내용**
- Amazon EFS 파일 시스템 생성
- EFS CSI Driver 설치 및 구성
- PersistentVolume 및 PersistentVolumeClaim 설정
- 정적 웹 콘텐츠 서빙
- 다중 Pod 간 스토리지 공유

**사용 사례**
- 여러 Pod에서 동시에 읽기/쓰기가 필요한 경우
- 정적 파일, 미디어 콘텐츠, 공유 설정 파일
- StatefulSet이 아닌 Deployment에서의 영구 스토리지

---

### [4. ECR-EKS-RDS 전체 통합](./04-ECR-EKS-RDS-Integration.md)

**주요 내용**
- Amazon ECR 프라이빗 레지스트리 구성
- 애플리케이션 컨테이너화 및 이미지 빌드
- Spring Boot Petclinic 배포
- RDS MySQL 데이터베이스 연동
- NGINX를 통한 정적/동적 콘텐츠 분리
- 전체 아키텍처 트래픽 흐름

**아키텍처 패턴**
```
User → ALB → Ingress 
  → Static Web Service (NGINX + EFS)
  → Application Service (Petclinic + RDS MySQL)
```

**핵심 통합 포인트**
- ECR 이미지 풀 권한 설정
- RDS 보안 그룹 및 접근 제어
- 환경 변수를 통한 DB 연결 정보 관리
- Health Check 엔드포인트 구성

---

### [5. Prometheus & Grafana 모니터링 스택](./05-EKS-Monitoring-Stack.md)

**주요 내용**
- Grafana Cloud 연동
- Grafana Alloy를 통한 메트릭 수집
- Kubernetes 클러스터 메트릭 모니터링
- kube-state-metrics 배포
- 대시보드 구성 및 알림 설정

**모니터링 지표**
- 클러스터 리소스 사용률 (CPU, Memory)
- Pod 상태 및 헬스체크
- Container 메트릭
- 네트워크 I/O
- 스토리지 사용량

**두 가지 접근 방식**
- **Grafana Cloud** : 관리형 서비스로 빠른 구성
- **Grafana Local** : 자체 호스팅으로 완전한 제어

---

### [6. K6를 활용한 부하 테스트](./06-Load-Testing-K6.md)

**주요 내용**
- HPA(Horizontal Pod Autoscaler) 구성
- Metrics Server 설치 및 검증
- K6 부하 테스트 시나리오 작성
- CPU/Memory 기반 오토스케일링
- 부하 테스트 결과 분석

**HPA 구성 예시**
- minReplicas : 2
- maxReplicas : 4
- Target CPU Utilization : 50%
- Scale Up/Down 정책 설정

---

## 🏗️ 전체 인프라 아키텍처

```
                          ┌─────────────┐
                          │   Internet  │
                          └──────┬──────┘
                                 │
                          ┌──────▼──────┐
                          │     ALB     │
                          │ (public)    │
                          └──────┬──────┘
                                 │
                 ┌───────────────┴───────────────┐
                 │                               │
          ┌──────▼──────┐               ┌───────▼────────┐
          │   Ingress   │               │    Ingress     │
          │  (NGINX)    │               │  (Petclinic)   │
          └──────┬──────┘               └───────┬────────┘
                 │                               │
          ┌──────▼──────┐               ┌───────▼────────┐
          │ Static Web  │               │  Application   │
          │   Service   │               │    Service     │
          │  (ClusterIP)│               │  (ClusterIP)   │
          └──────┬──────┘               └───────┬────────┘
                 │                               │
          ┌──────▼──────┐               ┌───────▼────────┐
          │  NGINX Pods │               │ Petclinic Pods │
          │  (Deployment│               │  (Deployment)  │
          │  + EFS)     │               │                │
          └─────────────┘               └───────┬────────┘
                                                 │
                                         ┌───────▼────────┐
                                         │   RDS MySQL    │
                                         │  (Multi-AZ)    │
                                         └────────────────┘
```

---

## 🛠️ 기술 스택

### Infrastructure & Cloud
- **Cloud Provider** : AWS (Amazon Web Services)
- **Container Orchestration** : Amazon EKS (Kubernetes)
- **Networking** : VPC, Subnets, Security Groups
- **Load Balancing** : Application Load Balancer (ALB)
- **Storage** : Amazon EFS (Elastic File System)
- **Database** : Amazon RDS for MySQL
- **Container Registry** : Amazon ECR

### Kubernetes Components
- **Ingress Controller** : AWS Load Balancer Controller
- **Storage Driver** : EFS CSI Driver
- **Metrics** : Metrics Server, kube-state-metrics
- **Autoscaling** : Horizontal Pod Autoscaler (HPA)

### Monitoring & Observability
- **Metrics Collection** : Grafana Alloy
- **Metrics Storage** : Prometheus
- **Visualization** : Grafana
- **Logging** : CloudWatch (optional)

### CI/CD & Testing
- **Load Testing** : K6
- **Image Building** : Docker
- **Version Control** : Git/GitHub

### Application Stack
- **Backend** : Spring Boot (Java)
- **Frontend** : NGINX
- **Database** : MySQL 8.0
- **Sample Application** : Spring Petclinic

---

## 🚀 시작하기

### 사전 요구사항

```bash
# AWS CLI
aws --version

# kubectl
kubectl version --client

# eksctl
eksctl version

# Helm
helm version

# Docker
docker --version
```

### 환경 설정

1. **AWS 자격 증명 구성**
```bash
aws configure
# AWS Access Key ID 입력
# AWS Secret Access Key 입력
# Default region: ap-northeast-2
```

2. **프로젝트 클론**
```bash
git clone <repository-url>
cd <repository-name>
```

3. **단계별 실습 진행**
- 각 문서를 순서대로 따라가며 실습 진행
- 01번부터 06번까지 순차적으로 구성

---

## 📖 상세 가이드 활용법

### 초급자 (Kubernetes 입문자)
1. [01. EKS Cluster 구성](./01-EKS-Cluster-Setup.md) - 기본 인프라 이해
2. [02. ALB 통합](./02-EKS-ALB-Integration.md) - 로드밸런싱 개념
3. [03. EFS 스토리지](./03-EKS-EFS-Storage.md) - 영구 스토리지 개념

### 중급자 (AWS/K8s 경험자)
1. [04. ECR-EKS-RDS 통합](./04-ECR-EKS-RDS-Integration.md) - 전체 아키텍처 구현
2. [05. 모니터링 스택](./05-EKS-Monitoring-Stack.md) - Observability 구축
3. [06. 부하 테스트](./06-Load-Testing-K6.md) - 성능 최적화

### 고급자 (Production 환경 구축)
- 모든 문서를 통합하여 프로덕션급 환경 구축
- 보안 강화 (IAM 최소 권한, Network Policy)
- 비용 최적화 (Spot Instances, Resource Requests/Limits)
- 고가용성 구성 (Multi-AZ, Auto Scaling)

---

## 🔧 트러블슈팅

### 일반적인 문제와 해결 방법

**1. EKS 클러스터 접근 불가**
```bash
# kubeconfig 업데이트
aws eks update-kubeconfig --region ap-northeast-2 --name <cluster-name>

# 연결 테스트
kubectl get nodes
```

**2. Pod가 Pending 상태**
```bash
# 이벤트 확인
kubectl describe pod <pod-name>

# 노드 리소스 확인
kubectl top nodes
```

**3. ALB Ingress 생성 안됨**
```bash
# AWS Load Balancer Controller 로그 확인
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# IAM 권한 확인
aws iam get-policy --policy-arn <policy-arn>
```

**4. RDS 연결 실패**
```bash
# 보안 그룹 확인
aws ec2 describe-security-groups --group-ids <sg-id>

# 환경 변수 확인
kubectl exec <pod-name> -- env | grep DB
```

---

## 📊 프로젝트 진행 체크리스트

- [ ] AWS 계정 및 IAM 사용자 설정
- [ ] VPC 및 네트워크 구성 완료
- [ ] EKS 클러스터 생성 및 Worker Node 배포
- [ ] kubectl 접근 확인
- [ ] AWS Load Balancer Controller 설치
- [ ] ALB Ingress 생성 및 테스트
- [ ] EFS 파일 시스템 생성 및 마운트
- [ ] ECR 레지스트리 생성 및 이미지 푸시
- [ ] RDS MySQL 인스턴스 생성
- [ ] Petclinic 애플리케이션 배포
- [ ] 정적 웹 서버 배포 (NGINX + EFS)
- [ ] 엔드투엔드 트래픽 흐름 테스트
- [ ] Prometheus & Grafana 모니터링 구성
- [ ] HPA 설정 및 테스트
- [ ] K6 부하 테스트 실행

---

## 💡 Best Practices

### 보안
- ✅ 최소 권한 원칙 (Least Privilege) IAM 정책 적용
- ✅ 프라이빗 서브넷에 워커 노드 배치
- ✅ Security Group 규칙 최소화
- ✅ Secret 관리 (AWS Secrets Manager, External Secrets)
- ✅ 네트워크 정책(Network Policy) 적용

### 성능
- ✅ Resource Requests & Limits 설정
- ✅ HPA 구성으로 자동 확장
- ✅ Node Affinity 및 Pod Affinity 활용
- ✅ 적절한 readiness/liveness probe 설정

### 비용 최적화
- ✅ Spot Instances 활용 (워커 노드)
- ✅ Auto Scaling 정책 최적화
- ✅ 미사용 리소스 정리 (ELB, EBS, EIP)
- ✅ CloudWatch 로그 보관 정책 설정

### 모니터링
- ✅ 핵심 메트릭 대시보드 구성
- ✅ 알람 임계값 설정
- ✅ 로그 수집 및 분석 파이프라인
- ✅ 분산 추적(Distributed Tracing) 도입

---

## 🙏 참고 자료

### 공식 문서
- [Amazon EKS 사용 설명서](https://docs.aws.amazon.com/eks/)
- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Amazon EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)

### 추가 학습 자료
- [EKS Workshop](https://www.eksworkshop.com/)
- [Kubernetes By Example](https://kubernetesbyexample.com/)
- [K6 Documentation](https://k6.io/docs/)
- [Grafana Labs](https://grafana.com/docs/)

---

**Last Updated**: 2024-12-04  
**Author**: Cloud Engineering Practice  
**Version**: 1.0.0