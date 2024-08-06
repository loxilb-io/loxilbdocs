loxilb 고가용성(HA) 배포 방법
========
이 문서에서는 loxilb를 고가용성(HA)으로 배포하는 다양한 시나리오에 대해 설명합니다. 이 페이지를 계속하기 전에  [kube-loxilb](https://loxilb-io.github.io/loxilbdocs/kube-loxilb/)와 loxilb가 지원하는 다양한 [NAT 모드](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/nat.md)에 대한 기본적인 이해를 가지는 것이 좋습니다. loxilb는 아키텍처 선택에 따라 인-클러스터 또는 Kubernetes 클러스터 외부에서 실행할 수 있습니다. 이 문서에서는 인-클러스터 내 배포를 가정하지만, 유사한 구성이 외부 구성 에서도 동일하게 가능합니다.

* [시나리오 1 - Flat L2 네트워킹 (액티브-백업)](#scenario-1----flat-l2-networking-active-backup)
* [시나리오 2 - L3 네트워크 (BGP를 사용하는 액티브-백업 모드)](#scenario-2----l3-network-active-backup-mode-using-bgp)
* [시나리오 3 - L3 네트워크 (BGP ECMP를 사용하는 액티브-액티브)](#scenario-3----l3-network-active-active-with-bgp-ecmp)
* [시나리오 4 - 연결 동기화를 사용하는 액티브-백업](#scenario-4----active-backup-with-connection-sync)
* [시나리오 5 - 빠른 Fail-over 감지를 사용하는 액티브-백업(BFD)](#scenario-5----active-backup-with-fast-failover-detection)

## 시나리오 1 - Flat L2 네트워킹 (액티브-백업)

### 설정
--------
이 배포 시나리오에서는 Kubernetes와 loxilb가 다음과 같이 설정됩니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-L2-1.drawio.svg)

Kubernetes는 2개의 마스터 노드와 2개의 워커 노드로 구성된 클러스터를 사용하며, 모든 노드는 동일한 192.168.80.0/24 서브넷을 사용합니다. 이 시나리오에서는 loxilb가 모든 마스터 노드에서 DaemonSet으로 배포됩니다. 그리고 kube-loxilb는 Deployment로 배포됩니다.

### 적합한 사용 사례
  1. 클라이언트와 서비스가 동일한 서브넷에 있어야 하는 경우.
  2. 엔드포인트가 동일한 서브넷에 있거나 없을 수 있는 경우.
  3. 간단한 배포를 원하는 경우.

### kube-loxilb의 역할과 책임:
  * 로컬 서브넷에서 CIDR 선택.
  * SetRoles 옵션을 선택하여 활성 loxilb pod를 선택할 수 있도록 합니다.
  * loxilb의 상태를 모니터링하고 Fail-over 시 새로운 마스터를 선출합니다.
  * 엔드포인트를 향한 One-arm 서비스 모드에서 loxilb를 설정합니다.

#### 구성 옵션

```
 containers:
      - name: kube-loxilb
        image: ghcr.io/loxilb-io/kube-loxilb:latest
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        ....
        ....
        - --externalCIDR=192.168.80.200/24
        - --setRoles=0.0.0.0
        - --setLBMode=1
```

  *  <b>"--externalCIDR=192.168.80.200/24" -</b> svc의 외부 서비스 IP는 externalCIDR 범위에서 선택됩니다. 이 시나리오에서는 클라이언트, svc 및 클러스터가 동일한 서브넷에 있습니다.
  *  <b>"--setRoles=0.0.0.0" -</b> 이 옵션을 사용하면 kube-loxilb가 loxilb 인스턴스 중에서 활성-백업을 선택하고 svc IP를 활성 loxilb 노드에 구성할 수 있습니다.
  *  <b>"--setLBMode=1" -</b> 이 옵션을 사용하면 kube-loxilb가 엔드포인트를 향한 One-arm 모드에서 svc를 구성할 수 있습니다.

샘플 kube-loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb.yaml)에서 찾을 수 있습니다.

### loxilb의 역할과 책임:

  * svc로 향하는 외부 트래픽을 추적하고 엔드포인트로 전달합니다.
  * 엔드포인트의 상태를 모니터링하고 활성 엔드포인트를 선택합니다(구성된 경우).

#### 구성 옵션

```
 containers:
      - name: loxilb-app
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        command: [ "/root/loxilb-io/loxilb/loxilb", "--egr-hooks", "--blacklist=cni[0-9a-z]|veth.|flannel." ]
        ports:
        - containerPort: 11111
        - containerPort: 179
        - containerPort: 50051
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
```

  
  * <b>"--egr-hooks" -</b> 워크로드가 마스터 노드에 스케줄링될 수 있는 경우 필요합니다. 워커 노드로 워크로드 스케줄링을 관리하는 경우 해당 변수를 설정할 필요는 없습니다.

  * <b>"--blacklist=cni[0-9a-z]|veth.|flannel." -</b> 클러스터 내 모드에서 실행하는 경우 필수입니다. loxilb는 모든 인터페이스에 ebpf 프로그램을 부착하지만 기본 네임스페이스에서 실행 중이므로 모든 인터페이스(CNI 인터페이스 포함)가 노출되고 loxilb는 이러한 인터페이스에 ebpf 프로그램을 부착하게 됩니다. 이는 원하는 바가 아니므로 사용자는 이러한 인터페이스를 제외하는 정규 표현식을 언급해야 합니다. 주어진 예제의 정규 표현식은 flannel 인터페이스를 제외합니다. "--blacklist=cali.|tunl.|vxlan[.]calico|veth.|cni[0-9a-z]" 정규 표현식은 calico CNI와 함께 사용해야 합니다.

샘플 loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/loxilb.yaml)에서 찾을 수 있습니다.

### Fail-Over

이 다이어그램은 Fail-over 시나리오를 설명합니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-L2-2.drawio.svg)

kube-loxilb는 loxilb의 상태를 지속적으로 모니터링합니다. 장애가 발생하면 loxilb의 상태 변경을 감지하고 사용 가능한  loxilb pod 풀에서 새로운 “활성”을 할당합니다. 새로운 pod는 이전에 다른 loxilb pod에 할당된 svcIP를 상속받아 서비스가 새롭게 활성화된 loxilb pod에 의해 제공됩니다.

## 시나리오 2 - L3 네트워크 (BGP를 사용하는 액티브-백업 모드)

### 설정
--------
이 배포 시나리오에서는 Kubernetes와 loxilb가 다음과 같이 설정됩니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-L3-1.drawio.svg)

Kubernetes는 2개의 마스터 노드와 2개의 워커 노드로 구성된 클러스터를 사용하며, 모든 노드는 동일한 192.168.80.0/24 서브넷을 사용합니다. SVC는 클러스터/로컬 서브넷이 아닌 외부 IP를 가집니다.
이 시나리오에서는 loxilb가 모든 마스터 노드에서 DaemonSet으로 배포됩니다.
그리고 kube-loxilb는 Deployment로 배포됩니다.

### 적합한 사용 사례
  1. 클라이언트와 클러스터가 다른 서브넷에 있는 경우.
  2. 클라이언트와 svc VIP가 다른 서브넷에 있어야 하는 경우(클러스터 엔드포인트도 다른 네트워크에 있을 수 있음).
  3. 클라우드 배포에 이상적입니다.

### kube-loxilb의 역할과 책임:
  * 다른 서브넷에서 CIDR을 선택합니다.
  * SetRoles 옵션을 선택하여 활성 loxilb pod를 선택할 수 있도록 합니다.
  * loxilb의 상태를 모니터링하고 Fail-over 시 새로운 마스터를 선출합니다.
  * loxilb Pod 간의 BGP 피어링 프로비저닝을 자동화합니다.
  * 엔드포인트를 향한 One-arm 서비스 모드에서 loxilb를 설정합니다.

#### 구성 옵션
```
 containers:
      - name: kube-loxilb
        image: ghcr.io/loxilb-io/kube-loxilb:latest
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        ....
        ....
        - --externalCIDR=123.123.123.1/24
        - --setRoles=0.0.0.0
        - --setLBMode=1
        - --setBGP=65100
        - --extBGPPeers=50.50.50.1:65101
```
  *  <b>"--externalCIDR=123.123.123.1/24" -</b> svc의 외부 서비스 IP는 externalCIDR 범위에서 선택됩니다. 이 시나리오에서는 클라이언트, svc 및 클러스터가 모두 다른 서브넷에 있습니다.
  *  <b>"--setRoles=0.0.0.0" -</b> 이 옵션을 사용하면 kube-loxilb가 loxilb 인스턴스 중에서 활성-백업을 선택하고 svc IP를 활성 loxilb 노드에 구성할 수 있습니다.
  *  <b>"--setLBMode=1" -</b> 이 옵션을 사용하면 kube-loxilb가 엔드포인트를 향한 One-arm 모드에서 svc를 구성할 수 있습니다.
  *  <b>"--setBGP=65100" -</b> 이 옵션을 사용하면 kube-loxilb가 BGP 인스턴스에 로컬 AS 번호를 구성할 수 있습니다.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> 이 옵션을 사용하면 BGP 인스턴스의 외부 이웃을 구성할 수 있습니다.

샘플 kube-loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb.yaml)에서 찾을 수 있습니다.

