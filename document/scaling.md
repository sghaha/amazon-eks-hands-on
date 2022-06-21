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

* 핵심은 replicas를 1로 하는거랑 마지막에 리소스 크기를 정해주는것
* 리전과 본인 ID등을 한번더 확인하세요

```
cd /home/ec2-user/environment/manifests
```
```
cat <<EOF> backend-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-nodejs-backend
  namespace: default
spec:
  replicas: 1
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
          image: $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/sample-nodejs-backend:latest
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
kubectl apply -f backend-deployment.yaml
```

#### 6.2.5 hpa yaml 생성
* name 같은거 본인에게 맞게 수정

```
cat <<EOF> backend-hpa.yaml
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: sample-nodejs-backend-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-nodejs-backend
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 30
EOF
```

#### 6.2.6 반영
```
kubectl apply -f backend-hpa.yaml
```


#### 6.2.7 확인
```
kubectl get hpa
```

* 결과 예시
```
NAME                        REFERENCE                          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
sample-nodejs-backend-hpa   Deployment/sample-nodejs-backend   0%/30%    1         3         1          31s
```



### 6.3 부하테스트
#### 6.3.1 모니터링
```
kubectl get hpa -w
```
* 혹은
```
kubectl get pod
```

#### 6.3.2 부하테스트 진행
* 마지막에 url은 본인에게 맞춰서 한다.
```
ab -c 200 -n 1000 -t 120 http://$(kubectl get ingress/backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/
```
* pod가 늘어났다가 줄어들 것이다.(위 명령어 설정값에 따라 최대 3개까지 늘어난다. c: 사용자수, n : 사용자가 보내는 요청의 수, t : 시간)

* 스케일아웃은 그래도 금방되는 편인데 스케일인은 10-15분 걸린다. 





### 6.4 Cluster Autoscaler
#### 6.4.1 설명
```
HPA때는 파드만 늘렸다
워커노드에 파드가 가득찼을때 사용하는것이 Cluster Autoscaler이다

pending 상태인 파드가 존재할 경우, 워커 노드를 스케일 아웃합니다.
```
* 혹시 클러스터의 상태를 시각화하고싶으면 https://codeberg.org/hjacobs/kube-ops-view 를 사용해 보자


#### 6.4.2 현재 워커의 오토스케일링 Max값 수정
```
Ec2 콘솔 - 오토스케일링 그룹 - eks 해당하는거 선택
편집 눌러서 Maximim capacity를 늘려주자
3으로 늘려주자
```


#### 6.4.3 yaml 다운
* Cluster Atuoscaler에서 제공하는 예제파일 다운로드
```
wget https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

#### 6.4.4 클러스터 이름 설정
vi cluster-autoscaler-autodiscover.yaml

163번째줄에 <YOUR CLUSTER NAME>을 나의 eks 클러스터 이름으로

#### 6.4.5 적용
```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

### 6.5 테스트
#### 6.5.1 모니터링 명령어
```
kubectl get nodes -w
```

#### 6.5.2 파드 늘리기
```
kubectl create deployment autoscaler-demo --image=nginx
```
```
kubectl scale deployment autoscaler-demo --replicas=12
```




#### 6.5.3 배포 진행 상태 파악하기 위해
```
kubectl get deployment autoscaler-demo --watch
```
* 워커노드가 3개까지 늘어남

#### 6.5.4 파드삭제
```
kubectl delete deployment autoscaler-demo
```




### 6.6 Kubernetes Operational View 
  
  현재 6.6은 헬름과 kube의 버전문제로 오류가 납니다. 건너 뛰세요   
  현재 6.6은 헬름과 kube의 버전문제로 오류가 납니다. 건너 뛰세요   
  현재 6.6은 헬름과 kube의 버전문제로 오류가 납니다. 건너 뛰세요   
  현재 6.6은 헬름과 kube의 버전문제로 오류가 납니다. 건너 뛰세요   
  
  
  
#### 6.6.1 설명
* 쿠버네티스 클러스터의 상태를 시각적으로 볼 수 있는 간단한 페이지
* 귀찮으면 안해도된다. 나중에 더 좋은 모니터링 툴 설치한다.

#### 6.6.2 helm cli 툴 설치
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

#### 6.6.3 확인
```
helm version --short
```

#### 6.6.4 stable 저장소 추가
```
helm repo add stable https://charts.helm.sh/stable
```


#### 6.6.5 차트 리스트들을 확인
```
helm search repo stable
```

#### 6.6.6 명령어를 위한 Bash completion을 구성

```
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```


#### 6.6.7 kube-ops-view 설치	
```
helm install kube-ops-view \
stable/kube-ops-view \
--set service.type=LoadBalancer \
--set rbac.create=True
```


#### 6.6.8 배포 확인	
```
helm list
```

#### 6.6.9 시각적으로 확인	
```
kubectl get svc kube-ops-view
에서 나온 external-ip를 복사해서 웹페이지 접속
```
