# lecture-2
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec2
```


# 1. DaemonSet
- fluent bit은 대표적인 로그 수집기중(LogStash, fluentd, fluentbit) 가장 가볍고(다른 수집기에 비해 10배이상 가볍다)
- 분산한경을 고려하여 만들어졌기에 최근 k8s의 로그 수집기로 각광받고 있다.
- https://fluentbit.io/

fluent-bit-daemonset.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
spec:
  selector:
    matchLabels:
      name: fluentbit
  template:
    metadata:
      labels:
        name: fluentbit
    spec:
      containers:
      - name: aws-for-fluent-bit
        image: amazon/aws-for-fluent-bit:2.1.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true       
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      
```
- replicas가 없다 
- 각 node 갯수만큼  fluent-bit이 생성된다  


```sh
k apply -f fluent-bit-daemonset.yaml

## daemonset 조회
k get ds 

## clear
k delete -f fluent-bit-daemonset.yaml
```

# 2. StatefulSet
nginx-statefulset.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  selector:
    matchLabels:
      app: nginx 
  serviceName: "nginx-headless-svc"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx 
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
```
- 데모를 위해서 Volume 없이 생성한 예이다 
- StatefulSet은 serviceName을 요구하며 해당 서비스는 headless (clusterIP: None) 이어야 한다 

```sh

k apply -f nginx-statefulset.yaml

## StatefulSet 조회
k get sts

# [참고] k8s resource api및 단축키 조회 가능한 명령어 
k api-resources

```

## headless service 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx

```
- headless 서비스는 `clusterIP: None` 으로 설정 하면 된다 

```sh
k get svc 

NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes           ClusterIP      10.43.0.1     <none>        443/TCP        26h
nginx-headless-svc   ClusterIP      None          <none>        80/TCP         12m

k describe svc nginx-headless-svc
```
- cluster-ip에 ip가 할당 되지 않는다  
- 따라서 다음과 같이 호출 해야 한다 
- 
```sh

## mycurlpod에서 실행 
## 없다면 아래로 실행 
## kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh

## cluster내에서 호출시 다음과 같이 cluster domain을 사용할수 있다 
## [서비스명].[네임스페이스].svc.cluster.local
## [서비스명].[네임스페이스].svc
## [서비스명].[네임스페이스]  ## 다른 네임스페이스
## [서비스명] ## 같은 네임스페이스 
curl nginx-headless-svc.default.svc.cluster.local
curl nginx-headless-svc.default.svc
curl nginx-headless-svc

nslookup nginx-headless-svc.default.svc.cluster.local
-----
Server:         10.43.0.10
Address:        10.43.0.10:53

Name:   nginx-headless-svc.default.svc.cluster.local
Address: 10.42.1.52
Name:   nginx-headless-svc.default.svc.cluster.local
Address: 10.42.1.51
Name:   nginx-headless-svc.default.svc.cluster.local
Address: 10.42.1.53

```

## 2.1  clear 
```sh
k delete -f nginx-statefulset.yaml
```