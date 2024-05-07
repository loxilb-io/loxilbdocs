## Quick Start Guide -  K3s with LoxiLB "service-proxy" 

This document will explain how to install a K3s cluster with [loxilb](https://github.com/loxilb-io/loxilb) in "service-proxy" mode alongside flannel networking (default for k3s). 

### What is service-proxy mode?
<b>service-proxy</b> mode is where kubernetes kube-proxy services are entirely replaced by loxilb for better performance. Users can continue to use their existing networking providers while enjoying streamlined performance and superior feature-set provided by loxilb. 

![service-proxy](photos/service-proxy.svg)   

Looking at the left side of the image, you will notice the traffic flow of the packet as it enters the Kubernetes cluster. Kube-proxy, the de-facto networking agent in the Kubernetes which runs on each node of the cluster which monitors the services and translates them to either iptables or IPVS tangible rules. If we talk about the functionality or a cluster with low volume traffic then kube-proxy is fine but when it comes to scalability or a high volume traffic then it acts as a bottle-neck.
loxilb <b><i>"service-proxy"</i></b> mode works with Flannel/Calico and kube-proxy in IPVS mode only as of now. It inherits the IPVS rules and imports these in it's in-kernel eBPF implementation. Traffic will reach at the interface, will be processed by eBPF and sent directly to the pod or to the other node, bypassing all the layers of Linux networking. This way, all the services, be it External, NodePort or ClusterIP, can be managed through LoxiLB and provide optimal performance for the users. The added benefit for the user's is the fact that there is no need to rip and replace their current networking provider (e.g flannel or calico).

### Topology   

For quickly bringing up loxilb "service-proxy" in K3s, we will be deploying a single node k3s cluster (v1.29.3+k3s1) :   

![loxilb topology](photos/service-proxy-topo.png)

loxilb and kube-loxilb components run as pods managed by kubernetes in this scenario.

## Setup K3s
### Configure K3s node
```
$ curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik \
  --disable servicelb --disable-cloud-controller --kube-proxy-arg proxy-mode=ipvs \
  cloud-provider=external --node-ip=${MASTER_IP} --node-external-ip=${MASTER_IP} \
  --bind-address=${MASTER_IP}" sh -
```
### Deploy kube-loxilb and loxilb ?
[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is used as an operator to manage loxilb. We need to deploy both kube-loxilb and loxilb components in your kubernetes cluster
```
sudo kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/main/manifest/service-proxy/kube-loxilb.yml
sudo kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/main/manifest/service-proxy/loxilb-service-proxy.yml
```

### Check the status
In k3s node:
```
## Check the pods created
$ sudo kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-6799fbcd5-c68ws                   1/1     Running   0          15m
kube-system   local-path-provisioner-6c86858495-rxk2w   1/1     Running   0          15m
kube-system   metrics-server-54fd9b65b-xtgk2            1/1     Running   0          15m
kube-system   loxilb-lb-5p6pg                           1/1     Running   0          6m58s
kube-system   kube-loxilb-5fb5566999-7xdkk              1/1     Running   0          6m59s
```
In loxilb pod, we can check internal LB rules:
```
$ udo kubectl exec -it -n kube-system loxilb-lb-5p6pg  -- loxicmd get lb -o wide
|     EXT IP     | SEC IPS | PORT  | PROTO |             NAME              | MARK | SEL |  MODE   |    ENDPOINT    | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|-------|-------|-------------------------------|------|-----|---------|----------------|-------|--------|--------|----------|
| 10.0.2.15      |         | 31377 | tcp   | ipvs_10.0.2.15:31377-tcp      |    0 | rr  | fullnat | 10.42.1.2      |  5001 |      1 | active | 0:0      |
| 10.42.1.0      |         | 31377 | tcp   | ipvs_10.42.1.0:31377-tcp      |    0 | rr  | fullnat | 10.42.1.2      |  5001 |      1 | active | 0:0      |
| 10.42.1.1      |         | 31377 | tcp   | ipvs_10.42.1.1:31377-tcp      |    0 | rr  | fullnat | 10.42.1.2      |  5001 |      1 | active | 0:0      |
| 10.43.0.10     |         |    53 | tcp   | ipvs_10.43.0.10:53-tcp        |    0 | rr  | default | 10.42.0.3      |    53 |      1 | -      | 0:0      |
| 10.43.0.10     |         |    53 | udp   | ipvs_10.43.0.10:53-udp        |    0 | rr  | default | 10.42.0.3      |    53 |      1 | -      | 0:0      |
| 10.43.0.10     |         |  9153 | tcp   | ipvs_10.43.0.10:9153-tcp      |    0 | rr  | default | 10.42.0.3      |  9153 |      1 | -      | 0:0      |
| 10.43.0.1      |         |   443 | tcp   | ipvs_10.43.0.1:443-tcp        |    0 | rr  | default | 192.168.80.10  |  6443 |      1 | -      | 0:0      |
| 10.43.202.90   |         | 55001 | tcp   | ipvs_10.43.202.90:55001-tcp   |    0 | rr  | default | 10.42.1.2      |  5001 |      1 | -      | 0:0      |
| 10.43.30.93    |         |   443 | tcp   | ipvs_10.43.30.93:443-tcp      |    0 | rr  | default | 10.42.0.4      | 10250 |      1 | -      | 0:0      |
| 192.168.80.101 |         | 31377 | tcp   | ipvs_192.168.80.101:31377-tcp |    0 | rr  | fullnat | 10.42.1.2      |  5001 |      1 | active | 15:1014  |
| 192.168.80.20  |         | 55001 | tcp   | default_iperf-service         |    0 | rr  | onearm  | 192.168.80.101 | 31377 |      1 | -      | 0:0      |
```

## Deploy a sample service

To deploy a sample service, we can create service as usual in Kubernetes with few extra annotations as follows :
```
sudo kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: iperf-service
  annotations:
   loxilb.io/lbmode: "onearm" 
   loxilb.io/staticIP: "192.168.80.20"
spec:
  externalTrafficPolicy: Local
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
      image: ghcr.io/nicolaka/netshoot:latest
      command:
        - iperf
        - "-s"
      ports:
        - containerPort: 5001
EOF
```

Check the service created :
```
$ sudo kubectl get svc
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP         PORT(S)           AGE
kubernetes      ClusterIP      10.43.0.1      <none>              443/TCP           17m
iperf-service   LoadBalancer   10.43.202.90   llb-192.168.80.20   55001:31377/TCP   2m34s
```

Test the service created (from a host outside the cluster) :
```
## Using service VIP
$ iperf -c 192.168.80.20 -p 55001  -i1 -t3
------------------------------------------------------------
Client connecting to 192.168.80.20, TCP port 55001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  1] local 192.168.80.80 port 55686 connected with 192.168.80.20 port 55001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-1.0000 sec   311 MBytes  2.61 Gbits/sec
[  1] 1.0000-2.0000 sec   309 MBytes  2.59 Gbits/sec
[  1] 2.0000-3.0000 sec   305 MBytes  2.56 Gbits/sec
[  1] 0.0000-3.0109 sec   926 MBytes  2.58 Gbits/sec

## Using node-port
$ iperf -c 192.168.80.101 -p 31377 -i1 -t10
------------------------------------------------------------
Client connecting to 192.168.80.101, TCP port 31377
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  1] local 192.168.80.80 port 34066 connected with 192.168.80.101 port 31377
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-1.0000 sec   792 MBytes  6.64 Gbits/sec
[  1] 1.0000-2.0000 sec   727 MBytes  6.10 Gbits/sec
[  1] 2.0000-3.0000 sec   784 MBytes  6.57 Gbits/sec
[  1] 3.0000-4.0000 sec   814 MBytes  6.83 Gbits/sec
[  1] 4.0000-5.0000 sec  1.01 GBytes  8.64 Gbits/sec
[  1] 5.0000-6.0000 sec  1.02 GBytes  8.79 Gbits/sec
[  1] 6.0000-7.0000 sec  1.03 GBytes  8.84 Gbits/sec
[  1] 7.0000-8.0000 sec   814 MBytes  6.83 Gbits/sec
[  1] 8.0000-9.0000 sec   965 MBytes  8.09 Gbits/sec
[  1] 9.0000-10.0000 sec   946 MBytes  7.93 Gbits/sec
[  1] 0.0000-10.0170 sec  8.76 GBytes  7.51 Gbits/sec
```
If you are wondering why there is a performance difference between serviceLB and node-port, there is an interesting blog about it [here](https://cloudybytes.medium.com/kubernetes-services-achieving-optimal-performance-is-elusive-5def5183c281) by one our users. For more detailed performance comparison with other solutions, kindly follow this [blog](https://www.loxilb.io/post/loxilb-cluster-networking-elevating-k8s-networking-capabilities) and for more detailed information
on incluster deployment of loxilb with bgp in a full-blown cluster, kindly follow this [blog](https://www.loxilb.io/post/k8s-nuances-of-in-cluster-external-service-lb-with-loxilb).   
