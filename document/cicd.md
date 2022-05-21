# 7. CI/CD for EKS Cluster

- 여기서부터는 본인의 깃헙 계정을 만들어서 사용합니다. 우선 깃헙계정을 만들어주세요
- GitHub Actions, Kustomize, ArgoCD를 이용한다
- GitHub Actions은 스크립트 실행시간 한달 2000분까지 무료이니 고려하자.
- 아마 이 실습을 진행하다보면 노드2개로는 파드 공간이 부족할것입니다. 하지만 노드 오토스케일링을 이전 장에서 진행했다면 자동으로 노드가 스케일 아웃됩니다.


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
* manifest-repo: K8S 관련 메니페스트가 위치한 레파지토리



#### 7.1.1 생성
* github에 myapp-repo, manifest-repo 레파지토리를 생성한다.
* 일단은 둘다 public하게 생성하자
* private으로 생성한다음에 추후 문제 생기는거 해결하는것도 좋은 경험일듯.


### 7.2 샘플 ReactApp 및 myapp-repo 클론
* Cloud9에서
```
cd ~/environment/
```

- 이 핸즈온을 따라했으면 아래 sample-react-app은 이미 받았을 것이다. 그러면 이 클론은하지 않아도 된다.
```
git clone https://github.com/sghaha/sample-react-app.git
```

- 내 repo를 클론한다.
```
git clone https://github.com/{내깃헙아이디}/myapp-repo.git
```




### 7.3 myapp-repo에 push
#### 7.3.1 파일 복사
- 혹시 숨김파일은 안보이게 설정되있을 수 있으니, cloud9파일 리스트 바로 위에 설정 클릭하고 show hidden files클릭하자

#### sample-react-app 디렉토리에 있는 파일을 .git / node_module빼고 전부 myapp-repo디렉토리에 복사하자, 그후 아래 명령어 수행

```
cd ~/environment/myapp-repo
```
```
npm install
```
```
npm run build
```





#### 7.3.2 내 깃허브에 push 전 확인할 사항
* 1) 푸시할 전체 파일은 약 25개입니다. 너무 많으면 잘못하고 계신겁니다.
* 2) .dockeringnore .gitingnore파일이 포함되어있는지 확인합시다. 없다면 아마 파일이 전부 복사 된것이 아닙니다.
* 3) nginx.conf의 proxy_pass가 나의 backend alb주소를 가리키고 있는지 확인합시다.

#### 7.3.3 내 깃허브에 push
* cloud9 왼쪽의 깃 아이콘(sourcecontrol)을 클릭
* myapp-repo에 change로 약 25개 파일이 뜰겁니다.
* 커밋을 합니다 ::: Changes에서 + 버튼누르고 메시지 입력후 컨트롤+엔터
* 푸시는 레포지토리 옆에 말풍선같은 아이콘 클릭후 push를 누른다음에 push하고싶은 repo를 클릭후 id/pw를 입력하면 된다 (id는 나의 깃헙 아이디, pw는 토큰)

* 제대로 되면 나의 깃헙 레포지토리에 푸시가 된것을 확인할 수 있다.



### 7.4 IAM 설정

#### 7.4.1 설명
* myapp 을 빌드 하고 docker 이미지로 만든 다음 이를 ECR 에 push 하는 과정은 gitHub Action을 통해 이루어 집니다. 

#### 7.4.2 IAM user 생성
* 이 과정에서 사용할 IAM User를 생성 합니다.
* cloud9에서 아래 명령어 실행
```
aws iam create-user --user-name github-action-{내아이디}
```

* 결과예시
```
{
    "User": {
        "UserName": "github-action-sghaha", 
        "Path": "/", 
        "CreateDate": "2022-04-11T12:05:03Z", 
        "UserId": "AID----------EWTUOTG", 
        "Arn": "arn:aws:iam::876630244803:user/github-action-sghaha"
    }
}
```

