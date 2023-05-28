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

#### 1.3.2 kubectl 자동완성
```
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```




### 1.4 eksctl
#### 1.4.1 eksctl 설치(0.142.0)
https://github.com/weaveworks/eksctl/blob/main/README.md#installation

- eksctl 바이너리를 다운로드

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/v0.142.0/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
- 바이너리를 /usr/local/bin으로 이동
```
sudo mv -v /tmp/eksctl /usr/local/bin
```
- 설치 여부 확인
```
eksctl version
```

### 1.5 jq

#### 1.5.1 jq 설치
- json을 다루는 유틸리티
```
sudo yum install -y jq
```


### 1.6 Homebrew

#### 1.6.1 ec2-user 패스워드
* homebrew설치할때 패스워드 묻는다. 미리 설정
```
sudo passwd ec2-user
```

#### 1.6.2 homebrew 설치
https://brew.sh/

* 위 url타고 들어가면 아래랑 비슷한 명령어가 있다. 날리자
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

* 인스톨후 뭐라뭐라 설명나오는데 아래 두개 명령어 날리면 된다 (명령어가 달라져있을수도있으니까 무턱대로 아래꺼 복붙 하진 말자)
```
(echo; echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"') >> /home/ec2-user/.bash_profile
```
```
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

#### 1.6.3 설치 확인
```
brew -v
```


### 1.7 k9s

#### 1.7.1 k9s 설치
```
brew install derailed/k9s/k9s
```


### 1.8 helm
#### 1.8.1 helm 설치
```
brew install helm
```

### 1.9 Account ID, Region 설정

```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
```


### Bash profile 저장
```
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export AZS=(${AZS[@]})" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region

```

