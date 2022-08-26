loxilb & calico BGP 연동
========
이 문서에서는 calico CNI를 사용하는 kubernetes와 loxilb를 연동하는 방법을 설명합니다.

환경
--------
이 예제에서는 kubernetes와 loxilb가 다음과 같이 연결되어 있다고 가정합니다.
kubernetes는 단순함을 위해서 단일 마스터 클러스터를 사용하며 모든 클러스터는 192.168.57.0/24 동일한 서브넷을 사용합니다.
loxilb 컨테이너가 실행중인 로드밸런서 노드 역시 kubernetes와 동일한 서브넷에 연결되어 있습니다. 외부에서 kubernetes접속은 모두 로드밸런서 노드와 loxilb를 거치도록 설정했습니다.

해당 예제에서는 docker를 사용해 loxilb 컨테이너를 실행합니다.
해당 예제에서는 kubernetes & calico는 이미 설치되어 있다고 가정하고 설명합니다.

## 1. loxilb container 생성
### 1.1 docker network 생성
우선 loxilb와 kubernetes 연동을 위해서는 서로 통신할 수 있어야 합니다. 
kubernetes & 로드밸런서 노드가 연결되어 있는 네트워크에 loxilb 컨테이너도 연결되도록 docker network를 생성합니다. 현재 로드밸런서 노드는 eno6 인터페이스를 통해 kubernetes와 연결되어 있습니다. 따라서 eno6 인터페이스를 parent로 사용하는 macvlan 타입 docker network 만들어서 loxilb 컨테이너에 제공하도록 하겠습니다.
다음 명령어로 docker network를 생성합니다.
```
sudo docker network create -d macvlan -o parent=eno6 \
  --subnet 192.168.57.0/24 \
  --gateway 192.168.57.1 \
  --aux-address 'cp1=192.168.57.101' \
  --aux-address 'cp2=192.168.57.102' \
  --aux-address 'cp3=192.168.57.103' k8snet
```
|옵션|설명|
|----|----|
|-d macvlan|네트워크 타입을 macvlan으로 지정|
|-o parent=eno6|eno6 인터페이스를 parent로 사용해서 macvlan type 네트워크 생성|
|--subnet 192.168.57.0/24|네트워크 서브넷 지정|
|--gateway 192.168.57.1|게이트웨이 설정(생략 가능)|
|--aux-address 'serverName=serverIP'|해당 네트워크에서 이미 사용중인 IP 주소들이 중복으로 컨테이너에 할당되지 않도록 미리 등록하는 옵션|
|k8snet|네트워크 이름을 k8snet으로 지정|

외부에서 kubernetes 서비스로 접근하는 트래픽 역시 loxilb를 거치도록, 외부와 통신이 가능한 docker network 역시 생성합니다. 로드밸런서 노드는 eno8을 통해 외부와 연결되어 있습니다.
```
sudo docker network create -d macvlan -o parent=eno8 \
  --subnet 192.168.20.0/24 \
  --gateway 192.168.20.1 llbnet
```

docker network list 명령어로 생성한 네트워크를 확인할 수 있습니다.
```
netlox@nd8:~$ sudo docker network list
NETWORK ID     NAME      DRIVER    SCOPE
5c97ae74fc32   bridge    bridge    local
6142f53e8be6   host      host      local
24ee7dbd7707   k8snet    macvlan   local
81c96ceda375   llbnet    macvlan   local
7bcd1738501b   none      null      local
```

### 1.2 loxilb container 생성
loxilb container 이미지는 [github][loxilbContainerImageUrl]에서 제공되고 있습니다.
컨테이너 이미지만 먼저 다운로드하고 싶을 경우 다음 명령어를 사용합니다.
```
docker pull ghcr.io/loxilb-io/loxilb:latest
```