### loxilb의 역할과 책임:

  * 상태(활성 또는 백업)에 따라 SVC IP를 광고합니다.
  * svc로 향하는 외부 트래픽을 추적하고 엔드포인트로 전달합니다.
  * 엔드포인트의 상태를 모니터링하고 활성 엔드포인트를 선택합니다(구성된 경우).

#### 구성 옵션
```
 containers:
      - name: loxilb-app
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        command: [ "/root/loxilb-io/loxilb/loxilb", "--bgp", "--egr-hooks", "--blacklist=cni[0-9a-z]|veth.|flannel." ]
        ports:
        - containerPort: 11111
        - containerPort: 179
        - containerPort: 50051
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
```

  
  * <b>"--bgp" -</b> 옵션은 loxilb가 goBGP 인스턴스와 함께 실행되어 활성/백업 상태에 따라 적절한 우선순위로 경로를 광고할 수 있게 합니다.
  
  * <b>"--egr-hooks" -</b> 워크로드가 마스터 노드에 스케줄링될 수 있는 경우 필요합니다. 워커 노드로 워크로드 스케줄링을 관리하는 경우 이 인수를 언급할 필요는 없습니다.

  * <b>"--blacklist=cni[0-9a-z]|veth.|flannel." -</b> 클러스터 내 모드에서 실행하는 경우 필수입니다. loxilb는 모든 인터페이스에 ebpf 프로그램을 부착하지만 기본 네임스페이스에서 실행 중이므로 모든 인터페이스(CNI 인터페이스 포함)가 노출되고 loxilb는 이러한 인터페이스에 ebpf 프로그램을 부착하게 됩니다. 이는 원하는 바가 아니므로 사용자는 이러한 인터페이스를 제외하는 정규 표현식을 언급해야 합니다. 주어진 예제의 정규 표현식은 flannel 인터페이스를 제외합니다. "--blacklist=cali.|tunl.|vxlan[.]calico|veth.|cni[0-9a-z]" 정규 표현식은 calico CNI와 함께 사용해야 합니다.