#### 7.4.3 ECR policy 생성
* 주의사항 1) 아래 명령어 날리기 전에 myapp-repo라는 ecr을 생성한다. ecr콘솔가서 생성, 프라이빗, 푸시할떄 스캔
* 주의사항 2) ap-northeast-1를 자신이 사용하는 리전으로 바꾸어주자
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
            "Resource": "arn:aws:ecr:ap-northeast-1:${ACCOUNT_ID}:repository/myapp-repo"
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
* {내아이디} 부분 수정에 유의
```
aws iam create-policy --policy-name ecr-policy-{내아이디} --policy-document file://ecr-policy.json
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
- {내아이디}부분 두곳을 잘 고쳐놓고 명령문을 날리자 {ACCOUNT_ID}는 냅두자. 

```
aws iam attach-user-policy --user-name github-action-{내아이디} --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecr-policy-{내아이디}
```

- IAM콘솔 - 사용자 로 들어가서 만든 ID를 클릭하면 policy가 연동되어있는걸 볼수 있다.




### 7.5 githup secrets

#### 7.5.1 설명
* githup secrets(AWS Credential, githup token) 생성
* github action 에서 사용할 AWS credential, github token을 생성하고, 설정 합니다.

#### 7.5.2 AWS Credential 생성
github action이 빌드된 myapp 을 docker image 로 만들어 ECR로 push 하는데
aws credential을 사용합니다.

그래서 gitbhub-action-{내아이디}이라는 iam user 만든것이고
이 유저의 access key랑 secret key 만들것입니다.

```
aws iam create-access-key --user-name github-action-{내아이디}
```

aws iam create-access-key --user-name github-action-sghaha

* 결과 에시
* 주의 1) "SecretAccessKey", "AccessKeyId"값을 따로 메모 저장, 꼭 저장
* 주의 2) 이 두 값이 노출되면 큰일남, 엄청 큰일남
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

- myapp-repo 레파지토리로 돌아가 Settings > Secrets > actions
- New repository secret 클릭
- Name : ACTION_TOKEN
- value : personal access token 값 (처음에 저장해놓은 github 토큰값)
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
- myapp 을 checkout 하고, build 한 다음, docker container 로 만들어 ECR 로 push 하는 과정을 담고 있는 github action build 스크립트를 작성 합니다.

```
cd ~/environment/myapp-repo/.github/workflows
```

- build.yaml을 다운받자
```
wget -O build.yaml https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/build.yaml?raw=true
```
- 위 명령어가 잘 안먹으면 https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/build.yaml 이걸 브라우저에 치고 나온결과물을 복사 해서 build.yaml파일로 만들자

- 27라인 : aws-region: ap-northeast-2 을 나의 리전에 맞추어주고
- 56라인 : repository: sghaha/manifest-repo을 repository: {내깃헙아이디}/manifest-repo로 바꾸어 주고
- 59라인 : path: k8s-manifest-repo이것도 path: manifest-repo이걸로 바꾸어주고
- 72라인 : git config --global user.email "{내메일주소}"
- 73라인 : git config --global user.name "{내깃헙아이디}"





#### 7.6.3 깃헙에 Push
* 만들어진 build.yaml을 push하면 깃헙에서 action이 수행됩니다.
* 깃헙 해당 repository- action클릭 해서 수행되는 걸 확인합니다.
* 아직까지는 ECR에 이미지 올리는것까지만 성공하고 fail이 날겁니다.




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
git clone https://github.com/sghaha/manifest-repo.git
```



```
mkdir -p ./manifest-repo/base
```
```
mkdir -p ./manifest-repo/overlays/dev
```
```
cd manifest-repo/base
```

