# 4. Ingress Controller 생성

### 4.0 콘솔 크레덴셜 더하기
#### 4.0.1 설명
- EKS 클러스터는 클러스터 접근 제어를 위해 IAM entity(사용자 또는 역할)를 사용합니다. 
- 해당 rule은 aws-auth라는 ConfigMap에서 실행됩니다. 
- 기본적으로 클러스터를 생성하는데 사용된 IAM entity에는 컨트롤 플레인에서 클러스터 RBAC 구성의 system:masters 권한이 자동적으로 부여됩니다.

#### 4.0.2 현재상황

- 현재 EKS에서 eks-demo 클러스터를 클릭하면

- "현재 사용자 또는 역할이 이 EKS 클러스터에 있는 Kubernetes 객체에 액세스할 수 없습니다.
이는 현재 사용자 또는 역할에 클러스터 리소스를 설명할 Kubernetes RBAC 권한이 없거나 클러스터의 인증 구성 맵에 항목이 없기 때문일 수 있습니다."
이렇게 뜸

- Cloud9의 IAM credential을 통해, 클러스터를 생성하였기 때문에 Amazon EKS 콘솔창 에서 해당 클러스터 정보를 확인하기 위해서는 실제 콘솔에 접근할 IAM entity(사용자 또는 역할)의 AWS Console credential을 클러스터에 추가하는 작업이 필요합니다.


#### 4.0.3 role ARN 정의
```
rolearn=$(aws cloud9 describe-environment-memberships --environment-id=$C9_PID | jq -r '.memberships[].userArn')
```
```
echo ${rolearn}
```


### 4.1 AWS Load Balancer 컨트롤러 생성
#### 4.1.1 manifests 이름의 폴더 생성
- /home/ec2-user/environment/manifests/alb-ingress-controller
```
cd ~/environment

mkdir -p manifests/alb-ingress-controller && cd manifests/alb-ingress-controller
```

#### 4.1.2 클러스터에 대한 IAM OIDC(OpenID Connect) identity Provider를 생성
```
eksctl utils associate-iam-oidc-provider --region ${AWS_REGION} --cluster eks-demo --approve
```

#### 4.1.3 AWS Load Balancer Controller에 부여할 IAM Policy 생성
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

#### 4.1.4 AWS Load Balancer Controller를 위한 ServiceAccount 생성
```
eksctl create iamserviceaccount \
    --cluster eks-demo \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```

#### 4.1.5 AWS Load Balancer controller를 클러스터에 추가
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.4.1/cert-manager.yaml
```

#### 4.1.6 Load balancer controller yaml 파일 다운로드
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/v2_2_1_full.yaml
```

#### 4.1.7 yaml 파일에서 클러스터의 cluster-name을 eks-demo으로 편집
```
spec:
    containers:
    - args:
        - --cluster-name=eks-demo # Insert EKS cluster that you created
        - --ingress-class=alb
        image: amazon/aws-alb-ingress-controller:v2.2.1
```

#### 4.1.8 yaml 파일에서 ServiceAccount yaml spec 삭제
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
```

#### 4.1.9 WS Load Balancer controller 파일 배포
```
kubectl apply -f v2_2_1_full.yaml
```

#### 4.1.10 배포가 성공적으로 되었는지 확인
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
