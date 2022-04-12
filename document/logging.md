# 9. Logging (EFK)
- 참고 : https://hanjustudy.tistory.com/35
- 지금까지 t2.small로 실습을 진행했는데, 이번실습을 진행하다보면 메모리 부족현상이 생겨서 노드 크기를 늘린 노드그룹으로 옮길것입니다.


### 9.0 노드 그룹 옮기기

#### 9.0.1 클러스터 yaml 수정
- eks-demo-cluster.yaml의 일부를 아래와 같이 수정합니다. 
- 노드그룹의 이름을 바꾸고, t3.large타입으로 바꾸고, 노드는 2개 띄울것입니다.
```
  - name: node-group-v2
    instanceType: t3.large
    desiredCapacity: 2
```

```
eksctl create nodegroup --config-file eks-demo-cluster.yaml
```

- 20분 정도 기다리면 노드가 하나 더 뜹니다.
- eks콘솔 > 클러스터 클릭 > 구성 > 컴퓨팅 클릭하면 노드가 더 뜬걸 볼수 있음


#### 9.0.2 기존 노드 죽이고 파드를 새 노드로 옮기기

- 사실 이방식 대로 하면 무중단 운영이 안됩니다. (몇분 장애가 납니다.) 그러나 실습이기 때문에 그냥 진행합니다.

- ec2 콘솔 > 오토스케일링 그룹 > 기존노드의 as그룹 선택 > 편집 > 원하는 용량 : 0, 최소용량 :0, 최대 용량 0

- 몇분뒤 모든 파드들이 새 노드로 옮겨집니다.


#### 9.0.3 기존 노드 삭제
- eks콘솔 > 클러스터 클릭 > 구성 > 컴퓨팅 > 기존노드 클릭 > 삭제 
    
    
    

### 9.1 elasticsearch



#### 9.1.1 yaml 다운로드
```
cd ~/environment/manifests
```

```
wget -O elasticsearch.yaml https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/elasticsearch.yaml?raw=true
```

- 위 명령어가 404나던가 하면 그냥 https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/elasticsearch.yaml을 브라우저에서 쳐서 복사하자



#### 9.1.2 elasticsearch 설치


```
kubectl apply -f elasticsearch.yaml
```



#### 9.1.3 확인	
```
kubectl patch svc elasticsearch-svc -p '{"spec": {"type": "LoadBalancer"}}'
```
* 몇분뒤에

```
kubectl get svc
```
에서 나온 alb주소로 9200포트로 들어간다


* 결과 예시
```
{
name: "7en---RO",
cluster_name: "docker-cluster",
cluster_uuid: "kgIY4TXQS-----yZWUA",
version: {
number: "6.4.0",
build_flavor: "default",
build_type: "tar",
build_hash: "59----e",
build_date: "2018-08-17T23:18:47.308994Z",
build_snapshot: false,
lucene_version: "7.4.0",
minimum_wire_compatibility_version: "5.6.0",
minimum_index_compatibility_version: "5.0.0"
},
tagline: "You Know, for Search"
}
```



### 9.2 kibana

#### 9.2.1 yaml파일 다운로드
```
wget -O kibana.yaml https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/kibana.yaml?raw=true
```
- 404 에러가 난다거나 하면 그냥 인터넷 브라우저에서 본다음에 복붙하자

- 그리고 http://elasticsearch-svc.default.svc.cluster.local:9200 이부분을 나의 elasticsearch 주소로 바꾸어주자



#### 9.2.2 kibana 설치
```
kubectl apply -f kibana.yaml
```

#### 9.2.3 alb연결
```
kubectl patch svc kibana-svc -p '{"spec": {"type": "LoadBalancer"}}'
```

#### 9.2.4 확인	
```
kubectl get svc
```
에서 나온 alb주소로 5601포트로 들어가면(좀 기다려야한다) 콘솔 나온다




### 9.3 fluent-bit 설치

#### 9.3.1 yaml파일 다운로드
```
wget -O fluent-bit.yaml https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/fluent-bit.yaml?raw=true
```

#### 9.3.2 yaml파일 수정
```
FLUENT_ELASTICSEARCH_HOST의 value를 나의 일레스틱서치 alb주소로 바꾸자
* http://빼고 넣자



#### 9.3.3 네임스페이스 생성
```
kubectl create namespace logging
```

#### 9.3.4 fluentd-bit 설치
```
kubectl apply -f fluent-bit.yaml
```


#### 9.3.5 확인
```
kubectl get pod -n logging
```
* 노드의 숫자만큼 파드가 뜬다고한다.



###9.4 kibana에서 확인

####9.4.1 인덱스 패던 만들기
* 키바나 콘솔 접근후 왼쪽 Management - index pattern
* 인덱스 패턴에 melon-* 하고 next step
* @timestamp 선택하고 create pattern
* 왼쪽에 discover 선택하면 
* 로그가 쌓이는게 보인다.
