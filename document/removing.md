# 10. 실습 후 리소스 정리

- 매일 퇴근전 실습한 리소스를 삭제하여 aws비용을 $0로 만들것입니다.


### 10.1 퇴근후 안껏을 때 추가되는 예상비용

- 12시간이라고 가정했을때
```
eks 클러스터 비용 : $1.2
NAT Gateway 비용 : $0.7
EC2 비용(t2.small 두개) : $0.7
로드밸런서 비용(3개) :$1.5

----
하루 : $4.1
한달 : $123
세금 : $12.3
총 추가 비용 : $135.3 (약 16만원)

* 로드 밸런서는 사실 3개 이상을 쓰는 경우가 훨씬 많지만 3개로 계산
* aws의 비용을 계산해보는것도 좋은 경험이 됩니다.
```


### 10.2 인그레스, 서비스, 파드 삭제

- eks클러스터를 지우면 같이 지워지긴하지만 명시적으로 미리 지워줍니다.

```
cd ~/environment/manifests
```

- eks-demo-cluster.yaml와 v2_2_1_full.yaml를 제외하고

```
kubectl delete -f 파일명
```

- aws-load-balancer-controller삭제
```
kubectl delete -f alb-ingress-controller/v2_2_1_full.yaml
```

### 10.3 eks클러스터 제거

- eks-demo제거 (eks-demo는 클러스터명입니다.)

```
eksctl delete cluster --name=eks-demo
```


### 10.4 ecr 레포지토리 제거

#### 10.4.1 ecr repo 리스트 불러오기

```
aws ecr describe-repositories
```

#### 10.4.2 제거
- 아래 3개는 예시입니다. 본인의 리턴값에 맞추어 지우면 됩니다.
```
aws ecr delete-repository --repository-name sample-nodejs-backend
```
```
aws ecr delete-repository --repository-name sample-react-app
```
```
aws ecr delete-repository --repository-name myapp-repo
```


### 10.4 로그 삭제
```
aws logs describe-log-groups --query 'logGroups[*].logGroupName' --output table | \
awk '{print $2}' | grep ^/aws/containerinsights/eks-demo | while read x; do  echo "deleting $x" ; aws logs delete-log-group --log-group-name $x; done
```

```
aws logs describe-log-groups --query 'logGroups[*].logGroupName' --output table | \
awk '{print $2}' | grep ^/aws/eks/eks-demo | while read x; do  echo "deleting $x" ; aws logs delete-log-group --log-group-name $x; done
```


### 10.5 Cloud9삭제
Cloud9 콘솔에 들어가 삭제


### 10.6 비고
```
위의 방법대로 하다보면 간간히 꼭 꼬일때가 있습니다. 
그럴땐 어쩔수 없이 하나하나 찾아서 지워줘야 합니다.
심지어 제대로 지워졌는지 확인해보기 위해서라도 하나한 서비스들이 잘 삭제되었는지 확인해보아야합니다.
확인해봐야 할 서비스로는
alb, cloudwatch, cloudformation등등이 있으니 하나씩 찾아보면서 공부하는것도 좋을듯 싶습니다.

* 정확히 어떤 서비스들을 확인해봐야하는지는 추후 업데이트 할 예정입니다.
```