샘플 loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/loxilb.yaml)에서 찾을 수 있습니다.

### Fail-over

이 다이어그램은 Fail-over 시나리오를 설명합니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-L3-2.drawio.svg)

kube-loxilb는 loxilb의 상태를 지속적으로 모니터링합니다. 장애가 발생하면 loxilb의 상태 변경을 감지하고 사용 가능한 loxilb pod 풀에서 새로운 “활성”을 할당합니다. 새로운 pod는 이전에 다른 loxilb pod에 할당된 svcIP를 상속받아 새 상태에 따라 SVC IP를 광고합니다. 클라이언트는 SVCIP에 대한 새로운 경로를 수신하고 서비스는 새로 활성화된 loxilb pod에 의해 제공됩니다.

## 시나리오 3 - L3 네트워크 (BGP ECMP를 사용하는 액티브-액티브)

### 설정
--------
이 배포 시나리오에서는 Kubernetes와 loxilb가 다음과 같이 설정됩니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-L3-ECMP.drawio.svg)

Kubernetes는 2개의 마스터 노드와 2개의 워커 노드로 구성된 클러스터를 사용하며, 모든 노드는 동일한 192.168.80.0/24 서브넷을 사용합니다. SVC는 클러스터/로컬 서브넷이 아닌 외부 IP를 가집니다.
이 시나리오에서는 loxilb가 모든 마스터 노드에서 DaemonSet으로 배포됩니다.
그리고 kube-loxilb는 Deployment로 배포됩니다.

