## What is kube-loxilb ?

[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is loxilb's implementation of kubernetes service load-balancer spec which includes support for load-balancer class, advanced IPAM (shared or exclusive) etc. kube-loxilb runs as a deloyment set in kube-system namespace. This component runs inside k8s cluster to gather information about k8s nodes/reachability/LB services etc but in itself does not implement packet/session load-balancing. It is done by [loxilb](https://github.com/loxilb-io/loxilb) which usually runs outside the cluster as an external-LB. 

Many users frequently ask us whether it is possible to run the actual packet/session load-balancing inside the cluster (in worker-nodes or master-nodes). The answer is "yes" but it is not our preferred way. So, there is a lack of documentation regarding this. The preferred way is to run <b>kube-loxilb</b> component inside the cluster and provision <b>loxilb</b> docker in any external node/vm as mentioned in this guide. The rationale is to provide users a similar look and feel to public-cloud environments which run external load-balancers/firewalls to provide a seamless and safe environment for the cluster workloads.

## How is kube-loxilb different from loxi-ccm ?

Another loxilb component known as [loxi-ccm](https://github.com/loxilb-io/loxi-ccm) also provides implementation of kubernetes load-balancer spec but it runs as a part of cloud-provider and provides load-balancer life-cycle management as part of it. If one needs to integrate loxilb with their existing cloud-provider implementation, they can use or include loxi-ccm as a part of it. Else, kube-loxilb is the right component to use for all scenarios. It also has the latest loxilb features integrated as development is currently focused on it.   

kube-loxilb is a standalone implementation of kubernetes load-balancer spec which does not depend on cloud-provider. It runs as a kube-system deployment and provisions load-balancer rules in loxilb based on load-balancer class. It only acts on load-balancers services for the LB classes that is provided by itself. This also allows us to have different load-balancers working together in the same K8s environment. In future, loxi-ccm and kube-loxilb will share the same code base but currently they are maintained separately.   

## Overall topology   

The overall topology including all components should be similar to the following :

![loxilb topology](photos/kube-loxilb.png)   

## How to use kube-loxilb ?

1.Make sure loxilb docker is downloaded and installed properly in a node external to your cluster. One can follow guides [here](https://loxilb-io.github.io/loxilbdocs/run/) or refer to various other [documentation](https://loxilb-io.github.io/loxilbdocs/#how-to-guides) . It is important to have network connectivity from this node to the master nodes of k8s cluster (where kube-loxilb will eventually run) as seen in the above figure.

2.Download the loxilb config yaml :

```
wget https://github.com/loxilb-io/kube-loxilb/raw/main/manifest/kube-loxilb.yaml
```

3.Modify arguments as per user's needs :
```
args:
        - --loxiURL=http://192.168.20.2:11111
        - --externalCIDR=123.123.123.1/24
        - --externalCIDR6=3ffe::1/96
        #- --monitor
        #- --setBGP=false
        #- --setLBMode=1
        #- --setUniqueIP=false
        #- --externalSecondaryCIDRs=124.124.124.1/24,125.125.125.1/24
```

The arguments have the following meaning :    
- loxiURL : API server address of loxilb. This is the mgmt IP address of loxilb docker of Step 1. (Can be a comma separated list got multiple loxilb instances)     
- externalCIDR : CIDR or IPAddress range to allocate addresses from. By default address allocated are shared for different services(shared Mode)    
- externalCIDR6 : Ipv6 CIDR or IPAddress range to allocate addresses from. By default address allocated are shared for different services(shared Mode)    
- monitor : Enable liveness probe for the LB end-points (default : unset)    
- setBGP : Use BGP to advertise this service (default :false). Please check [here](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/integrate_bgp_eng.md) how it works.    
- setLBMode : 0, 1, 2   
  0 - default (only DNAT, preserves source-IP)       
  1 - onearm (source IP is changed to load balancer’s interface IP)     
  2 - fullNAT (sourceIP is changed to virtual IP)    
- setUniqueIP : Allocate unique service-IP per LB service (default : false)   
- externalSecondaryCIDRs: Secondary CIDR or IPAddress ranges to allocate addresses from in case of multi-homing support    

Many of the above flags and arguments can be overriden on a per-service basis based on loxilb specific annotation as mentioned in section 6 below.      

4. Apply the following :
```
kubectl apply -f kube-loxilb.yaml
```

5. The above should make sure kube-loxilb is successfully running. Check kube-loxilb is running :

```
kubectl get pods -A | grep kube-loxilb
```


6. Finally to create service LB, we can use and apply the following template yaml    
(<b>Note</b> -  Check <b>*loadBalancerClass*</b> and other <b>*loxilb*</b> specific annotation) :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: iperf-service
  annotations:
   # If there is a need to do liveness check from loxilb
   loxilb.io/liveness: "yes"
   # Specify LB mode - one of default, onearm or fullnat 
   loxilb.io/lbmode: "default"
   # Specify loxilb IPAM mode - one of ipv4, ipv6 or ipv6to4 
   loxilb.io/ipam: "ipv4"
   # Specify number of secondary networks for multi-homing
   # Only valid for SCTP currently
   # loxilb.io/num-secondary-networks: "2
spec:
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: perf-test
  ports:
    - port: 55001
      targetPort: 5001
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: iperf1
  labels:
    what: perf-test
spec:
  containers:
    - name: iperf
      image: eyes852/ubuntu-iperf-test:0.5
      command:
        - iperf
        - "-s"
      ports:
        - containerPort: 5001
```
Users can change the above as per their needs.

7. Verify LB service is created
```
kubectl get svc
```

For more example yaml templates, kindly refer to kube-loxilb's manifest [directory](https://github.com/loxilb-io/kube-loxilb/tree/main/manifest)   





