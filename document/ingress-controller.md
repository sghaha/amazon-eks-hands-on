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


#### 4.0.4 추가수행 (만약!!!!! assumed-role이 있다면 )
- 일반적인 경우에는 필요하지 않습니다. 다음단계로 넘어가셔도 됩니다.
```
assumedrolename=$(echo ${rolearn} | awk -F/ '{print $(NF-1)}')
```
```
rolearn=$(aws iam get-role --role-name ${assumedrolename} --query Role.Arn --output text) 
```

#### 4.0.5 identity 맵핑을 생성
```
eksctl create iamidentitymapping --cluster eks-demo --arn ${rolearn} --group system:masters --username admin
```

#### 4.0.6 확인
```
kubectl describe configmap -n kube-system aws-auth
```



### 4.1 AWS Load Balancer 컨트롤러 생성
#### 4.1.1 manifests 이름의 폴더 생성
- /home/ec2-user/environment/manifests/alb-ingress-controller
```
cd ~/environment

mkdir -p manifests/alb-ingress-controller && cd manifests/alb-ingress-controller
```

#### 4.1.2 클러스터에 대한 IAM OIDC(OpenID Connect) identity Provider를 생성
- 쿠버네티스가 직접 관리하는 사용자 계정을 의미하는 service account에 IAM role을 사용하기 위해 생성합니다.
```
eksctl utils associate-iam-oidc-provider --region ${AWS_REGION} --cluster eks-demo --approve
```


- 확인
```
aws eks describe-cluster --name eks-demo --query "cluster.identity.oidc.issuer" --output text
```

- 위 명령어에서 나오는 id 값 확인후 아래와 같이 날려보자
```
aws iam list-open-id-connect-providers | grep 7C9832F25C000000000000C3
```


- 결과예시
```
"Arn": "arn:aws:iam::876630244803:oidc-provider/oidc.eks.ap-northeast-1.amazonaws.com/id/7C9832F25C000000000000C3"
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
- 인증서 구성을 웹훅에 삽입할 수 있도록 cert-manager 를 설치합니다
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.4.1/cert-manager.yaml
```
- Cert-manager는 쿠버네티스 클러스터 내에서 TLS인증서를 자동으로 프로비저닝 및 관리하는 오픈 소스입니다.


#### 4.1.6 Load balancer controller yaml 파일 다운로드
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/v2_2_1_full.yaml
```

#### 4.1.7 yaml 파일에서 클러스터의 cluster-name을 eks-demo으로 편집

- 아래 vi 명령을 안하고 그냥 cloud9상에서 수정해도됩니다.
```
vi v2_2_1_full.yaml
```
- 796라인 근처
```
spec:
    containers:
    - args:
        - --cluster-name=eks-demo # Insert EKS cluster that you created
        - --ingress-class=alb
        image: amazon/aws-alb-ingress-controller:v2.2.1
```

#### 4.1.8 yaml 파일에서 ServiceAccount yaml spec 삭제
- 서비스 어카운트를 이미 생성했기 때문에 아래 내용 삭제
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


- 결과 예시 (Ready 부분에 1/1이라고 떠야합니다. 1분정도 걸릴수 있습니다.)

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           49s
```


#### 4.1.11 서비스 어카운트 확인
```
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```


#### 4.1.12 애드온 로그 확인

- Yaml 파일에서 애드온 네임스페이스를 kube-system으로 명시했기에
```
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
```

#### 4.1.13 속성값 파악
```
ALBPOD=$(kubectl get pod -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
```
```
kubectl describe pod -n kube-system ${ALBPOD}
```