다음 명령어로 loxilb 컨테이너를 생성할 수 있습니다.
```
sudo docker run -u root --cap-add SYS_ADMIN --restart unless-stopped \
  --privileged -dit -v /dev/log:/dev/log \
  --net=k8snet --ip=192.168.57.4 --name loxilb ghcr.io/loxilb-io/loxilb:latest \
  --host=0.0.0.0
```
사용자가 지정해야 하는 옵션은 다음과 같습니다.
|옵션|설명|
|----|----|
|--net=k8snet|컨테이너를 연결할 네트워크|
|--ip=192.168.57.4|컨테이너가 사용할 IP address 지정. 지정하지 않을 경우 network 서브냇 범위 내에서 임의의 IP 사용|
|--name loxilb|컨테이너 이름 설정|

docker ps 명령어로 생성한 컨테이너를 확인할 수 있습니다.
```
netlox@nd8:~$ sudo docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS       NAMES
eae349a283ae   loxilbio/loxilb:beta   "/root/loxilb-io/lox…"   11 days ago   Up 11 days               loxilb
```

위에서 컨테이너 생성할 때 kubernetes 네트워크만 연결했기 때문에, 외부 통신용 docker network와도 연결해야 합니다.
다음 명령어로 컨테이너에 네트워크를 주가로 연결할 수 있습니다.
```
sudo docker network connect llbnet loxilb
```

연결이 완료되면 다음과 같이 컨테이너의 인터페이스 2개를 확인할 수 있습니다
```
netlox@netlox:~$ sudo docker exec -ti loxilb ip route
default via 192.168.20.1 dev eth0
192.168.20.0/24 dev eth0 proto kernel scope link src 192.168.20.4
192.168.30.0/24 dev eth1 proto kernel scope link src 192.168.30.2
```

[loxilbContainerImageUrl]: https://github.com/loxilb-io/loxilb/pkgs/container/loxilb

## 2. kubernetes에 loxi-ccm 설치
loxi-ccm은 loxilb 로드밸런서를 kubernetes에게 제공하기 위한 [cloud-controller-manager][k8sCcmDoc] 로서, kubernetes와 loxilb 연동에 반드시 필요합니다.
[해당 문서][loxiCcmHowTo]를 참고해서, configMap의 apiServerURL을 위에서 생성한 loxilb의 IP 주소로 변경한 후 kubernetes에 설치하시면 됩니다.
loxi-ccm까지 정상적으로 설치되었다면 연동 작업이 완료됩니다.

[k8sCcmDoc]: https://kubernetes.io/ko/docs/concepts/architecture/cloud-controller/
[loxiCcmHowTo]: https://github.com/loxilb-io/loxilbdocs/blob/main/docs/ccm.md

## 3. 기본 연동 확인
2번 항목까지 완료되었다면, 이제 kubernetes에서 LoadBalancer 타입 서비스를 생성하면 External IP가 부여됩니다.
다음과 같이 테스트용으로 test-nginx-svc.yaml 파일을 생성합니다.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
      - containerPort: 80
        name: http-web-svc
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 8888
    targetPort: http-web-svc
```

파일을 생성한 다음, 아래 명령으로 nginx pod와 LoadBalancer 서비스를 생성합니다.
```
kubectl apply -f test-nginx-svc.yaml
```

서비스 nginx-service가 LoadBalancer 타입으로 생성되었고 External IP를 할당받았음을 확인할 수 있습니다.   
이제 IP 123.123.123.15와 port 8888을 사용해 외부에서 kubernetes 서비스로 접근이 가능합니다.
```
vagrant@node1:~$ sudo kubectl get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
kubernetes      ClusterIP      10.233.0.1      <none>           443/TCP          28d
nginx-service   LoadBalancer   10.233.21.235   123.123.123.15   8888:31655/TCP   3s
```

LoadBalancer 룰은 loxilb 컨테이너에도 생성됩니다. 로드밸런서 노드에서 다음과 같이 확인할 수 있습니다.
```
netlox@nd8:~$ sudo docker exec -ti loxilb loxicmd get lb
|  EXTERNAL IP   | PORT | PROTOCOL | SELECT | # OF ENDPOINTS |
|----------------|------|----------|--------|----------------|
| 123.123.123.15 | 8888 | tcp      |      0 |              2 |
```

## 4. calico BGP & loxilb 연동
calico에서 BGP 모드로 네트워크를 구성할 경우, loxilb 역시 BGP 모드로 동작해야 합니다. loxilb는 goBGP 기반으로 BGP 네트워크 기능을 지원합니다.
이하 내용은 calico가 BGP mode로 설정되어 있다고 가정하고 설명합니다.

### 4.1 loxilb BGP 모드로 실행
다음 명령어로 loxilb 컨테이너를 생성하면 BGP 모드로 실행됩니다. 명령어 마지막의 -b 옵션이 BGP 모드 옵션입니다.
```
sudo docker run -u root --cap-add SYS_ADMIN --restart unless-stopped \
  --privileged -dit -v /dev/log:/dev/log \
  --net=k8snet --ip=192.168.57.4 --name loxilb ghcr.io/loxilb-io/loxilb:latest \
  --host=0.0.0.0 -b
