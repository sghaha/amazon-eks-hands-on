# 5. Deploy Microservices

* 주의) 사실 가이드 대로 demo-flask-backend와 demo-nodejs-backend, demo-frontend를 배포하면 마지막에 세개가 연동이 되어야 하는데 잘 안됩니다. 일단 따라하면서 대강의 절차를 파악하고 마지막 결과는 잘 안나와도 무시하시길...

### 5.1 서비스 배포하기(1st 백엔드)
* Amazon ECR에 이미지 올리기 실습 부분이 선행되어야 합니다. (demo-flask-backend)

#### 5.1.1 디렉토리 이동
```
cd ~/environment/manifests/
```


#### 5.1.2 deloy manifest 생성	
```
cat <<EOF> flask-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-flask-backend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-flask-backend
  template:
    metadata:
      labels:
        app: demo-flask-backend
    spec:
      containers:
        - name: demo-flask-backend
          image: $ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/demo-flask-backend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
EOF
```

