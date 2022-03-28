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