### 적합한 사용 사례
  1. 클라이언트와 클러스터가 다른 서브넷에 있는 경우.
  2. 클라이언트와 svc VIP가 다른 서브넷에 있어야 하는 경우(클러스터 엔드포인트도 다른 네트워크에 있을 수 있음).
  3. 클라우드 배포에 이상적입니다.
  4. 액티브-액티브 클러스터링으로 인해 더 나은 성능이 필요하지만 네트워크 장치/호스트가 ECMP를 지원해야 합니다.

### kube-loxilb의 역할과 책임:
  * 다른 서브넷에서 CIDR을 선택합니다.
  * 이 경우 SetRoles 옵션을 선택하지 마세요(svcIP가 동일한 attributes/prio/med 로 광고됩니다).
  * loxilb Pod 간의 BGP 피어링 프로비저닝을 자동화합니다.
  * 엔드포인트를 향한 One-arm 서비스 모드에서 loxilb를 설정합니다.

#### 구성 옵션
```
 containers:
      - name: kube-loxilb
        image: ghcr.io/loxilb-io/kube-loxilb:latest
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        ....
        ....
        - --externalCIDR=123.123.123.1/24
        - --setLBMode=1
        - --setBGP=65100
        - --extBGPPeers=50.50.50.1:65101
```
 
  *  <b>"--externalCIDR=123.123.123.1/24" -</b> svc의 외부 서비스 IP는 externalCIDR 범위에서 선택됩니다. 이 시나리오에서는 클라이언트, svc 및 클러스터가 모두 다른 서브넷에 있습니다.
  *  <b>"--setLBMode=1" -</b> 이 옵션을 사용하면 kube-loxilb가 엔드포인트를 향한 One-arm 모드에서 svc를 구성할 수 있습니다.
  *  <b>"--setBGP=65100" -</b> 이 옵션을 사용하면 kube-loxilb가 BGP 인스턴스에 로컬 AS 번호를 구성할 수 있습니다.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> 이 옵션을 사용하면 BGP 인스턴스의 외부 이웃을 구성할 수 있습니다.

샘플 kube-loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb.yaml)에서 찾을 수 있습니다.

### loxilb의 역할과 책임:

  * 동일한 속성으로 SVC IP를 광고합니다.
  * svc로 향하는 외부 트래픽을 추적하고 엔드포인트로 전달합니다.
  * 엔드포인트의 상태를 모니터링하고 활성 엔드포인트를 선택합니다(구성된 경우).