- 중간에 리전을 내 리전에 맞게 변경
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
          image: ${ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/myapp-repo:latest
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
cd ~/environment/manifest-repo/overlays/dev
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

* 중간에 내 리전으로 바꿀것

```
cat <<EOF> kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
- name: ${ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/myapp-repo
  newName: ${ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/myapp-repo
  newTag: aa68ef2e
resources:
- ../../base
patchesStrategicMerge:
- myapp-deployment-patch.yaml
- myapp-service-patch.yaml
EOF
```

#### 7.7.3 푸시
해당 작업된것들을 깃헙에 푸시한다.

#### 7.7.4 github Action 재가동
* 깃헙 myapp-repo 들어가서 action클릭, 아까 실패한 job클릭, re-run all jobs. 클릭
* 여기까지 잘 따라왔다면 이제 job이 성공할것이다.

* 이제 push를 하면 myapp-repo의 변경점을 포함한 형상을 도커 이미지로 만들고, 그걸 나의 ecr에 푸시하고, kustomization에 해당 ecr 이미지의 태그를 넣어준다.




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

#### 7.9.7 github id/pw세팅
* argocd 페이지 왼쪽 기어모양 클릭
* repositrories 클릭
* Connect Repo using https 클릭
* url이랑 id/pw쓰고 커넥트
```
url은 manifest-repo의 url (ex: https://github.com/sghaha/manifest-repo)
id는 내 깃헙 아이디
pw는 토큰
```

#### 7.9.8 ArgoCD 설정 1
* 좌측 겹쳐있는 아이콘이 어플리케이션 설정 메뉴. 클릭하자
* Create App
* Application Name : myapp-cd-pipeline
* Project : default


#### 7.9.9 ArgoCD 설정 2
* 아래로 내리면 Source 섹션
* repo url : git repo주소 (https://github.com/{내 깃헙 주소}/manifest-repo.git)
* Revision : main
* Path :  overlays/dev


#### 7.9.10 ArgoCD 설정 3
* 아래로 내리면 Destination 섹션
* Cluster URL : https://kubernetes.default.svc
* Namespace : default
* 이렇게 하고 Create 클릭함


ArgoCD 화면으로 돌아가 배포 상태를 확인 합니다. 
Applications > myapp-cd-pipeline 으로 이동 하여 확인 해보면 
CURRENT SYNC STATUS의 값이 Out of Synced 입니다.

git repository 가 변경되면 자동으로 sync 작업이 수행 하도록 하려면 Auto-Sync 를 활성화 해야 합니다. 
이를 위해 APP DETAILS 로 이동 하여 ENABLE AUTO-SYNC 버튼을 눌러 활성화 합니다.


#### 7.9.11 반영됐는지 확인

파드들이 뜰것입니다.

clou9으로 돌아가 아래 명령어를 날립니다.
```
kubectl get pod
```

* 결과예시) 노드가 2개밖에 없어서 처음엔 파드가 팬딩 상태입니다
```
NAME                                    READY   STATUS    RESTARTS   AGE
myapp-57fc49c469-5h5cb                  0/1     Pending   0          90s
myapp-57fc49c469-x8l7d                  0/1     Pending   0          90s
sample-nodejs-backend-578855f99-zkdss   1/1     Running   0          3d7h
sample-react-app-54779f7fc7-cpcrp       1/1     Running   0          3d6h
sample-react-app-54779f7fc7-qv4s5       1/1     Running   0          3d6h
```

* 하지만 시간이 지나면 노드가 하나 더 자동으로 스케일 아웃되고 아래와 같이 파드가 러닝상태로 바뀝니다.
```
NAME                                    READY   STATUS    RESTARTS   AGE
myapp-57fc49c469-5h5cb                  1/1     Running   0          2m47s
myapp-57fc49c469-x8l7d                  1/1     Running   0          2m47s
sample-nodejs-backend-578855f99-zkdss   1/1     Running   0          3d7h
sample-react-app-54779f7fc7-cpcrp       1/1     Running   0          3d6h
sample-react-app-54779f7fc7-qv4s5       1/1     Running   0          3d6h
```



#### 7.9.12 myapp ingress 설정

```
cd /home/ec2-user/environment/manifests
```

```
cat <<EOF> myapp-ingress.yaml
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
kubectl apply -f myapp-ingress.yaml
```


* 확인 (약 5분후)
```
kubectl get ingress
```

```
NAME               CLASS    HOSTS   ADDRESS                                                                       PORTS   AGE
backend-ingress    <none>   *       k8s-default-backendi-6566bc7d31-1782616189.ap-northeast-1.elb.amazonaws.com   80      3d7h
frontend-ingress   <none>   *       k8s-default-frontend-3a6fb0b00c-1248002473.ap-northeast-1.elb.amazonaws.com   80      3d7h
myapp-ingress      <none>   *       k8s-default-myapping-78a29f8d18-1418731677.ap-northeast-1.elb.amazonaws.com   80      38s
```

* myapp-ingress의 alb주소를 브라우저에 입력해서 접속



###7.10 배포해보기

* /myapp-repo/src/layout/Layout.js의 41번째줄, "ExamApp"을 다른문구로 바꾸어 봅시다

* 커밋&푸시

* 깃허브 액션이 잘돌아가나 확인

* ArgoCD가 잘돌아가나확인

* 배포 완료후 alb주소로 문구 바뀐것 확인

* 배포도중에 잠깐 bad gateway가 뜨는것은 롤링 업데이트 설정으로 극복가능하지만 이 실습에선 다루지 않겠습니다.



### 7.11 정리
축하드립니다. 여러분은 코드 커밋만 하면 자동으로 배포가 되는 과정을 겪었습니다.



### 7.12 sample-react-app 삭제

* myapp을 띄웠으니 이제 sample-react-app은 필요하지 않습니다.

```
cd /home/ec2-user/environment/manifests
```

```
kubectl delete -f frontend-ingress.yaml 
```

```
kubectl delete -f frontend-service.yaml 
```
```
kubectl delete -f frontend-deployment.yaml 
```

```
kubctl get pod
```



* sample-react-app이 삭제된걸 볼수 있습니다.
```
NAME                                    READY   STATUS    RESTARTS   AGE
myapp-595d8d967f-ljdk6                  1/1     Running   0          6m18s
myapp-595d8d967f-m4wmd                  1/1     Running   0          6m21s
sample-nodejs-backend-578855f99-zkdss   1/1     Running   0          3d7h
```
