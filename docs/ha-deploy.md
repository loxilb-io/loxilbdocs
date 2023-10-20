How to deploy loxilb with High Availability
========
This article describes different scenarios about how to deploy loxilb with High Availability. Before continuing to this page, all readers are advised to have a basic understanding about [kube-loxilb](https://loxilb-io.github.io/loxilbdocs/kube-loxilb/) and the different [NAT modes](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/nat.md) supported by loxilb.

* [Scenario 1 -  Flat L2 Networking (active-backup)](https://github.com/loxilb-io/loxilbdocs/edit/main/docs/ha-deploy.md#scenario-1----flat-l2-networking-active-backup)
* [Scenario 2 -  L3 network (active-backup mode using BGP)](https://github.com/loxilb-io/loxilbdocs/edit/main/docs/ha-deploy.md#scenario-2----l3-network-active-backup-mode-using-bgp)
* [Scenario 3 -  L3 network (active-active with BGP ECMP)](https://github.com/loxilb-io/loxilbdocs/edit/main/docs/ha-deploy.md#scenario-3----l3-network-active-active-with-bgp-ecmp)
* [Scenario 4 - ACTIVE-BACKUP with Connection Sync](https://github.com/loxilb-io/loxilbdocs/edit/main/docs/ha-deploy.md#scenario-4----active-backup-with-connection-sync)

## Scenario 1 -  Flat L2 Networking (active-backup)

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L2-1.drawio.svg)

Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet.
In this scenario, loxilb will be deployed as a DaemonSet in all the master nodes. 
And, kube-loxilb will be deployed as Deployment. 

### Ideal for use when
  1. Clients and end-points need to be in same-subnet.
  2. Clients and svc VIP need to be in same-subnet  (end-points may be in different networks).
  3. Simpler to deploy.
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR from local subnet.
  * Choose SetRoles option so it can choose active loxilb pod.
  * Monitors loxilb's health and elect new master on failover.
  * Sets up loxilb in one-arm svc mode towards end-points.
    
#### Configuration options
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

  *  <b>"--externalCIDR=192.168.80.200/24" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are in the same subnet.
  *  <b>"--setRoles=0.0.0.0" -</b> This option will enable kube-loxilb to choose active-backup amongst the loxilb instance and the svc IP to be configured on the active loxilb node.
  *  <b>"--setLBMode=1" -</b> This option will enable kube-loxilb to configure svc in one-arm mode towards the endpoints.

### Roles and Responsiblities for loxilb:

  * Tracks and steers the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured.

#### Configuration options

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

  * <b>"--egr-hooks" -</b> required for those cases in which workloads can be scheduled in the master nodes. No need to mention this argument when you are managing the workload scheduling to worker nodes.

  * <b>"--blacklist=cni[0-9a-z]|veth.|flannel." -</b> mandatory for running in in-cluster mode. As loxilb attaches it's ebpf programs on all the interfaces but since we running it in the default namespace then all the interfaces including CNI interfaces will be exposed and loxilb will attach it's ebpf program in those interfaces which is definitely not desired. So, user needs to mention a regex for  excluding all those interfaces. The regex in the given example will exclude the flannel interfaces. "--blacklist=cali.|tunl.|vxlan[.]calico|veth.|cni[0-9a-z]" regex must be used with calico CNI.

### Failover

This diagram describes the failover scenario:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L2-2.drawio.svg)

kube-loxilb actively monitors loxilb's health. In case of failure, it detects change in state of loxilb and assigns new “active” from available healthy loxilb pod pool.
The new pod inherits svcIP assigned previously to other loxilb pod and Services are served by newly active loxilb pod.


## Scenario 2 -  L3 network (active-backup mode using BGP)

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L3-1.drawio.svg)

Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet. SVCs will have an external IP, not from the cluster/local subnet.
In this scenario, loxilb will be deployed as a DaemonSet in all the master nodes. 
And, kube-loxilb will be deployed as Deployment. 

### Ideal for use when
  1. Clients and Cluster is in different subnet.
  2. Clients and svc VIP need to be in different subnet  (end-points may be in different networks).
  3. Ideal for cloud deployments.
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR from a different subnet.
  * Choose SetRoles option so it can choose active loxilb pod.
  * Monitors loxilb's health and elect new master on failover.
  * Automates provisioning of bgp-peering between loxilb pods.
  * Sets up loxilb in one-arm svc mode towards end-points.

#### Configuration options
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

  *  <b>"--externalCIDR=123.123.123.1/24" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are all in the different subnet.
  *  <b>"--setRoles=0.0.0.0" -</b> This option will enable kube-loxilb to choose active-backup amongst the loxilb instance and the svc IP to be configured on the active loxilb node.
  *  <b>"--setLBMode=1" -</b> This option will enable kube-loxilb to configure svc in one-arm mode towards the endpoints.
  *  <b>"--setBGP=65100" -</b> This option will let kube-loxilb to configure local AS number in the bgp instance.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> This option will configure the bgp instance's external neighbors

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP as per the state(active or backup).
  * Tracks and steers the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured.

#### Configuration options

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

  * <b>"--bgp" -</b> option enables loxilb to run with goBGP instance which will advertise the routes with appropriate preference as per active/backup state.
  
  * <b>"--egr-hooks" -</b> required for those cases in which workloads can be scheduled in the master nodes. No need to mention this argument when you are managing the workload scheduling to worker nodes.

  * <b>"--blacklist=cni[0-9a-z]|veth.|flannel." -</b> mandatory for running in in-cluster mode. As loxilb attaches it's ebpf programs on all the interfaces but since we running it in the default namespace then all the interfaces including CNI interfaces will be exposed and loxilb will attach it's ebpf program in those interfaces which is definitely not desired. So, user needs to mention a regex for  excluding all those interfaces. The regex in the given example will exclude the flannel interfaces. "--blacklist=cali.|tunl.|vxlan[.]calico|veth.|cni[0-9a-z]" regex must be used with calico CNI.

### Failover

This diagram describes the failover scenario:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L3-2.drawio.svg)

kube-loxilb actively monitors loxilb's health. In case of failure, it detects change in state of loxilb and assigns new “active” from available healthy loxilb pod pool.
The new pod inherits svcIP assigned previously to other loxilb pod and advertises the SVC IP with the preference as per the new state. Client receives the new route to SVCIP and the Services are served by newly active loxilb pod.


## Scenario 3 -  L3 network (active-active with BGP ECMP)

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L3-ECMP.drawio.svg)

Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet. SVCs will have an external IP, not from the cluster/local subnet.
In this scenario, loxilb will be deployed as a DaemonSet in all the master nodes. 
And, kube-loxilb will be deployed as Deployment. 

