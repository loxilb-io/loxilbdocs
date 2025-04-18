How to deploy loxilb with High Availability
========
This article describes different scenarios about how to deploy loxilb with High Availability. Before continuing to this page, all readers are advised to have a basic understanding about [kube-loxilb](https://loxilb-io.github.io/loxilbdocs/kube-loxilb/) and the different [NAT modes](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/nat.md) supported by loxilb. loxilb can run in-cluster or external to kubernetes cluster depending on architectural choices. For this documentation, we have assumed an incluster deployment wherever applicable but similar configuration should suffice for an external deployment as well.   

* [Scenario 1 -  Flat L2 Networking (active-backup)](#scenario-1----flat-l2-networking-active-backup)
* [Scenario 2 -  L3 network (active-backup mode using BGP)](#scenario-2----l3-network-active-backup-mode-using-bgp)
* [Scenario 3 -  L3 network (active-active with BGP ECMP)](#scenario-3----l3-network-active-active-with-bgp-ecmp)
* [Scenario 4 - ACTIVE-BACKUP with Connection Sync](#scenario-4----active-backup-with-connection-sync)
* [Scenario 5 - ACTIVE-BACKUP with Fast Failover Detection(BFD)](#scenario-5----active-backup-with-fast-failover-detection)

## Scenario 1 -  Flat L2 Networking (active-backup)

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L2-1.drawio.svg)

Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet. In this scenario, loxilb will be deployed as a DaemonSet in all the master nodes. And, kube-loxilb will be deployed as Deployment.    

### Ideal for use when
  1. Clients and services need to be in same-subnet.    
  2. End-points may or may not be in same subnet.      
  3. Simpler deployment is desired.   
      
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

Sample kube-loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb.yaml).

### Roles and Responsiblities for loxilb:

  * Tracks and directs the external traffic destined to svc to the endpoints.   
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

Sample loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/loxilb.yaml).

### Failover

This diagram describes the failover scenario:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L2-2.drawio.svg)

kube-loxilb actively monitors loxilb's health. In case of failure, it detects change in state of loxilb and assigns new “active” from available healthy loxilb pod pool.
The new pod inherits svcIP assigned previously to other loxilb pod and Services are served by newly active loxilb pod. 

For users, who want to distribute services across various HA sharding instances so that there is better resource utilization in a similar scenario, please check this [guide](https://docs.loxilb.io/latest/service-sharding/). This can be helpful for folks who don't want their slave pods to sit idle without doing anything.


## Scenario 2 -  L3 network (active-backup mode using BGP)

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-L3-1.drawio.svg)

Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet. SVCs will have an external IP, not from the cluster/local subnet.
In this scenario, loxilb will be deployed as a DaemonSet in all the master nodes. 
And, kube-loxilb will be deployed as Deployment. 

### Ideal for use when
  1. Clients and Cluster are in different subnets.
  2. Clients and svc VIP need to be in different subnet  (cluster end-points may also be in different networks).
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
  *  <b>"--setRoles=0.0.0.0" -</b> This option will enable kube-loxilb to choose active-backup amongst the loxilb instances and the svc IP to be configured on the active loxilb node.
  *  <b>"--setLBMode=1" -</b> This option will enable kube-loxilb to configure svc in one-arm mode towards the endpoints.
  *  <b>"--setBGP=65100" -</b> This option will let kube-loxilb to configure local AS number in the bgp instance.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> This option will configure the bgp instance's external neighbors.

Sample kube-loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb.yaml).

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP as per the state(active or backup).
  * Tracks and directs the external traffic destined to svc to the endpoints.
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

Sample loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/loxilb.yaml).

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
  1. Clients and Cluster are in different subnets.
  2. Clients and svc VIP need to be in different subnet  (cluster end-points may also be in different networks).
  3. Ideal for cloud deployments.
  4. Better performance is desired due to active-active clustering but network devices/hosts must be capable of supporting ecmp.
      
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

Sample kube-loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb.yaml).

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP with same attributes.
  * Tracks and directs the external traffic destined to svc to the endpoints.
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

Sample loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/loxilb.yaml).

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
  1. Need to preserve long running connections during lb pod failures
  2. Another LB mode known as DSR mode can be used to preserve connections but has the following limitations :
     1. Can't ensure stateful filtering and connection-tracking.
     2. Can't support multihoming features since different 5-tuples might belong to the same connection. 
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR as required.
  * Choose SetRoles option so it can choose active loxilb (svcIPs will be advertised with different attributes/prio/med).
  * Automates provisioning of bgp-peering between loxilb containers (if required).
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
        - --setRoles=0.0.0.0
        - --loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111
        - --externalCIDR=123.123.123.1/24
        - --setLBMode=2
        - --setBGP=65100
        - --extBGPPeers=50.50.50.1:65101
