# 5. Deploy Microservices

### 5.1 서비스 배포하기(1st 백엔드)
* Amazon ECR에 이미지 올리기 실습 부분이 선행되어야 합니다.

#### 5.1.1 디렉토리 이동
```
cd ~/environment/manifests/
```


#### 5.1.2 deloy manifest 생성	
- 아래 ap-northeast-2부분을 자신의 리전에 맞게 변경해야합니다.

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
        - name: demo-flask-backend
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
kubectl apply -f flask-deployment.yaml
```
```
kubectl apply -f flask-service.yaml
```
```
kubectl apply -f ingress.yaml
```


#### 5.1.6 alb 주소 확인	
```
echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/contents/aws
```


### 5.2 서비스 배포하기(2nd 백엔드)

#### 5.2.1 디렉토리 이동
```
cd ~/environment/manifests/
```


### 5.2.2 deploy manifest를 생성
```
cat <<EOF> nodejs-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-nodejs-backend
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-nodejs-backend
  template:
    metadata:
      labels:
        app: demo-nodejs-backend
    spec:
      containers:
        - name: demo-nodejs-backend
          image: public.ecr.aws/y7c9e1d2/joozero-repo:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
EOF
```


#### 5.2.3 service manifest 파일 생성	
```
cat <<EOF> nodejs-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: demo-nodejs-backend
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/services/all"
spec:
  selector:
    app: demo-nodejs-backend
  type: NodePort
  ports:
    - port: 8080
      targetPort: 3000
      protocol: TCP
EOF
```

#### 5.2.4 ingress manifest 파일을 수정

* 생성이아니고 수정임 아까 만들었던 ingress.yaml 파일을 수정함
* 아래 명령어 날림


```
cat <<EOF> ingress.yaml
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
          - path: /contents
            pathType: Prefix
            backend:
              service:
                name: "demo-flask-backend"
                port:
                  number: 8080
          - path: /services
            pathType: Prefix
            backend:
              service:
                name: "demo-nodejs-backend"
                port:
                  number: 8080
EOF
```



#### 5.2.5 배포	
```
kubectl apply -f nodejs-deployment.yaml
```
```
kubectl apply -f nodejs-service.yaml
```
```
kubectl apply -f ingress.yaml
```


#### 5.2.6 url 확인	
```
echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/services/all
```





### 5.3 프론트 앤드 배포
#### 5.3.1 React 소스 다운	
```
cd /home/ec2-user/environment
```
```
git clone https://github.com/joozero/amazon-eks-frontend.git
```


#### 5.3.2 이미지 리포지토리를 생성	
```
aws ecr create-repository \
--repository-name demo-frontend \
--image-scanning-configuration scanOnPush=true \
--region ${AWS_REGION}
```

#### 5.3.3 일부 소스 코드 수정 1

* Cloud9에서 수정하자
* /home/ec2-user/environment/amazon-eks-frontend/src/App.js url 값을  수정

```
$ echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/contents/'${search}'
```

* 이것의 결과값으로 넣어주자


#### 5.3.4 일부 소스 코드 수정 2	

* /home/ec2-user/environment/amazon-eks-frontend/src/page/UpperPage.js 수정
```
$ echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/services/all"
```



#### 5.3.5 npm 인스톨&빌드	

```
cd /home/ec2-user/environment/amazon-eks-frontend
```
* 들어가서

```
npm install
```
```
npm run build
```

#### 5.3.6 ecr 리포지토리 생성 & 이미지 푸시
```
docker build -t demo-frontend .
```
```
docker tag demo-frontend:latest $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
```
```
docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
```


#### 5.3.7 혹시	

* denied: Your authorization token has expired. Reauthenticate and try again. 메세지를 받는다면 아래의 명령어를 실행한 후, 다시 위의 명령어를 수행합니다.

```
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```




#### 5.3.8 디렉토리 이동
```
cd /home/ec2-user/environment/manifests
```



#### 5.3.9 디플로이 yaml 생성	
```
cat <<EOF> frontend-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-frontend
  template:
    metadata:
      labels:
        app: demo-frontend
    spec:
      containers:
        - name: demo-frontend
          image: $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
EOF
```



#### 5.3.10 서비스 yaml 생성	

```
cat <<EOF> frontend-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: demo-frontend
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: "/"
spec:
  selector:
    app: demo-frontend
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```



#### 5.3.11 인그래스 yaml	
```
cat <<EOF> ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ""backend-ingress""
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /contents
            pathType: Prefix
            backend:
              service:
                name: "demo-flask-backend"
                port:
                  number: 8080
          - path: /services
            pathType: Prefix
            backend:  
              service:
                name: "demo-nodejs-backend"
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: "demo-frontend"
                port:
                  number: 80
EOF
```



#### 5.3.12 매니페스트 배포	
```
kubectl apply -f frontend-deployment.yaml
```
```
kubectl apply -f frontend-service.yaml
```
```
kubectl apply -f ingress.yaml
```



#### 5.3.13 url 확인
```
echo http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
```


* 아래와 같은 에러가 나올것입니다. node에서 주는 데이터와 react 에서 받는 데이터가 틀어진 느낌입니다. 화면이 보이진 않아도 잘 따라온것이니 일단 넘어갑시다

```
react-dom.production.min.js:209 
        
       TypeError: Cannot read properties of undefined (reading 'map')
    at k (UpperPage.js:50:25)
    at Qi (react-dom.production.min.js:153:146)
    at za (react-dom.production.min.js:175:309)
    at vl (react-dom.production.min.js:263:406)
    at su (react-dom.production.min.js:246:265)
    at lu (react-dom.production.min.js:246:194)
    at Zl (react-dom.production.min.js:239:172)
    at react-dom.production.min.js:123:115
    at t.unstable_runWithPriority (scheduler.production.min.js:19:467)
    at $o (react-dom.production.min.js:122:325)
el @ react-dom.production.min.js:209

Uncaught (in promise) TypeError: Cannot read properties of undefined (reading 'map')
    at k (UpperPage.js:50:25)
    at Qi (react-dom.production.min.js:153:146)
    at za (react-dom.production.min.js:175:309)
    at vl (react-dom.production.min.js:263:406)
    at su (react-dom.production.min.js:246:265)
    at lu (react-dom.production.min.js:246:194)
    at Zl (react-dom.production.min.js:239:172)
    at react-dom.production.min.js:123:115
    at t.unstable_runWithPriority (scheduler.production.min.js:19:467)
    at $o (react-dom.production.min.js:122:325)
```
    

