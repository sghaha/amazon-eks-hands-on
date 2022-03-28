# 7. CI/CD for EKS Cluster


- GitHub Actions, Kustomize, ArgoCD를 이용한다
- GitHub Actions은 스크립트 실행시간 한달 2000분까지 무료이니 고려하자.


### 7.0 github 토큰 생성
#### 7.0.1 접속
github 접속 settings - developer settings - personal access tokens - Generate new token
#### 7.0.2 
노트는 아무거나 적고, 만료없는걸로 선택하자 (만료있는게 사실 더 좋다)
repo랑 workflow 클릭한다음에 Gen token 클릭
* 토큰이 보여질텐데 꼭 복사해놓자, 한번만 보인다.



### 7.1 git Repositories
실습을 위해 두 개의 github 레파지토리가 필요 합니다.
* myapp-repo: Frontend 소스가 위치한 레파지토리
* k8s-manifest-repo: K8S 관련 메니페스트가 위치한 레파지토리



#### 7.1.1 생성
* github에 myapp-repo, k8s-manifest-repo 레파지토리를 생성한다.
* 일단은 둘다 public하게 생성하자



### 7.2 샘플 ReactApp 및 myapp-repo 클론
* Cloud9에서
```
cd ~/environment/
git clone https://github.com/sghaha/sample-react-app.git
git clone https://github.com/{내깃헙아이디}/myapp-repo.git
```


### 7.3 myapp-repo에 push
#### 7.3.1 파일 복사
```
cp -r sample-react-app/* myapp-repo/
```
#### 7.3.2 push
* cloud9 왼쪽의 깃 아이콘을 클릭한후 커밋 & push하자
* 커밋은 + 버튼누르고 메시지 입력후 컨트롤+엔터 하면되고
* 푸시는 레포지토리 옆에 말풍선같은 아이콘 클릭후 push를 누른다음에 push하고싶은 repo를 클릭후 id/pw를 입력하면 된다



### 7.4 IAM 설정

#### 7.4.1 설명
* myapp 을 빌드 하고 docker 이미지로 만든 다음 이를 ECR 에 push 하는 과정은 gitHub Action을 통해 이루어 집니다. 

#### 7.4.2 IAM user 생성
* 이 과정에서 사용할 IAM User를 생성 합니다.
* cloud9에서 아래 명령어 실행
```
aws iam create-user --user-name github-action
```

* 결과예시
```
{
    "User": {
        "UserName": "github-action", 
        "Path": "/", 
        "CreateDate": "2022-01-28T04:08:37Z", 
        "UserId": "000000000", 
        "Arn": "arn:aws:iam::000000000:user/github-action"
    }
}
```

#### 7.4.3 ECR policy 생성
* 주의사항 1) 아래 명령어 날리기 전에 myapp-repo라는 ecr을 생성한다.
* 주의사항 2) ap-northeast-2를 자신이 사용하는 리전으로 바꾸어주자
```
cd ~/environment
```
```
cat <<EOF> ecr-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowPush",
            "Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "arn:aws:ecr:ap-northeast-2:${ACCOUNT_ID}:repository/myapp-repo"
        },
        {
            "Sid": "GetAuthorizationToken",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}
EOF
```



#### 7.4.4 IAM policy를 생성
* policy 이름으로 ecr-policy 를 사용 합니다. 
```
aws iam create-policy --policy-name ecr-policy --policy-document file://ecr-policy.json
```

* 결과예시
```
{
    "Policy": {
        "PolicyName": "ecr-policy", 
        "PermissionsBoundaryUsageCount": 0, 
        "CreateDate": "2022-01-28T04:10:57Z", 
        "AttachmentCount": 0, 
        "IsAttachable": true, 
        "PolicyId": "000000000000", 
        "DefaultVersionId": "v1", 
        "Path": "/", 
        "Arn": "arn:aws:iam::0000000:policy/ecr-policy", 
        "UpdateDate": "2022-01-28T04:10:57Z"
    }
}
```
* iam 콘솔 policy들어가면 ecr-policy가 생성된거 확인 가능
* iam user도 github-action이 생긴거 확인가능



#### 7.4.5 ECR policy를 IAM user에 부여
```
aws iam attach-user-policy --user-name github-action --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecr-policy
```



### 7.5 githup secrets

#### 7.5.1 설명
* githup secrets(AWS Credential, githup token) 생성
* github action 에서 사용할 AWS credential, github token을 생성하고, 설정 합니다.

#### 7.5.2 AWS Credential 생성
github action이 빌드된 myapp 을 docker image 로 만들어 ECR로 push 하는데
aws credential을 사용합니다.

그래서 gitbhub-action이라는 iam user 만든것이고
이 유저의 access key랑 secret key 만들것입니다.

```
aws iam create-access-key --user-name github-action
```

* 결과 에시
* 주의 1) "SecretAccessKey", "AccessKeyId"값을 따로 메모 저장
* 주의 2) 이 두 값이 노출되면 큰일남
```
{
    "AccessKey": {
        "UserName": "github-action", 
        "Status": "Active", 
        "CreateDate": "2022-01-28T04:22:01Z", 
        "SecretAccessKey": "0000000000000", 
        "AccessKeyId": "00000000000000"
    }
}
```

#### 7.5.2 github secret 설정

- front-app-repo 레파지토리로 돌아가 Settings > Secrets > actions
- New repository secret 클릭
- Name : ACTION_TOKEN
- value : personal access token 값
- Add secret 클릭



