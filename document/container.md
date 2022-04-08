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
```
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com
```

#### 2.1.4 도커 이미지를 빌드
```
cd ~/environment/amazon-eks-flask

docker build -t demo-flask-backend .
```

#### 2.1.5 이미지 태그 지정
```
docker tag demo-flask-backend:latest $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/demo-flask-backend:latest
```

#### 2.1.6 이미지를 리포지토리에 푸쉬
```
docker push $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/demo-flask-backend:latest
```
