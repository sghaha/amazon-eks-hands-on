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

- 위 명령어를 날리면 url이 return 되는데, 약 3~5분뒤에 인터넷 브라우져로 들어가봅니다. json이 리턴되면 성공









### 5.2 서비스 배포하기(2nd, 프론트엔드)
* Amazon ECR에 이미지 올리기 실습 부분이 선행되어야 합니다.

#### 5.2.1 디렉토리 이동
```
cd ~/environment/manifests/
```


#### 5.2.2 deloy manifest 생성	
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


#### 5.2.3 service manifest 생성	

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


#### 5.2.4 ingress manifest 파일 생성	

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




#### 5.2.5 ALB 프로비저닝	
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


#### 5.2.6 alb 주소 확인	
```
echo http://$(kubectl get ingress/frontend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/
```

- 위 명령어를 날리면 url이 return 되는데, 약 3~5분뒤에 인터넷 브라우져로 들어가봅니다. 앱이 뜨면 성공

