# 3. EKS Cluster 생성

### 3.1 eksctl로 클러스터 생성하기
#### 3.1.1 eks-demo-cluster.yaml 파일 생성
```
cd ~/environment
```
```
mkdir manifests
```
```
cd manifests
```


```
cat << EOF > eks-demo-cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-demo # 생성할 EKS 클러스터명
  region: ${AWS_REGION} # 클러스터를 생성할 리전
  version: "1.21"

vpc:
  cidr: "192.168.0.0/16" # 클러스터에서 사용할 VPC의 CIDR

managedNodeGroups:
  - name: node-group # 클러스터의 노드 그룹명
    instanceType: t3.small # 클러스터 워커 노드의 인스턴스 타입
    desiredCapacity: 2 # 클러스터 워커 노드의 갯수
    volumeSize: 10  # 클러스터 워커 노드의 EBS 용량 (단위: GiB)
    ssh:
      enableSsm: true
    iam:
      withAddonPolicies:
        imageBuilder: true # Amazon ECR에 대한 권한 추가
        # albIngress: true  # albIngress에 대한 권한 추가
        cloudWatch: true # cloudWatch에 대한 권한 추가
        autoScaler: true # auto scaling에 대한 권한 추가

cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
EOF
```

#### 3.1.2 클러스터를 배포
```
eksctl create cluster -f eks-demo-cluster.yaml
```
- 약 20분 걸림

#### 3.1.3 노드가 제대로 배포되었는지 확인
```
kubectl get nodes 
```

- 결과예시
```
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-18-99.ap-northeast-1.compute.internal    Ready    <none>   90s   v1.21.5-eks-9017834
ip-192-168-70-110.ap-northeast-1.compute.internal   Ready    <none>   85s   v1.21.5-eks-9017834
```


#### 3.1.4 자격증명 더해진것 확인
```
cat ~/.kube/config
```

#### 3.1.5 cloudwatch put disable
- 실습하는 동안 단한번도 보지 않을 eks 코어단 로그가 cloudwatch에 쌓인다. 괜히 하루에 1달러 이상나가기 때문에 꺼준다.
- eks 콘솔 > eks-demo 클릭 > 구성 > 로깅 관리 > 모두 끄고 > 변경사항 저장
