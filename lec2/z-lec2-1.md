# lecture-2
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec2
```


# Service 

## 1. nginx deployment 배포 
```bash
## 이전 배포 clear
kubectl delete -f deploy.yaml

## 서비스용 배포 
kubectl apply -f deploy.yaml

```
## 2. clusterIP 
clusterip-svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP

```
```sh
k apply -f clusterip-svc.yaml

k get svc
## 가상의 서비스 ip가 생성되었다 

## master-1에서 다음과 같이 cluster-ip로  조회가 가능해진다 
curl http://10.43.144.200

## 서비스가 로드밸렁스 하는지 확인해 본다 (3개 에 모두 적용)
## install-vm에서 실행 
### k9s에서 진행하는것을 추천
k  exec -it nginx-deployment-6cff568d77-ljms8  -- bash

echo nginx-1 > /usr/share/nginx/html/index.html
echo nginx-2 > /usr/share/nginx/html/index.html
echo nginx-3 > /usr/share/nginx/html/index.html

## master-1에서 실행해 본다 
curl http://10.43.82.134

## 또는 for-loop
for i in {1..1000} 
do
  curl http://10.43.144.200
  echo " (${i})"
  sleep 1
done


# endpoints 조회
k get endpoints 
k get ep

```

## 3. nodePort
nodeport-svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      nodePort: 30001
      port: 80
      targetPort: 80
  type: NodePort

```
```bash

k apply -f nodeport-svc.yaml

k get svc
# podd의 속한  node 확인한다 
k get pod -o wide
k get nodes -o wide

# install-vm에서 node의 ip로 조회 해본다( cluseter 외부에서 조회 가능)
curl 172.26.8.74:30001 ## master-1
curl 172.26.3.104:30001  ## worker-1
curl 172.26.9.60:30001   ## worker-2

# 클러스 외부의 Browser에서 
# aws worker node network   방화벽 30001 번 오픈 (master-1,worker-1,worker-2)
# worker node external-ip 로 접속 
curl http://3.39.152.82:30001  ## master-1
curl http://43.201.116.156:30001 ## worker-1
curl http://3.38.191.50:30001  ## worker-2

```

## 4. LoadBalancer
loabalancer-svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
```sh
k apply -f loabalancer-svc.yaml
## pending 확인
```

## externalName
- naver로 테스트 해본다 
external-svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: externalname1
spec:
  type: ExternalName
  externalName: naver.com
```
```sh
k apply -f external-svc.yaml

## test pod 내에서 curl 확인 한다  
## curlimages 이미지로  mycurlpod pod 생성 완료되면  pod 안에 접속해 있게 된다 

kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh
## 직접 접속도 가능
## k exec -it  mycurlpod -- bash

## 네이버 메인페이지 출력
curl -L externalname1.default.svc.cluster.local

```

# clear
```sh
k delete -f loabalancer-svc.yaml
k delete -f deploy.yaml
k delete -f external-svc.yaml

```