# 2. Container Image

### Amazon ECR에 이미지 올리기
#### Amazon ECR 리포지토리 생성 및 이미지 올리기
- 컨테이너라이징할 소스 코드를 다운
```
git clone https://github.com/joozero/amazon-eks-flask.git
``` 

#### 이미지 리포지토리를 생성
```
aws ecr create-repository --repository-name demo-flask-backend --image-scanning-configuration scanOnPush=true --region ${AWS_REGION}
```

#### 인증 토큰을 가져와서, 컨테이너 이미지를 리포지토리에 푸쉬
```
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com
```

#### 도커 이미지를 빌드
```
cd ~/environment/amazon-eks-flask

docker build -t demo-flask-backend .
```

#### 이미지 태그 지정
```
docker tag demo-flask-backend:latest $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/demo-flask-backend:latest
```

#### 이미지를 리포지토리에 푸쉬
```
docker push $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/demo-flask-backend:latest
```
