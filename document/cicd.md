# 7. CI/CD for EKS Cluster


- GitHub Actions, Kustomize, ArgoCD를 이용한다
- GitHub Actions은 스크립트 실행시간 한달 2000분까지 무료이니 고려하자.


### 7.0 github 토큰 생성
#### 7.0.1 접속
github 접속 settings - developer settings - personal access tokens - Generate new token
#### 7.0.2 
노트는 아무거나 적고, 만료없는걸로 선택하자 (만료있는게 사실 더 좋다)
repo클릭한다음에 Gen token 클릭
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

*myapp 을 빌드 하고 docker 이미지로 만든 다음 이를 ECR 에 push 하는 과정은 gitHub Action을 통해 이루어 집니다. 
*이 과정에서 사용할 IAM User를 생성 합니다.
*cloud9에서 아래 명령어 실행
```
aws iam create-user --user-name github-action
```

*ECR policy 생성
*주의사항 1) 아래 명령어 날리기 전에 myapp이라는 ecr을 생성한다.
*주의사항 2) ap-northeast-2를 자신이 사용하는 리전으로 바꾸어주자
```
cd ~/environment
```
```
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
            "Resource": "arn:aws:ecr:ap-northeast-2:${ACCOUNT_ID}:repository/myapp"
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


- 
-
-
-
-
-




### 이 아래부터는 예시니까 무시하자 


### 1.3 kubectl
#### 1.3.1 kubectl 설치
- 배포할 Amazon EKS 버전과 상응하는 kubectl를 설치
  https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
```
sudo curl -o /usr/local/bin/kubectl  \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
```
```
sudo chmod +x /usr/local/bin/kubectl
```

