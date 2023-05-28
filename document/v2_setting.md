# 1. 환경 세팅
* 최초 작성 2023년 05월 28일 
### 1.1 AWS Cloud9 구성
#### 1.1.1 AWS Cloud9으로 IDE 구성
- Cloud9 console > Create environment
- Name 입력(ex : sghaha-cloud9)
- New ec2 instance
- Instance type : t3.small(t2.micro도 상관없음)
- platform : Amazon Linux 2
- Timeout : 30 mins

#### 1.1.2 IAM Role 생성
Administrator access 정책을 가진 IAM Role을 생성
- AWS 콘솔 - IAM - ROLE(역할) - Create Role
- aws service - ec2 선택 후 next
- AdministratorAccess 검색후 선택, next
- Role Name을 적어주자(ex: sghaha-role-c9-admin) 그리고 create

#### 1.1.3 AWS Cloud9 Instance에 IAM Role 부여
- EC2 instnace 콘솔 > Cloud9관련 ec2 클릭
- Actions(작업) > Security(보안) > Modify IAM Role(IAM 역할 수정)
- Change IAM role (1.1.2에서 생성한 role)

#### 1.1.4 IDE에서 IAM 설정 업데이트
- 설명 : AWS Cloud9 credentials 비활성화하고 IAM Role을 붙임(해당 credentials는 EKS IAM authentication과 호환되지 않음)
- Cloud9 IDE > 우측 상단 기어 아이콘 클릭 > AWS SETTINGS in the sidebar > Credentials > Disable the AWS managed temperature credits 


기존의 자격 증명 파일도 제거
```
rm -vf ${HOME}/.aws/credentials
```
- Cloud9 IDE가 올바른 IAM Role을 사용하고 있는지 확인
```
aws sts get-caller-identity

```
- 결과 예시
```
{
    "Account": "000000000000", 
    "UserId": "AROAYPGKIIIIIZGAYYYYY:i-0436079f403eeeeee", 
    "Arn": "arn:aws:sts::582392222222:assumed-role/sghaha-role-c9-admin/i-0436079f403eeeeee"
}
```

### 1.2 AWS CLI
* aws cli를 v2로 변경한다.
#### 1.2.1 버전 확인
```
aws --version
```
#### 1.2.2 AWS CLI Version 1 삭제
```
sudo rm /usr/bin/aws
```
```
sudo rm /usr/bin/aws_completer
```
```
sudo rm -rf /usr/local/aws-cli
```
#### 1.2.3 AWS CLI Version 2 설치
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
```
unzip awscliv2.zip
```
```
sudo ./aws/install
```
```
export PATH=/usr/local/bin:$PATH
```
```
source ~/.bash_profile
```
#### 1.2.4 버전 확인
```
aws --version
```

- 결과 예시
```
aws-cli/2.11.23 Python/3.11.3 Linux/4.14.314-237.533.amzn2.x86_64 exe/x86_64.amzn.2 prompt/off
```


### 1.3 kubectl
#### 1.3.1 kubectl 설치 (1.26)
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html   
위 url타고 가서 1.26 설치 부분 따라하자
```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.26.4/2023-05-11/bin/linux/amd64/kubectl
```
```
chmod +x ./kubectl
```
```
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
```
```
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
```

- kubectl 설치 확인
```
kubectl version --short --client
```
* 결과 예시
```
Client Version: v1.26.4-eks-0a21954
Kustomize Version: v4.5.7
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