```

### 4.2 gobgp_loxilb.yaml 파일 생성
loxilb 컨테이너의 /opt/loxilb/ 디렉토리에 gobgp_loxilb.yaml 파일을 생성합니다.
```
global:
    config:
        as: 65002
        router-id: 172.1.0.2
neighbors:
    - config:
        neighbor-address: 192.168.57.101
        peer-as: 64512
    - config:
        neighbor-address: 192.168.20.55
        peer-as: 64001
```

global 항목에는 loxilb 컨테이너의 as-id와 router-id 등 BGP 정보를 등록해야 합니다.
neighbors는 loxilb와 Peering되는 BGP 라우터의 IP 주소 및 as-id 정보를 등록합니다. 해당 예제에서는 calico의 BGP 정보(192.168.57.101)와 외부 BGP 정보(129.168.20.55) 를 등록했습니다.

### 4.3 loxilb 컨테이너의 lo 인터페이스에 router-id 추가
gobgp_loxilb.yaml 파일에서 router-id로 등록한 IP를 lo 인터페이스에 추가해야 합니다.
```
sudo docker exec -ti loxilb ip addr add 172.1.0.2/32 dev lo
```

### 4.4 loxilb 컨테이너 재시작
gobgp_loxilb.yaml에 작성한 설정이 적용되도록 컨테이너를 재시작합니다.
```
sudo docker stop loxilb
sudo docker start loxilb
```

### 4.5 calico에 BGP Peer 정보 추가
calico에도 loxilb의 BGP Peer 정보를 추가해야 합니다. 다음과 같이 calico-bgp-config.yaml 파일을 생성합니다.
```
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: my-global-peers2
spec:
  peerIP: 192.168.57.4
  asNumber: 65002
```

peerIP에 loxilb의 IP 주소를 입력합니다. asNumber에는 위에서 설정한 loxilb BGP의 as-ID를 입력합니다.
파일을 생성한 다음, 아래 명령어로 calico에 BGP Peer 정보를 추가합니다.
```
sudo calicoctl apply -f calico-bgp-config.yaml
```

### 4.6 BGP 설정 확인
이제 다음과 같이 loxilb 컨테이너에서 BGP 연결을 확인할 수 있습니다.
```
netlox@nd8:~$ sudo docker exec -ti loxilb3 gobgp neigh
Peer              AS  Up/Down State       |#Received  Accepted
192.168.57.101 64512 00:00:59 Establ      |        4         4
```

정상적으로 연결되었다면 State가 Establish로 표시됩니다.   
gobgp global rib 명령으로 calico의 route 정보를 확인할 수 있습니다.
```
netlox@nd8:~$ sudo docker exec -ti loxilb3 gobgp global rib
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.233.71.0/26       192.168.57.101       64512                01:02:03   [{Origin: i}]
*> 10.233.74.64/26      192.168.57.101       64512                01:02:03   [{Origin: i}]
*> 10.233.75.0/26       192.168.57.101       64512                01:02:03   [{Origin: i}]
*> 10.233.102.128/26    192.168.57.101       64512                01:02:03   [{Origin: i}]
```
