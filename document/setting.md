# 1. Setting workspace 

### 1.1 AWS Cloud9 구성
#### 1.1.1 AWS Cloud9으로 IDE 구성
- Cloud9 console > Create environment
- Name 입력 > next step
- Instance type : t3.small(t2.micro도 상관없음), platform : Amazon Linux 2 > next step > Create environment

#### 1.1.2 IAM Role 생성
Administrator access 정책을 가진 IAM Role을 생성
- AWS 콘솔 - IAM - ROLE(역할) - Create Role
- aws service - ec2 선택 후 next
- AdministratorAccess 검색후 선택, next
- Role Name에 "HandsOn-Admin-본인아이디"를 적어주자 그리고 create

#### 1.1.3 AWS Cloud9 Instance에 IAM Role 부여
- EC2 instnace console > Select AWS Cloud9 instance
- Actions(작업) > Security(보안) > Modify IAM Role(IAM 역할 수정)
- Change IAM role (1.1.2에서 생성한 role)

#### 1.1.4 IDE에서 IAM 설정 업데이트
- AWS Cloud9 credentials 비활성화하고 IAM Role을 붙임(해당 credentials는 EKS IAM authentication과 호환되지 않음)
- Cloud9 IDE > 우측 상단 기어 아이콘 클릭 > AWS SETTINGS in the sidebar > Credentials > Disable the AWS managed temperature credits 
- 기존의 자격 증명 파일도 제거
```
rm -vf ${HOME}/.aws/credentials
```
- Cloud9 IDE가 올바른 IAM Role을 사용하고 있는지 확인
```
aws sts get-caller-identity --query Arn | grep HandsOn-Admin-본인아이디

```
- 결과 예시
```
arn:aws:sts::876630244803:assumed-role/HandsOn-Admin-sghaha/i-03e9af70279e891de
```

### 1.2 AWS CLI
#### 1.2.1 AWS CLI 업데이트
```
sudo pip install --upgrade awscli
```
#### 1.2.2 버전 확인
```
aws --version
```

### 1.3 kubectl
#### 1.3.1 kubectl 설치
- 배포할 Amazon EKS 버전과 상응하는 kubectl를 설치
  https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
```
sudo curl -o /usr/local/bin/kubectl  \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
```
```
sudo chmod +x /usr/local/bin/kubectl
```

- kubectl 설치 확인
```
kubectl version --client=true --short=true
```

### 1.4 etc
#### 1.4.1 jq 설치
- json을 다루는 유틸리티
```
sudo yum install -y jq
```
#### 1.4.2 bash-completion 설치
- 쉘에 completion script를 소싱하면 kubectl 명령어의 자동 완성을 가능하게 만들 수 있습니다. 
하지만 이런 completion script는 bash-completion에 의존하기 때문에 아래의 명령어를 통해, bash-completion 을 설치해야 합니다.
```
sudo yum install -y bash-completion
```

### 1.5 eksctl 설치
#### 1.5.1 eksctl 설치
- 최신의 eksctl 바이너리를 다운로드
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
- 바이너리를 /usr/local/bin으로 이동
```
sudo mv -v /tmp/eksctl /usr/local/bin
```
- 설치 여부 확인
```
eksctl version
```

### 1.6 AWS Cloud9 추가 설정
#### 1.6.1 현재 실행 Region을 기본값으로 설정
```
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
    
aws configure set default.region ${AWS_REGION}
```


- 리전 세팅 확인
```
aws configure get default.region
```
#### 1.6.2 현재 계정 ID을 등록
```
export ACCOUNT_ID=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.accountId')

echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
```
