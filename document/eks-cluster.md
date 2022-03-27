# 3. EKS Cluster 생성

### eksctl로 클러스터 생성하기
#### eks-demo-cluster.yaml 파일 생성
```
cd ~/environment
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
    instanceType: m5.large # 클러스터 워커 노드의 인스턴스 타입
    desiredCapacity: 3 # 클러스터 워커 노드의 갯수
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

#### 클러스터를 배포
```
eksctl create cluster -f eks-demo-cluster.yaml
```

#### 노드가 제대로 배포되었는지 확인
```
kubectl get nodes 
```