#### 구성 옵션
```
 containers:
      - name: loxilb-app
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        command: [ "/root/loxilb-io/loxilb/loxilb", "--bgp", "--egr-hooks", "--blacklist=cni[0-9a-z]|veth.|flannel." ]
        ports:
        - containerPort: 11111
        - containerPort: 179
        - containerPort: 50051
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
```

  
  * <b>"--bgp" -</b> 옵션은 loxilb가 동일한 속성으로 경로를 광고할 수 있게 합니다.
  
  * <b>"--egr-hooks" -</b> 워크로드가 마스터 노드에 스케줄링될 수 있는 경우 필요합니다. 워커 노드로 워크로드 스케줄링을 관리하는 경우 이 인수를 언급할 필요는 없습니다.

  * <b>"--blacklist=cni[0-9a-z]|veth.|flannel." -</b> 클러스터 내 모드에서 실행하는 경우 필수입니다. loxilb는 모든 인터페이스에 ebpf 프로그램을 부착하지만 기본 네임스페이스에서 실행 중이므로 모든 인터페이스(CNI 인터페이스 포함)가 노출되고 loxilb는 이러한 인터페이스에 ebpf 프로그램을 부착하게 됩니다. 이는 원하는 바가 아니므로 사용자는 이러한 인터페이스를 제외하는 정규 표현식을 언급해야 합니다. 주어진 예제의 정규 표현식은 flannel 인터페이스를 제외합니다. "--blacklist=cali.|tunl.|vxlan[.]calico|veth.|cni[0-9a-z]" 정규 표현식은 calico CNI와 함께 사용해야 합니다.

샘플 loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/loxilb.yaml)에서 찾을 수 있습니다.

### Fail-over

이 다이어그램은 Fail-over 시나리오를 설명합니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-L3-ECMP-2.drawio.svg)

장애가 발생한 경우, 클라이언트에서 실행 중인 BGP는 ECMP 경로를 업데이트하고 트래픽을 활성 ECMP 엔드포인트로 보내기 시작합니다.

## 시나리오 4 - 연결 동기화를 사용하는 액티브-백업

### 설정
--------
이 배포 시나리오에서는 Kubernetes와 loxilb가 다음과 같이 설정됩니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-CT-Sync-1.drawio.svg)

이 기능은 loxilb가 기본 모드 또는 Full NAT 모드에서 Kubernetes 클러스터 외부에서 실행될 때만 지원됩니다. Kubernetes는 2개의 마스터 노드와 2개의 워커 노드로 구성된 클러스터를 사용하며, 모든 노드는 동일한 192.168.80.0/24 서브넷을 사용합니다. SVC는 외부 IP를 가집니다.

외부 클라이언트, loxilb 및 Kubernetes 클러스터의 연결에 따라 몇 가지 가능한 시나리오가 있습니다. 이 시나리오에서는 L3 연결을 고려합니다.

### 적합한 사용 사례
  1. loxilb pod 장애 시 장기 실행 연결을 유지해야 하는 경우
  2. DSR 모드로 알려진 다른 LB 모드를 사용하여 연결을 유지할 수 있지만 다음과 같은 제한 사항이 있습니다:
     1. 상태 기반 필터링 및 연결 추적을 보장할 수 없습니다.
     2. 멀티호밍 기능을 지원할 수 없습니다. 다른 5-튜플이 동일한 연결에 속할 수 있기 때문입니다.

### kube-loxilb의 역할과 책임:
  * 필요한 대로 CIDR을 선택합니다.
  * SetRoles 옵션을 선택하여 활성 loxilb를 선택할 수 있도록 합니다(svcIP가 다른 attributes/prio/med 로 광고됩니다).
  * loxilb 컨테이너 간의 BGP 피어링 프로비저닝을 자동화합니다(필요한 경우).
  * 엔드포인트를 향한 Full NAT 서비스 모드에서 loxilb를 설정합니다.

