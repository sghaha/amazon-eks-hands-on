# 2. EKS Cluster 생성

### 2.1 VPC 생성
- VPC 콘솔에서
- VPC 생성
- VPC 등
- VPC 이름 : sghaha
- az수 : 2개로 하자
- 퍼블릭 서브넷 두개, 프라이빗 서브넷 두개
- nat게이트웨이는 1개만 하자


### 2.2 eksctl로 클러스터 생성하기
#### 2.2.1 eks-demo-cluster.yaml 파일 생성
* Cloud9에서 진행한다.
```
cd ~/environment
```
```
mkdir manifests
```
```
cd manifests
```


* 아래 <eks-demo>는 내가 만들고 싶은 eks의 클러스터 명을 적는다
* 아래 <Private-Subnet-id-1>과 <Private-Subnet-id-2>에는 만들엇진 프라이빗 서브넷 id를 넣는다.
* 만들어진 서브넷에 따라 ap-northeast-2b아닐수도있으니 이것도 주의
* ${AWS_REGION}는 안바꿔도 알아서 붙는걸로 아는데 이상하게 안되어서 , 그냥 ap-northeast-2로 명시해주니까 됐다
  
```
cat << EOF > eks-demo-cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: <eks-demo> # 생성할 EKS 클러스터명
  region: ${AWS_REGION} # 클러스터를 생성할 리전
  version: "1.26"

vpc:
  subnets:
    private:
      ap-northeast-2a: { id: <Private-Subnet-id-1> }
      ap-northeast-2b: { id: <Private-Subnet-id-2> }

managedNodeGroups:
  - name: node-group # 클러스터의 노드 그룹명
    instanceType: m5.large # 클러스터 워커 노드의 인스턴스 타입
    desiredCapacity: 2 # 클러스터 워커 노드의 갯수
    volumeSize: 20  # 클러스터 워커 노드의 EBS 용량 (단위: GiB)
    privateNetworking: true
    ssh:
      enableSsm: true
    iam:
      withAddonPolicies:
        imageBuilder: true # Amazon ECR에 대한 권한 추가
        albIngress: true  # albIngress에 대한 권한 추가
        cloudWatch: true # cloudWatch에 대한 권한 추가
        autoScaler: true # auto scaling에 대한 권한 추가
        ebs: true # EBS CSI Driver에 대한 권한 추가

cloudWatch:
  clusterLogging:
    enableTypes: ["*"]

iam:
  withOIDC: true
EOF

```
