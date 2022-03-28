# 8. Monitoring (Prometheus & Grafana)

### 8.1 헬름 설치

#### 8.1.1 헬름 설치 확인
```
helm version
```

#### 8.1.2 설치
* 설치 안되어있으면 설치
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
```
```
chmod 700 get_helm.sh
```
```
./get_helm.sh
```


### 8.2 Prometheus

#### 8.2.1 Prometheus 네임스페이스 생성
```
kubectl create namespace prometheus
```

#### 8.2.2 네임스페이스 생성 확인	
```
kubectl get namespace	
```

* 결과 예시
```
NAME              STATUS   AGE
cert-manager      Active   22m
default           Active   33m
kube-node-lease   Active   33m
kube-public       Active   33m
kube-system       Active   33m
prometheus        Active   16s"
```


#### 8.2.3 prometheus-community 차트 리포지토리 추가

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```


#### 8.2.4 Prometheus 배포	

```
helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

#### 8.2.5 참고 	
```
이 명령을 실행할 때 
Error: failed to download "stable/prometheus" (hint: running `helm repo update` may help) 
오류가 발생하면 helm repo update를 실행한 다음 레포 등록부터 해보자

Error: rendered manifests contain a resource that already exists 
오류가 발생하면 helm uninstall your-release-name -n namespace를 실행한 다음 배포하자
```

#### 8.2.6 확인
```
kubectl get pods -n prometheus	
```
* 결과예시
```
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-alertmanager-6c845bbb9c-jhxxj         2/2     Running   0          3m14s
prometheus-kube-state-metrics-6c44ff7fb6-fwhcz   1/1     Running   0          3m14s
prometheus-node-exporter-6hfcj                   1/1     Running   0          3m14s
prometheus-pushgateway-86679dcf68-l5bds          1/1     Running   0          3m14s
prometheus-server-869789c557-g74pt               2/2     Running   0          3m14s
```


#### 8.2.7 프로메테우스 alb 생성
```
kubectl patch svc prometheus-server -n prometheus -p '{"spec": {"type": "LoadBalancer"}}'
```



#### 8.2.8 url확인	
```
kubectl get svc -n prometheus
```
* 결과 예시
```
NAME                            TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)        AGE
prometheus-alertmanager         ClusterIP      10.100.167.44    <none>                                                                        80/TCP         13m
prometheus-kube-state-metrics   ClusterIP      10.100.192.6     <none>                                                                        8080/TCP       13m
prometheus-node-exporter        ClusterIP      None             <none>                                                                        9100/TCP       13m
prometheus-pushgateway          ClusterIP      10.100.143.247   <none>                                                                        9091/TCP       13m
prometheus-server               LoadBalancer   10.100.14.124    a4989f9b647394afd92aa8f1e0f65099-199140108.ap-northeast-2.elb.amazonaws.com   80:31117/TCP   13m
```


#### 8.2.9 prometheus 콘솔 접속	
a4989f9b647394afd92aa8f1e0f65099-199140108.ap-northeast-2.elb.amazonaws.com



### 8.3 Grafana

#### 8.3.1 Grafana 설치	

* https://grafana.com/docs/grafana/latest/installation/kubernetes/ 에 있는 grafana.yaml을 쓴다
```
kubectl apply -f grafana.yaml
```

#### 8.3.2 url 확인
```
kubectl get svc
```
* 결과 예시
```
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)          AGE
grafana      LoadBalancer   10.100.158.37    ad68155bf162b44c1b6dd61faa7ceef6-1225394448.ap-northeast-2.elb.amazonaws.com   3000:32767/TCP   30s
kubernetes   ClusterIP      10.100.0.1       <none>                                                                         443/TCP          60m
react-v4     NodePort       10.100.237.248   <none>                                                                         80:30897/TCP     42m"
```


#### 8.3.3 콘솔 확인	

* ad68155bf162b44c1b6dd61faa7ceef6-1225394448.ap-northeast-2.elb.amazonaws.com:3000



#### 8.3.4 로그인	

* 초기 설정
* id : admin
* pw : admin


#### 8.3.5 data source 지정
* 왼쪽 configuration 버튼  - data source - add data source - prometheus


#### 8.3.6 data source 지정 2	

* http - url에 프로메테우스 alb 주소 넣음
* http://a4989f9b647394afd92aa8f1e0f65099-199140108.ap-northeast-2.elb.amazonaws.com/



#### 8.3.7 test
* save & test 버튼 클릭
* Data source is working

#### 8.3.8 확인
* Data Source 페이지 가면 추가된것이 보임



#### 8.3.9 Dashboard import

* 왼쪽 + 버튼 클릭 - import 클릭
* Import via grafana.com 에 11074 넣고 load
* 화면 바뀌면 VictoriaMetrics에 프로메테우스 지정
* import클릭
* (https://grafana.com/grafana/dashboards/11074 많이 사용하는 거란다)
