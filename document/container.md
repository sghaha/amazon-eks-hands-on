# 2. Container Image

### 2.1 Amazon ECR에 이미지 올리기
#### 2.1.1 Amazon ECR 리포지토리 생성 및 이미지 올리기
- 컨테이너라이징할 소스 코드를 다운
```
cd ~/environment/
```
```
git clone https://github.com/sghaha/sample-nodejs-backend.git
``` 
```
npm install
```


#### 2.1.2 이미지 리포지토리를 생성
```
aws ecr create-repository --repository-name sample-nodejs-backend --image-scanning-configuration scanOnPush=true --region ${AWS_REGION}
```

- 결과 예시
```
{
    "repository": {
        "repositoryUri": "876630244803.dkr.ecr.ap-northeast-1.amazonaws.com/sample-nodejs-backend", 
        "imageScanningConfiguration": {
            "scanOnPush": true
        }, 
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }, 
        "registryId": "876630244803", 
        "imageTagMutability": "MUTABLE", 
        "repositoryArn": "arn:aws:ecr:ap-northeast-1:876630244803:repository/sample-nodejs-backend", 
        "repositoryName": "sample-nodejs-backend", 
        "createdAt": 1649380735.0
    }
}
```

- "아마존 Amazon Elastic Kubernetes Service 콘손 > Amazon ECR > 프라이빗 리포지토리" 에 들어가면 현재 생성한 레포지토리를 확인 가능



#### 2.1.3 인증 토큰을 가져와서, 컨테이너 이미지를 리포지토리에 푸쉬

- "아마존 Amazon Elastic Kubernetes Service 콘손 > Amazon ECR > 프라이빗 리포지토리"에 생성된 레포지토리 클릭

- 푸시명령보기를 클릭하면 명령어가 나옵니다. 아래에 쓰여있는 명령어랑 비슷한데 다르다면 "푸시명령보기"에 나온 명령어를 우선시 합니다.(리전부분, 본인 ID부분 등이 조금씩 다를수있습니다.)

```
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com
```

- 결과 예시(워닝은... 무시하자)
```
WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

#### 2.1.4 도커 이미지를 빌드
```
cd ~/environment/sample-nodejs-backend
```
```
docker build -t sample-nodejs-backend .
```

#### 2.1.5 이미지 태그 지정
```
docker tag sample-nodejs-backend:latest $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/sample-nodejs-backend:latest
```

#### 2.1.6 이미지를 리포지토리에 푸쉬
```
docker push $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/sample-nodejs-backend:latest
```

- 여기까지 진행하면 ecr콘솔에 latest태그로 도커 이미지가 올라간걸 확인 가능








2.2 같은작업을 한번 더 진행

- 2.1에는 backend로 쓰일 이미지 작업을 한것이고 2.2챕터에서는 frontend로 쓰일 이미지를 작업합니다.
- 이 두 이미지는 실습이 끝날때까지 쓰일 예정입니다.


#### 2.2.1 Amazon ECR 리포지토리 생성 및 이미지 올리기
- 컨테이너라이징할 소스 코드를 다운
```
cd ~/environment/
```
```
git clone https://github.com/sghaha/sample-react-app.git
``` 

#### 2.2.2 이미지 리포지토리를 생성
```
aws ecr create-repository --repository-name sample-react-app --image-scanning-configuration scanOnPush=true --region ${AWS_REGION}
```

- "아마존 Amazon Elastic Kubernetes Service 콘손 > Amazon ECR > 프라이빗 리포지토리" 에 들어가면 현재 생성한 레포지토리를 확인 가능



#### 2.2.3 인증 토큰을 가져와서, 컨테이너 이미지를 리포지토리에 푸쉬

- "아마존 Amazon Elastic Kubernetes Service 콘손 > Amazon ECR > 프라이빗 리포지토리"에 생성된 레포지토리 클릭

- 푸시명령보기를 클릭하면 명령어가 나옵니다. 아래에 쓰여있는 명령어랑 비슷한데 다르다면 "푸시명령보기"에 나온 명령어를 우선시 합니다. (리전부분, 본인 ID부분 등이 조금씩 다를수있습니다.)

```
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com
```


#### 2.2.4 도커 이미지를 빌드
```
cd ~/environment/sample-react-app
```
```
npm install
```

```
npm run build
```

```
docker build -t sample-react-app .
```

#### 2.2.5 이미지 태그 지정
```
docker tag sample-react-app:latest $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/sample-react-app:latest
```

#### 2.2.6 이미지를 리포지토리에 푸쉬
```
docker push $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/sample-react-app:latest
```

- 여기까지 진행하면 ecr콘솔에 latest태그로 도커 이미지가 올라간걸 확인 가능