#### 7.5.3 깃헙에 aws 키 등록
- New repository secret 클릭
- Name : AWS_ACCESS_KEY_ID
- value : 00000000000<- 아까받은거
- Add secret 클릭


#### 7.5.4 깃헙에 aws 키 등록 2
- New repository secret 클릭
- Name : AWS_SECRET_ACCESS_KEY
- value : 000000000000   <- 아까받은거
- Add secret 클릭



### 7.6 github action 을 위한 build 스크립트
#### 7.6.1 .github 폴더 생성
* cloud9
```
cd ~/environment/myapp-repo
```
```
mkdir -p ./.github/workflows
```


#### 7.6.2 github action 이 사용할 build.yaml 생성
- myapp 을 checkout 하고, build 한 다음, 
- docker container 로 만들어 ECR 로 push 하는 과정을 담고 있는 github action build 스크립트를 작성 합니다.

```
cd ~/environment/myapp-repo/.github/workflows
```

- 주의) 계정명, 리전명, repo명 등을 확인하자
```
cat > build.yaml <<EOF


name: Build Front

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v2

      - name: Check Node v
        run: node -v

      - name: Build front
        run: |
          npm install
          npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Get image tag(verion)
        id: image
        run: |
          VERSION=$(echo ${{ github.sha }} | cut -c1-8)
          echo VERSION=$VERSION
          echo "::set-output name=version::$VERSION"

      - name: Build, tag, and push image to Amazon ECR
        id: image-info
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp-repo
          IMAGE_TAG: ${{ steps.image.outputs.version }}
        run: |
          echo "::set-output name=ecr_repository::$ECR_REPOSITORY"
          echo "::set-output name=image_tag::$IMAGE_TAG"
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

EOF
```

#### 7.6.3 깃헙에 Push
* 만들어진 build.yaml을 push하면 깃헙에서 action이 수행됩니다.





### 7.7 Kustomize Overview

#### 7.7.1 설명
- Kustomize 는 쿠버네티스 manifest 를 사용자 입맛에 맞도록 정의 하는데 도움을 주는 도구
- Kustomize 를 활용해 kuberenetes Deployment 리소스에 동일한 label, metadata 값 을 주도록 하며, 
- myapp의 새로운 변경 사항 발생에 따른 새로운 Image Tag를 Deployment 리소스에 적용할 겁니다.

#### 7.7.2 kubernetes manifest 디렉토리 구성

```
cd ~/environment
```
```
mkdir -p ./k8s-manifest-repo/base
```
```
mkdir -p ./k8s-manifest-repo/overlays/dev
```
```
cd ../k8s-manifest-repo/base
```

* 중간에 내 ecr주소로 바꾸어서 명령어 실행
```
cat <<EOF> myapp-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: {내ecr주소}.dkr.ecr.ap-northeast-2.amazonaws.com/myapp-repo:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
EOF
```

```
cat <<EOF> myapp-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  selector:
    app: myapp
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```


```
cat <<EOF> ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "myapp-ingress"
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: "myapp"
                port:
                  number: 80
EOF
```

```
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - myapp-deployment.yaml
  - myapp-service.yaml
EOF
```


```
cd ~/environment/k8s-manifest-repo/overlays/dev
```

```
cat <<EOF> myapp-service-patch.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
  labels:
    env: dev
spec:
  selector:
    app: myapp

EOF
```

```
cat <<EOF> myapp-deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
  labels:
    env: dev
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
EOF
```

* 중간에 내 ecr주소로 바꿀것

```
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
- name: {내 ecr주소}.dkr.ecr.ap-northeast-2.amazonaws.com/myapp-repo
  newName: {내 ecr주소}.dkr.ecr.ap-northeast-2.amazonaws.com/myapp-repo
  newTag: aa68ef2e
resources:
- ../../base
patchesStrategicMerge:
- myapp-deployment-patch.yaml
- myapp-service-patch.yaml
EOF
```



### 7.8 Kustomize를 위한 github
#### 7.8.1 k8s-manifest-repo 이름으로 깃헙 repo 생성
#### 7.8.2 소스 push
* 중간에 내 깃험 주소로 변경하자
```
cd ~/environment/k8s-manifest-repo/
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/sghaha/k8s-manifest-repo.git
git push -u origin main
```


### 7.9 ArgoCD
#### 7.9.1 ArgoCD 설치
```
kubectl create namespace argocd
```
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### 7.9.2 ArgoCD CLI 를 설치
```
cd ~/environment
```
```
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
```
```
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
```
```
sudo chmod +x /usr/local/bin/argocd
```

#### 7.9.3 ELB연동
* ArgoCD 서버는 기본적으로 퍼블릭 하게 노출되지 않습니다. 실습의 목적상 이를 변경하여 ELB 를 통해 접속 가능하도록 하겠습니다.
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
* 3~4분 기다리자

#### 7.9.4 ELB 주소 확인
```
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output .status.loadBalancer.ingress[0].hostname`
```
```
echo $ARGOCD_SERVER
```


#### 7.9.5 password 얻기
```
ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
```
```
echo $ARGO_PWD
```


#### 7.9.6 로그인해보기
* 앞서 얻은 alb 주소랑 id/pw를 이용하여 로그인 (id : admin)


#### 7.9.7 ArgoCD 설정 1

































