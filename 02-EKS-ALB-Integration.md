EKS → ALB
EKS  NLB로 하려고 했으나 ALB로 연결하는 작업을 진행
대규모 EKS에서 보통 여러 마이크로서비스가 존재
각 서비스마다 NLB를 만들면 비용과 관리 복잡도 폭증
ALB는 하나로 여러 Ingress Rule을 처리 가능
/api/* /auth/* /static/* 등 경로 기반 분리
설치
AWS Load Balancer Controller
ALB를 배포하다가 권한이 제대로 주어지지 않아서 에러 발생
공식 문서를 보면 1.22 버전 이후로 부터는 새로운 권한을 2개 추가해야함
EKS 버전 꼭 확인 요망!
참고 :
참고 : https://github.com/aws/eks-charts?tab=readme-ov-file#aws-load-
balancer-controller
AWS Load Balancer Controller가 AWS 리소스를 관리하는 데 필요한 IAM 권한을
생성
# v2.14.1 IAM Policy 다운로드
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/
aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
# IAM Policy 생성
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json
IRSA IAM Roles for Service Accounts) 서비스 계정 권한 생성
ACCOUNT_ID$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN"arn:aws:iam::$ACCOUNT_ID}:policy/AWSLoadBalancerCon
trollerIAMPolicy"
eksctl create iamserviceaccount \
EKS  ALB 1

--cluster=petclinic-kr-eks \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=$POLICY_ARN \
--override-existing-serviceaccounts \
--region=ap-northeast-2 \
--approve
Helm으로 Controller 설치
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller
\
-n kube-system \
--set clusterName=petclinic-kr-eks \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=ap-northeast-2 \
--set vpcId=vpc-05dbb2a501951af95
태그 설정
Subnet
External ALB 태그 붙히기
aws ec2 create-tags \
--resources subnet-0b59cec4b30885866 subnet-07c475fa1608156d4 \
--tags \
Key=kubernetes.io/role/elb,Value=1 \
Key=kubernetes.io/cluster/petclinic-kr-eks,Value=shared
및 생성
Service Ingress
ClusterIP Service 생성
EKS  ALB 2

apiVersion: v1
kind: Service
metadata:
name: test-service
spec:
type: ClusterIP
selector:
app: test
ports:
- protocol: TCP
port: 80
targetPort: 80
Ingress 생성 (ALB)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: test-ingress
annotations:
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP" 80'
alb.ingress.kubernetes.io/healthcheck-path: /
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
alb.ingress.kubernetes.io/success-codes: '200'
alb.ingress.kubernetes.io/tags: Environment=production,Project=petclini
c
spec:
ingressClassName: alb
rules:
- http:
paths:
- path: /
pathType: Prefix
backend:
EKS  ALB 3

service:
name: test-service
port:
number: 80
Subnet에 제대로 Tag가 붙었는지 확인
aws ec2 describe-subnets \
--subnet-ids subnet-0b59cec4b30885866 subnet-07c475fa1608156d4 \
--query 'Subnets[*].{SubnetId:SubnetId,Tags:Tags[?contains(Key,`kubern
etes`)]}' \
--output table
테스트
테스트 배포
kubectl apply -f ~/eks-config/test-service.yaml
kubectl apply -f ~/eks-config/test-ingress.yaml
상태 확인
# Ingress 상태 확인
kubectl get ingress test-ingress
# ALB DNS 확인
kubectl get ingress test-ingress -o jsonpath='{.status.loadBalancer.ingress
0.hostname}'
# Pod 상태 확인
kubectl get pods -o wide
# Service 확인
kubectl get svc test-service
# Ingress 상세 정보
kubectl describe ingress test-ingress
EKS  ALB 4

# Controller 로그
kubectl logs -n kube-system deployment/aws-load-balancer-controller
# ALB 확인
aws elbv2 describe-load-balancers \
--query 'LoadBalancers[?contains(LoadBalancerName, `k8s-default-testi
ngr`)]'
# Subnet 태그 확인
aws ec2 describe-subnets \
--subnet-ids subnet-0b59cec4b30885866 \
--query 'Subnets[0].Tags'
EKS  ALB 5