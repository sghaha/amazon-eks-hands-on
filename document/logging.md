



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