```

  *  <b>"--setRoles=0.0.0.0" -</b> This option will enable kube-loxilb to choose active-backup amongst the loxilb instance.
  *  <b>"--loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111" -</b> loxilb URLs to connect with.
  *  <b>"--externalCIDR=123.123.123.1/24" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are all in the different subnet.
  *  <b>"--setLBMode=2" -</b> This option will enable kube-loxilb to configure svc in fullnat mode towards the endpoints.
  *  <b>"--setBGP=65100" -</b> This option will let kube-loxilb to configure local AS number in the bgp instance.
  *  <b>"--extBGPPeers=50.50.50.1:65101" -</b> This option will configure the bgp instance's external neighbors
    
Sample kube-loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/ext-cluster/kube-loxilb.yaml).

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP as per the state(active/backup).
  * Tracks and directs the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured.
  * Syncs the long-lived connections to all other configured loxilb peers.

#### Running options

```
#llb1
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest  --cluster=$llb2IP --self=0 -b

#llb2
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest --cluster=$llb1IP --self=1 -b
```

  * <b>"--cluster=\<llb-peer-IP\>" -</b> option configures the peer loxilb IP for syncing.
  * <b>"--self=0/1" -</b> option to identify the instance.
  * <b>"-b" -</b> option enables loxilb to run with goBGP instance which will advertise the routes with appropriate preference as per active/backup state.
  

### Failover

This diagram describes the failover scenario:

![setup](photos/loxilb-k8s-arch-LoxiLB-HA-CT-Sync-2.drawio.svg)

In case of failure, kube-loxilb will detect the failure. It will select a new loxilb from the pool of active loxilbs and update it's state to new master. New master loxilb will advertise the svcIPs with higher proference which will force the BGP running on the client to send the traffic towards new Master loxilb. Since, the connections are all synced up, new master loxilb will start sending the traffic to the designated endpoints.

Please read this detailed blog about ["Hitless HA"](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing) to know about this feature.

## Scenario 5 -  ACTIVE-BACKUP with Fast Failover Detection

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](photos/bfd-1.svg)

This feature is only supported when loxilb runs externally outside the Kubernetes cluster. Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet. SVCs will have an external IP. 

There are few possible scenarios which depends upon the connectivity of External Client, loxilb and the Kubernetes cluster. For this scenario, we are here considering L2 connectivity with connection sync.


### Ideal for use when
  1. Need fast failover detection and service continuity.
  2. This feature works in both L2 or L3 network settings. 
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR as required.
  * Disable SetRoles option so it should not choose active loxilb.
  * Automates provisioning of bgp-peering between loxilb containers (if required).
  * Sets up loxilb in configured svc mode towards end-points.

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
        # Disable setRoles option
        #- --setRoles=0.0.0.0
        - --loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111
        - --externalCIDR=192.168.80.5/32
        - --setLBMode=2
```

  *  <b>"--setRoles=0.0.0.0" -</b> We have to make sure to disable this option as it will enable kube-loxilb to choose active-backup amongst the loxilb instance.
  *  <b>"--loxiURL=http://192.168.80.1:11111,http://192.168.80.2:11111" -</b> loxilb URLs to connect with.
  *  <b>"--externalCIDR=192.168.80.5/32" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are all in the same subnet.
  *  <b>"--setLBMode=2" -</b> This option will enable kube-loxilb to configure svc in fullnat mode towards the endpoints.
    
Sample kube-loxilb.yaml can be found [here](https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/ext-cluster/kube-loxilb.yaml).

### Roles and Responsiblities for loxilb:

  * Advertises SVC IP as per the state(active/backup).
  * Tracks and directs the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured.
  * Syncs the long-lived connections to all other configured loxilb peers.

#### Running options

```
#llb1
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest  --cluster=192.168.80.2 --self=0 --ka=192.168.80.2:192.168.80.1

#llb2
 docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest --cluster=192.168.80.1 --self=1 --ka=192.168.80.1:192.168.80.2
```

  * <b>"--ka=\<llb-peer-IP\>:\<llb-self-IP\>" -</b> option configures the peer loxilb IP and source IP for BFD.
  * <b>"--cluster=\<llb-peer-IP\>" -</b> option configures the peer loxilb IP for syncing.
  * <b>"--self=0/1" -</b> option to identify the instance.

### Failover

This diagram describes the failover scenario:

![setup](photos/bfd-2.svg)

In case of failure, BFD will detect the failure. BACKUP loxilb will update it's state to new master. New master loxilb will advertise the svcIPs through gARP or with higher proference, if running with BGP, which will force the client to send the traffic towards new Master loxilb. Since, the connections are all synced up, new master loxilb will start sending the traffic to the designated endpoints.

Please read this detailed blog about ["Fast Failover Detection with BFD"](https://www.loxilb.io/post/bringing-sub-second-resilience-in-kubernetes-cluster) to know about this feature.
## Note :

There are ways to use loxilb in DSR mode, DNS etc which is still not covered in details in this doc. We will keep updating the scenarios.