### Ideal for use when
  1. Clients and Cluster is in different subnet.
  2. Clients and svc VIP need to be in different subnet  (end-points may be in different networks).
  3. Ideal for cloud deployments.
  4. Better performance is desired but network devices/hosts must be capable of supporting ecmp.
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR from a different subnet.
  * <b>Do not choose SetRoles option</b> in this case (svcIPs will be advertised with same attributes/prio/med).
  * Automates provisioning of bgp-peering between loxilb pods.
  * Sets up loxilb in one-arm svc mode towards end-points.

#### Configuration options
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

  *  <b>"--externalCIDR=123.123.123.1/24" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are all in the different subnet.
  *  <b>"--setLBMode=1" -</b> This option will enable kube-loxilb to configure svc in one-arm mode towards the endpoints.
  *  <b>"--setBGP=65100" -</b> This option will let kube-loxilb to configure local AS number in the bgp instance.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> This option will configure the bgp instance's external neighbors

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP with same attributes.
  * Tracks and steers the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured.

#### Configuration options

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

  * <b>"--bgp" -</b> option enables loxilb to run with goBGP instance which will advertise the routes with same attributes.
  
  * <b>"--egr-hooks" -</b> required for those cases in which workloads can be scheduled in the master nodes. No need to mention this argument when you are managing the workload scheduling to worker nodes.

  * <b>"--blacklist=cni[0-9a-z]|veth.|flannel." -</b> mandatory for running in in-cluster mode. As loxilb attaches it's ebpf programs on all the interfaces but since we running it in the default namespace then all the interfaces including CNI interfaces will be exposed and loxilb will attach it's ebpf program in those interfaces which is definitely not desired. So, user needs to mention a regex for  excluding all those interfaces. The regex in the given example will exclude the flannel interfaces. "--blacklist=cali.|tunl.|vxlan[.]calico|veth.|cni[0-9a-z]" regex must be used with calico CNI.

### Failover

This diagram describes the failover scenario:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L3-ECMP-2.drawio.svg)

In case of failure, BGP running on the client will update the ECMP route and start sending the traffic to the active ECMP endpoints.

## Scenario 4 -  ACTIVE-BACKUP with Connection Sync

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-CT-Sync-1.drawio.svg)

This feature is only supported when loxilb runs externally outside the Kubernetes cluster in either default or fullnat mode. Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet. SVCs will have an external IP. 

There are few possible scenarios which depends upon the connectivity of External Client, loxilb and the Kubernetes cluster. For this scenario, we are here considering L3 connectivity.


### Ideal for use when
  1. Clients and Cluster is in different subnet.
  2. Clients and svc VIP need to be in different subnet  (end-points may be in different networks).
  3. Ideal for cloud deployments.
  4. Better performance is desired but network devices/hosts must be capable of supporting ecmp.
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR from a different subnet.
  * Choose SetRoles option so it can choose active loxilb (svcIPs will be advertised with different attributes/prio/med).
  * Automates provisioning of bgp-peering between loxilb containers.
  * Sets up loxilb in fullnat svc mode towards end-points.

#### Configuration options
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
        - --setLBMode=2
        - --setBGP=65100
        - --extBGPPeers=50.50.50.1:65101
```

  *  <b>"--externalCIDR=123.123.123.1/24" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are all in the different subnet.
  *  <b>"--setLBMode=2" -</b> This option will enable kube-loxilb to configure svc in fullnat mode towards the endpoints.
  *  <b>"--setBGP=65100" -</b> This option will let kube-loxilb to configure local AS number in the bgp instance.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> This option will configure the bgp instance's external neighbors

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP as per the state(active/backup).
  * Tracks and steers the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured.
  * Syncs the long-lived connections to all other configured loxilb peers.

#### Running options

```
#llb1
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest  --cluster=$llb2IP --self=0

#llb2
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest --cluster=$llb1IP --self=1
```

  * <b>"--cluster=\<llb-peer-IP\>" -</b> option configures the peer loxilb IP for syncing.
  * <b>"--self=0/1" -</b> option to identify the instance.
  

### Failover

This diagram describes the failover scenario:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-CT-Sync-2.drawio.svg)

In case of failure, kube-loxilb will detect the faailure. It will select a new loxilb from the pool of active loxilbs and update it's state to new master. New master loxilb will advertise the svcIPs with higher proference which will force the BGP running on the client to send the traffic towards new Master loxilb. Since, the connections are all synced up, new master loxilb will start sending the traffic to the designated endpoints.

Please read this detailed blog about ["Hitless HA"](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing) to know about this feature.
