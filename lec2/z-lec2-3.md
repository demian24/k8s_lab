# lecture-2
- install-vm에서 실행 
- ubuntu유저로  실행   
```sh
# cd ~
# git clone https://github.com/io203/k8s-edu.git
cd  k8s-edu/lec2
```

## Horizontal Pod Autoscaling
- 쿠버네티스에서, HorizontalPodAutoscaler 는 워크로드 리소스(예: 디플로이먼트 또는 스테이트풀셋)를 자동으로 업데이트하며, 워크로드의 크기를 수요에 맞게 자동으로 스케일링하는 것을 목표로 한다.
- 수평 스케일링은 부하 증가에 대해 파드를 더 배치하는 것을 뜻한다. 이는 수직 스케일링(쿠버네티스에서는, 해당 워크로드를 위해 이미 실행 중인 파드에 더 많은 자원(예: 메모리 또는 CPU)를 할당하는 것)과는 다르다.
- 부하량이 줄어들고, 파드의 수가 최소 설정값 이상인 경우, HorizontalPodAutoscaler는 워크로드 리소스(디플로이먼트, 스테이트풀셋, 또는 다른 비슷한 리소스)에게 스케일 다운을 지시한다.

<image src="images/hpa.png" />

## 어플리케이션 apache 배포
```sh 
k apply -f hpa-apache.yaml

## hpa 생성 
k apply -f hpa-apache-cpu.yaml
# 명령 기반으로도 가능 
# kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# "hpa" 또는 "horizontalpodautoscaler" 둘 다 사용 가능하다.
kubectl get hpa

# 부하 생성을 유지하면서 나머지 스텝을 수행할 수 있도록,
# 다음의 명령을 별도의 터미널에서 실행한다.
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

## apache pod 수가 증가한다 
k get pod 

## 부하발생이 정지 되면 한참 있다가  다시 원래  minReplicas: 1  개까지 축소 한다 
```