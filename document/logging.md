



# 9. Logging (EFK)

### 9.1 elasticsearch

#### 9.1.1 yaml파일 다운로드
```
wget -O elasticsearch.yaml https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/elasticsearch.yaml?raw=true
```

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



### 9.2 kibana

#### 9.2.1 yaml파일 다운로드
```
wget -O kibana.yaml https://github.com/sghaha/amazon-eks-hands-on/blob/main/file/kibana.yaml?raw=true
```

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




### 9.3 fluentd-bit 설치

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
