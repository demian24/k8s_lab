# lecture-3
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec3
```

# 1. ingress

## 1.1 ingress controller
nginx ingresscontroller
```sh
# rke2는 기본적으로 nginx-ingressController가 설치 되어 있다 
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml

```
```sh
## rke2 의 ingressController 조회
k get pod -n kube-system | grep ingress-nginx-controller
k get svc -n kube-system | grep ingress-nginx-controller

```
## 1.2 ingress backend 서비스 용 nginx/apache 배포 

nginx-apache.yaml
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
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
  labels:
    app: "httpd"
spec:
  type: ClusterIP
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: "httpd"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
  labels:
    app: "httpd"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: "httpd"
  template:
    metadata:
      labels:
        app: "httpd"
    spec:
      containers:
      - name: httpd
        image: httpd:latest
        ports:
        - name: http
          containerPort: 80
```
```sh
## nginx/apache 배포
k apply -f nginx-apache.yaml

```

## 1.3 ingress rule
- ingress 를 적용하면 외부에서 접속이 가능하다 
ingress-rule.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
      - path: /apache
        pathType: Prefix
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
```
```sh

k apply -f ingress-rule.yaml

k get ingress
## 바로 적용 안될수 있다 조금 시간이 걸릴수 있다 


curl http://172.26.13.104/nginx
curl http://172.26.13.104/apache
curl http://172.26.4.253/apache
curl http://172.26.4.253/nginx
curl http://172.26.5.74/apache
curl http://172.26.5.74/nginx

## external-ip로 접속이 가능해 졌다(browser로도 가능)
## 모든 노드의 external-ip로 접속 가능(default: 80 open)  
curl http://3.39.152.82/nginx
curl http://3.39.152.82/apache

```
## 1.4 host 기반으로 설정 가능하다 
ingress-host-rule.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
spec:
  ingressClassName: nginx
  rules:
  - host: "nginx.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  - host: "apache.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpd-svc
            port:
              number: 80
```
```sh
k apply -f ingress-host-rule.yaml

k get ing

curl http://nginx.3.39.152.82.sslip.io/
curl `http://apache.3.39.152.82.sslip.io/

```

# 2 TLS Termination(Self-signed)

```sh

## 인증서를 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginxsvc/O=nginxsvc"

## TLS Secret
kubectl create secret tls nginx-tls-secret --key tls.key --cert tls.crt

## default namespace에서 조회
k get secret

```

ingress-tls.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ing
  # annotations:
  #   nginx.ingress.kubernetes.io/force-ssl-redirect: "true"  
spec:
  ingressClassName: nginx
  tls:
  - secretName: nginx-tls-secret
  rules:
  - host: "nginx.3.39.152.82.sslip.io"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```
```sh
## ingress-rule 배포
k apply -f ingress-tls.yaml
# 브라우저에서 
http://nginx.3.39.152.82.sslip.io  ## 접속안됨

# aws lightsail의 master-1 의 방화벽(network) 443 포트 추가 
https://nginx.3.39.152.82.sslip.io 

```
## redirect http to https(강제적)
```yaml
annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

# clear 
```yaml
k delete -f ingress-tls.yaml
k delete -f nginx-apache.yaml

```


