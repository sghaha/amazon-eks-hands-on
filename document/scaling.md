# 6. Autoscaling Pod & Cluster

### 6.1 설명
```
k8s에는 두가지 오토스케일링 기능이 있다
- HPA(Horizontal Pod AutoScaler)
- Cluster Autoscaler

HPA는 CPU 사용량 또는 사용자 정의 메트릭을 관찰하여 파드 개수를 자동으로 스케일합니다. 
그러나 해당 파드가 올라가는 EKS 클러스터 자체 자원이 모자라게 되는 경우, Cluster Autoscaler를 고려해야 합니다.
```

### 6.2 HPA 적용하기
#### 6.2.1 metrics server를 생성
* Metrics Server는 쿠버네티스 클러스터 전체의 리소스 사용 데이터를 집계합니다
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

#### 6.2.2 metrics server 확인
```
kubectl get deployment metrics-server -n kube-system
```


#### 6.2.3 백엔드 yaml 수정
* 예제는 백엔드로 demo-flask-backend를 쓰고있는데 입맛에 맞게 수정하면된다.
* 핵심은 replicas를 1로 하는거랑 마지막에 리소스 크기를 정해주는것
```
cd /home/ec2-user/environment/manifests
```
```
cat <<EOF> flask-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-flask-backend
  namespace: default
spec:
  replicas: 1
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
          resources:
            requests:
              cpu: 250m
            limits:
              cpu: 500m
EOF
```


#### 6.2.4 반영
```
kubectl apply -f flask-deployment.yaml
```

#### 6.2.5 hpa yaml 생성
* name 같은거 본인에게 맞게 수정

```
cat <<EOF> flask-hpa.yaml
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: demo-flask-backend-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo-flask-backend
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 30
EOF
```

#### 6.2.6 반영
```
kubectl apply -f flask-hpa.yaml
```


#### 6.2.7 확인
```
kubectl get hpa
```

* 결과 예시
```
NAME                     REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
demo-flask-backend-hpa   Deployment/demo-flask-backend   4%/30%    1         5         1          36s
```




