#### 구성 옵션
```
 containers:
      - name: kube-loxilb
        image: ghcr.io/loxilb-io/kube-loxilb:latest
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        ....
        ....
        - --setRoles=0.0.0.0
        - --loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111
        - --externalCIDR=123.123.123.1/24
        - --setLBMode=2
        - --setBGP=65100
        - --extBGPPeers=50.50.50.1:65101
```

  
  *  <b>"--setRoles=0.0.0.0" -</b> 이 옵션을 사용하면 kube-loxilb가 loxilb 인스턴스 중에서 활성-백업을 선택할 수 있습니다.
  *  <b>"--loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111" -</b> 연결할 loxilb URL입니다.
  *  <b>"--externalCIDR=123.123.123.1/24" -</b> svc의 외부 서비스 IP는 externalCIDR 범위에서 선택됩니다. 이 시나리오에서는 클라이언트, svc 및 클러스터가 모두 다른 서브넷에 있습니다.
  *  <b>"--setLBMode=2" -</b> 이 옵션을 사용하면 kube-loxilb가 엔드포인트를 향한 Full NAT 모드에서 svc를 구성할 수 있습니다.
  *  <b>"--setBGP=65100" -</b> 이 옵션을 사용하면 kube-loxilb가 BGP 인스턴스에 로컬 AS 번호를 구성할 수 있습니다.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> 이 옵션을 사용하면 BGP 인스턴스의 외부 이웃을 구성할 수 있습니다.

샘플 kube-loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/ext-cluster/kube-loxilb.yaml)에서 찾을 수 있습니다.

### loxilb의 역할과 책임:

  * 상태(활성/백업)에 따라 SVC IP를 광고합니다.
  * svc로 향하는 외부 트래픽을 추적하고 엔드포인트로 전달합니다.
  * 엔드포인트의 상태를 모니터링하고 활성 엔드포인트를 선택합니다(구성된 경우).
  * 장기 실행 연결을 다른 구성된 loxilb 피어와 동기화합니다.

#### 실행 옵션

```
#llb1
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest  --cluster=$llb2IP --self=0 -b

#llb2
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest --cluster=$llb1IP --self=1 -b
```

  
  * <b>"--cluster=\<llb-peer-IP\>" -</b> 옵션은 동기화를 위한 피어 loxilb IP를 구성합니다.
  * <b>"--self=0/1" -</b> 옵션은 인스턴스를 식별합니다.
  * <b>"-b" -</b> 옵션은 loxilb가 활성/백업 상태에 따라 적절한 우선순위로 경로를 광고할 수 있도록 goBGP 인스턴스와 함께 실행되도록 합니다.

### Fail-over

이 다이어그램은 Fail-over 시나리오를 설명합니다:

![설정](photos/loxilb-k8s-arch-LoxiLB-HA-CT-Sync-2.drawio.svg)

장애가 발생하면 kube-loxilb는 장애를 감지합니다. 활성 loxilb 풀에서 새로운 loxilb를 선택하고 이를 새로운 마스터로 업데이트합니다. 새로운 마스터 loxilb는 높은 우선순위로 svcIP를 광고하여 클라이언트에서 실행 중인 BGP가 트래픽을 새로운 마스터 loxilb로 보내도록 강제합니다. 연결이 모두 동기화되어 있으므로 새로운 마스터 loxilb는 지정된 엔드포인트로 트래픽을 보내기 시작합니다.

이 기능에 대해 자세히 알아보려면 ["Hitless HA"](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing) 블로그를 읽어보세요.

## 시나리오 5 - 빠른 Fail-over 감지를 사용하는 액티브-백업

### 설정
--------
이 배포 시나리오에서는 Kubernetes와 loxilb가 다음과 같이 설정됩니다:

![설정](photos/bfd-1.svg)

이 기능은 loxilb가 Kubernetes 클러스터 외부에서 실행될 때만 지원됩니다. Kubernetes는 2개의 마스터 노드와 2개의 워커 노드로 구성된 클러스터를 사용하며, 모든 노드는 동일한 192.168.80.0/24 서브넷을 사용합니다. SVC는 외부 IP를 가집니다.

외부 클라이언트, loxilb 및 Kubernetes 클러스터의 연결에 따라 몇 가지 가능한 시나리오가 있습니다. 이 시나리오에서는 연결 동기화와 함께 L2 연결을 고려합니다.

### 적합한 사용 사례
  1. 빠른 Fail-over 감지 및 서비스 연속성이 필요한 경우.
  2. 이 기능은 L2 또는 L3 네트워크 설정에서 모두 작동합니다.

