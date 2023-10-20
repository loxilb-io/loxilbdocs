How to deploy loxilb with High Availability
========
This article describes different scenarios about how to deploy loxilb with High Availability.

## Scenario 1 -  Flat L2 Networking (active-backup)

### Setup
--------
For this deployment scenario, kubernetes and loxilb are setup as follows:

![setup](docs/photos/loxilb-k8s-arch-LoxiLB-HA-L2-1.drawio.svg)

Kubernetes uses a cluster with 2 Master Nodes and 2 Worker Nodes, all the nodes use the same 192.168.80.0/24 subnet.
In this scenario, loxilb will be deployed as a DaemonSet in all the master nodes. 
And, kube-loxilb will be deployed as Deployment. 

[kube-loxilb](https://loxilb-io.github.io/loxilbdocs/kube-loxilb/) is loxilb's Kubernetes LoadBalancer Spec implementation. 
It will be acting as a bridge between Kubernetes and loxilb, responsible for configuring rules in the loxilb for all the LB services created in the Kubernestes Cluster.

### Ideal for use when
  1. Clients and end-points need to be in same-subnet 
  2. Clients and svc VIP need to be in same-subnet  (end-points may be in different networks)
  3. Simpler to deploy
      
### Roles and Responsiblities for kube-loxilb: 
  * Choose CIDR from local subnet
  * Choose SetRoles option so it can choose active loxilb pod
  * Monitors loxilb's health and elect new master on failover
  * Sets up loxilb in one-arm svc mode towards end-points

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
        - --externalCIDR=192.168.80.200/32
        - --setRoles=0.0.0.0
        - --setLBMode=1
```

  *  <b>"--externalCIDR=192.168.80.200/32" -</b> The external service IP for a svc is chosen from the externalCIDR range. In this scenario, the Client, svc and cluster are in the same subnet.
  *  <b>"--setRoles=0.0.0.0" -</b> This option will enable kube-loxilb to choose active-backup amongst the loxilb instance and the svc IP to be configured on the active loxilb node.
  *  <b>"--setLBMode=1" -</b> This option will enable kube-loxilb to configure svc in one-arm mode towards the endpoints.

### Roles and Responsiblities for loxilb:

  * Tracks and steers the external traffic destined to svc to the endpoints.
  * Monitors endpoint's health and chooses active endpoints, if configured

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

![setup](docs/photos/loxilb-k8s-arch-LoxiLB-HA-L2-2.drawio.svg)

kube-loxilb actively monitors loxilb's health. In case of failure, it detects change in state of loxilb and assigns new “active” from available healthy loxilb pod pool.
The new pod inherits svcIP assigned previously to other loxilb pod and Services are served by newly active loxilb pod.  


