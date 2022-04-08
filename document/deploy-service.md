# 5. Deploy Microservices

### 5.1 서비스 배포하기(1st 백엔드)
* Amazon ECR에 이미지 올리기 실습 부분이 선행되어야 합니다.

#### 5.1.1 디렉토리 이동
```
cd ~/environment/manifests/
```


#### 5.1.2 deloy manifest 생성	
- 아래 ap-northeast-1부분을 자신의 리전에 맞게 변경해야합니다.

```
cat <<EOF> backend-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-nodejs-backend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-nodejs-backend
  template:
    metadata:
      labels:
        app: sample-nodejs-backend
    spec:
      containers:
        - name: sample-nodejs-backend
          image: $ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/sample-nodejs-backend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
EOF
```


#### 5.1.3 service manifest 생성	

```
cat <<EOF> backend-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: sample-nodejs-backend
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  selector:
    app: sample-nodejs-backend
  type: NodePort
  ports:
    - port: 8080 # 서비스가 생성할 포트  
      targetPort: 8080 # 서비스가 접근할 pod의 포트
      protocol: TCP
EOF
```


#### 5.1.4 ingress manifest 파일 생성	

```
cat <<EOF> backend-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: "backend-ingress"
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
                name: "sample-nodejs-backend"
                port:
                  number: 8080
EOF
```




#### 5.1.5 ALB 프로비저닝	
```
kubectl apply -f backend-deployment.yaml
```

- 확인
```
kubectl get pod
```

```
kubectl apply -f backend-service.yaml
```
- 확인
```
kubectl get svc
```


```
kubectl apply -f backend-ingress.yaml
```
- 확인
```
kubectl get ingress
```


#### 5.1.6 alb 주소 확인	
```
echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/dump/all
```

- 위 명령어를 날리면 url이 return 되는데, 약 3~5분뒤에 인터넷 브라우로 들어가봅니다. json이 리턴되면 성공









### 5.2 서비스 배포하기(2nd, 프론트엔드)
* Amazon ECR에 이미지 올리기 실습 부분이 선행되어야 합니다.

#### 5.2.1 backend url확인
```
echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
```

#### 5.2.2 소스 수정
```
cd ~/environment/sample-react-app
```

```
vi nginx.conf
```

- http://aaa.bbb.ccc를 위에서 확인한 backend url로 바꾸어줍니다.

#### 5.2.3 빌드

```
npm run build
```


#### 5.2.4 이미지 ecr에 푸시

- "아마존 Amazon Elastic Kubernetes Service 콘손 > Amazon ECR > 프라이빗 리포지토리"에 sample-react-app 레포지토리 클릭

- 푸시명령보기를 클릭하면 명령어가 나옵니다. 아래에 쓰여있는 명령어랑 비슷한데 다르다면 "푸시명령보기"에 나온 명령어를 우선시 합니다. (리전부분, 본인 ID부분 등이 조금씩 다를수있습니다.)

```
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com
```

```
docker build -t sample-react-app .
```

```
docker tag sample-react-app:latest $ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/sample-react-app:latest
```

```
docker push $ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/sample-react-app:latest
```

#### 5.2.5 방금 만든 도커 이미지 삭제
- 바로바로 안지워주면 나중에 귀찮아 진다.


```
docker images -a
```

- 결과 예시
```
REPOSITORY                                                           TAG          IMAGE ID       CREATED         SIZE
<none>                                                               <none>       85fb9a09b553   2 minutes ago   143MB
<none>                                                               <none>       e8c96d7d909e   2 minutes ago   143MB
876630244803.dkr.ecr.ap[중간생략].com/sample-react-app                 latest       6e84f53bb0ea   2 minutes ago   143MB
sample-react-app                                                     latest       6e84f53bb0ea   2 minutes ago   143MB
<none>                                                               <none>       991ab0515040   2 minutes ago   143MB
<none>                                                               <none>       e956badfd97e   2 minutes ago   142MB
<none>                                                               <none>       159cef29cf02   2 minutes ago   143MB
<none>                                                               <none>       0f79de9797c8   2 minutes ago   142MB
<none>                                                               <none>       f04b20055c8c   2 minutes ago   142MB
<none>                                                               <none>       d0fb434ea76d   2 minutes ago   142MB
node                                                                 12-alpine    8dc0ee810c0a   2 days ago      91MB
```

- 876630244803.dkr.ecr.ap[중간생략].com/sample-nodejs-backend 라고 써있는 것의 Image ID를 복사해서


```
docker rmi --force 6e84f53bb0ea
```




#### 5.2.6 디렉토리 이동
```
cd ~/environment/manifests/
```


#### 5.2.7 deloy manifest 생성	
- 아래 ap-northeast-1부분을 자신의 리전에 맞게 변경해야합니다.

```
cat <<EOF> frontend-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-react-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-react-app
  template:
    metadata:
      labels:
        app: sample-react-app
    spec:
      containers:
        - name: sample-react-app
          image: $ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/sample-react-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
EOF
```


#### 5.2.8 service manifest 생성	

```
cat <<EOF> frontend-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: sample-react-app
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  selector:
    app: sample-react-app
  type: NodePort
  ports:
    - port: 80 # 서비스가 생성할 포트  
      targetPort: 80 # 서비스가 접근할 pod의 포트
      protocol: TCP
EOF
```


#### 5.2.9 ingress manifest 파일 생성	

```
cat <<EOF> frontend-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: "frontend-ingress"
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
                name: "sample-react-app"
                port:
                  number: 80
EOF
```




#### 5.2.10 ALB 프로비저닝	
```
kubectl apply -f frontend-deployment.yaml
```

- 확인
```
kubectl get pod
```

```
kubectl apply -f frontend-service.yaml
```
- 확인
```
kubectl get svc
```


```
kubectl apply -f frontend-ingress.yaml
```
- 확인
```
kubectl get ingress
```


#### 5.2.11 alb 주소 확인	
```
echo http://$(kubectl get ingress/frontend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/
```

- 위 명령어를 날리면 url이 return 되는데, 약 3~5분뒤에 인터넷 브라우저로 들어가봅니다. 사이트가 뜨고 데이터가 보인다면 성공

