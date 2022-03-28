# 4. Ingress Controller 생성

### AWS Load Balancer 컨트롤러 생성
#### manifests 이름의 폴더 생성
- /home/ec2-user/environment/manifests/alb-ingress-controller
```
cd ~/environment

mkdir -p manifests/alb-ingress-controller && cd manifests/alb-ingress-controller
```

#### 클러스터에 대한 IAM OIDC(OpenID Connect) identity Provider를 생성
```
eksctl utils associate-iam-oidc-provider --region ${AWS_REGION} --cluster eks-demo --approve
```

#### AWS Load Balancer Controller에 부여할 IAM Policy 생성
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

#### AWS Load Balancer Controller를 위한 ServiceAccount 생성
```
eksctl create iamserviceaccount \
    --cluster eks-demo \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```

#### AWS Load Balancer controller를 클러스터에 추가
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.4.1/cert-manager.yaml
```

#### Load balancer controller yaml 파일 다운로드
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/v2_2_1_full.yaml
```

#### yaml 파일에서 클러스터의 cluster-name을 eks-demo으로 편집
```
spec:
    containers:
    - args:
        - --cluster-name=eks-demo # Insert EKS cluster that you created
        - --ingress-class=alb
        image: amazon/aws-alb-ingress-controller:v2.2.1
```

#### yaml 파일에서 ServiceAccount yaml spec 삭제
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

#### AWS Load Balancer controller 파일 배포
```
kubectl apply -f v2_2_1_full.yaml
```

#### 배포가 성공적으로 되었는지 확인
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