### kube-loxilb의 역할과 책임:
  * 필요한 대로 CIDR을 선택합니다.
  * SetRoles 옵션을 비활성화하여 활성 loxilb를 선택하지 않도록 합니다.
  * loxilb 컨테이너 간의 BGP 피어링 프로비저닝을 자동화합니다(필요한 경우).
  * 엔드포인트를 향한 구성된 서비스 모드에서 loxilb를 설정합니다.

#### 구성 옵션
```
 containers:
      - name: kube-loxilb
        image: ghcr.io/loxilb-io/kube-loxilb:latest
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        ....
        ....
        # Disable setRoles option
        #- --setRoles=0.0.0.0
        - --loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111
        - --externalCIDR=192.168.80.5/32
        - --setLBMode=2
```

  
  *  <b>"--setRoles=0.0.0.0" -</b> SetRoles 옵션을 비활성화해야 합니다. 이 옵션을 활성화하면 kube-loxilb가 loxilb 인스턴스 중에서 활성-백업을 선택하게 됩니다.
  *  <b>"--loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111" -</b> 연결할 loxilb URL입니다.
  *  <b>"--externalCIDR=192.168.80.5/32" -</b> svc의 외부 서비스 IP는 externalCIDR 범위에서 선택됩니다. 이 시나리오에서는 클라이언트, svc 및 클러스터가 모두 동일한 서브넷에 있습니다.
  *  <b>"--setLBMode=2" -</b> 이 옵션을 사용하면 kube-loxilb가 엔드포인트를 향한 Full NAT 모드에서 svc를 구성할 수 있습니다.

샘플 kube-loxilb.yaml은 [여기](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/ext-cluster/kube-loxilb.yaml)에서 찾을 수 있습니다.

### loxilb의 역할과 책임:

  * 상태(활성/백업)에 따라 SVC IP를 광고합니다.
  * svc로 향하는 외부 트래픽을 추적하고 엔드포인트로 전달합니다.
  * 엔드포인트의 상태를 모니터링하고 활성 엔드포인트를 선택합니다(구성된 경우).
  * 장기 실행 연결을 다른 구성된 loxilb 피어와 동기화합니다.

#### 실행 옵션

```
#llb1
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest  --cluster=192.168.80.2 --self=0 --ka=192.168.80.2:192.168.80.1

#llb2
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest --cluster=192.168.80.1 --self=1 --ka=192.168.80.1:192.168.80.2
```

  
  * <b>"--ka=\<llb-peer-IP\>:\<llb-self-IP\>" -</b> 옵션은 BFD를 위한 피어 loxilb IP와 소스 IP를 구성합니다.
  * <b>"--cluster=\<llb-peer-IP\>" -</b> 옵션은 동기화를 위한 피어 loxilb IP를 구성합니다.
  * <b>"--self=0/1" -</b> 옵션은 인스턴스를 식별합니다.

### Fail-over

이 다이어그램은 Fail-over 시나리오를 설명합니다:

![설정](photos/bfd-2.svg)

장애가 발생하면 BFD가 장애를 감지합니다. 백업 loxilb는 새로운 마스터로 상태를 업데이트합니다. 새로운 마스터 loxilb는 gARP 또는 BGP를 실행 중인 경우 더 높은 우선순위로 svcIP를 광고하여 클라이언트가 트래픽을 새로운 마스터 loxilb로 보내도록 강제합니다. 연결이 모두 동기화되어 있으므로 새로운 마스터 loxilb는 지정된 엔드포인트로 트래픽을 보내기 시작합니다.

이 기능에 대해 자세히 알아보려면 ["Fast Failover Detection with BFD"](https://www.loxilb.io/post/bringing-sub-second-resilience-in-kubernetes-cluster) 블로그를 읽어보세요.

## 참고:

loxilb를 DSR 모드, DNS 등에서 사용하는 방법은 이 문서에서 자세히 다루지 않았습니다. 시나리오를 계속 업데이트할 예정입니다.
