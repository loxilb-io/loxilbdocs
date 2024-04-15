## Quick Start Guide -  K3s, LoxiLB "service-proxy" and Calico 

This document will explain how to install a K3s cluster with loxilb in "service-proxy" mode alongside calico networking. 

### What is service-proxy mode?
<b>service-proxy</b> mode is where kubernetes kube-proxy services are entirely replaced by loxilb for better performance. Users can continue to use existing their existing networking providers while enjoying streamlined performance provided by loxilb. 

![service-proxy](photos/service-proxy.svg)
Looking at the left side of the image, you will notice the traffic flow of the packet as it enters the Kubernetes cluster. Kube-proxy, the de-facto networking agent in the Kubernetes which runs on each node of the cluster which monitors the services and translates them to either iptables or IPVS tangible rules. If we talk about the functionality or a cluster with low volume traffic then kube-proxy is fine but when it comes to scalability or a high volume traffic then it acts as a bottle-neck.
loxilb <b><i>"service-proxy"</i></b> mode works with Flannel/Calico and kube-proxy in IPVS mode only as of now. It inherits the IPVS rules and imports these in it's in-kernel eBPF implementation. Traffic will reach at the interface, will be processed by eBPF and sent directly to the pod or to the other node, bypassing all the layers of Linux networking. This way, all the services, be it External, NodePort or ClusterIP, can be managed through LoxiLB.

### Topology   

For quickly bringing up loxilb "service-proxy" in K3s with Calico, we will be deploying a single node k3s cluster :   

![loxilb topology](photos/service-proxy-topo.png)

loxilb and kube-loxilb components run as pods managed by kubernetes in this scenario.

## Setup K3s
### Configure K3s node
```
$ curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik \
  --disable servicelb --disable-cloud-controller --kube-proxy-arg proxy-mode=ipvs \
  cloud-provider=external --flannel-backend=none --disable-network-policy --cluster-cidr=10.42.0.0/16 \
  --node-ip=${MASTER_IP} --node-external-ip=${MASTER_IP} \
  --bind-address=${MASTER_IP}" sh -
```
### Deploy calico 

K3s uses by default flannel for networking but here we are using calico to provide the same:
```
sudo kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml
sudo kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/custom-resources.yaml
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
NAMESPACE          NAME                                      READY   STATUS    RESTARTS        AGE
tigera-operator    tigera-operator-689d868448-wwvts          1/1     Running   0               2d23h
calico-system      calico-typha-67d4484996-2cmzs             1/1     Running   0               2d23h
calico-system      calico-node-l8r8b                         1/1     Running   0               2d23h
kube-system        local-path-provisioner-6c86858495-mrtzv   1/1     Running   0               2d23h
calico-system      csi-node-driver-ssbnf                     2/2     Running   0               2d23h
calico-apiserver   calico-apiserver-7dccc79b59-txnl5         1/1     Running   0               2d10h
calico-apiserver   calico-apiserver-7dccc79b59-vk68t         1/1     Running   0               2d10h
calico-system      calico-node-glm64                         1/1     Running   0               2d23h
calico-system      calico-node-hs7pw                         1/1     Running   0               2d23h
calico-system      csi-node-driver-xqjcd                     2/2     Running   0               2d23h
calico-system      calico-typha-67d4484996-wctwv             1/1     Running   0               2d23h
kube-system        kube-loxilb-5fb5566999-4vvls              1/1     Running   0               38h
calico-system      csi-node-driver-hz87c                     2/2     Running   0               2d23h
kube-system        coredns-6799fbcd5-mhgwg                   1/1     Running   0               2d8h
calico-system      calico-kube-controllers-f5c6cdbdc-vztls   1/1     Running   0               32h
calico-system      calico-node-mjjs5                         1/1     Running   0               2d23h
calico-system      csi-node-driver-l5r75                     2/2     Running   0               2d23h
default            iperf1                                    1/1     Running   0               32h
kube-system        metrics-server-54fd9b65b-78mwr            1/1     Running   0               2d23h
kube-system        loxilb-lb-px6th                           1/1     Running   0               20h
kube-system        loxilb-lb-4grfd                           1/1     Running   0               20h
kube-system        loxilb-lb-fpt25                           1/1     Running   0               20h
kube-system        loxilb-lb-56snx                           1/1     Running   0               20h
```
In loxilb pod, we can check internal LB rules:
```
$ sudo kubectl exec -it -n kube-system loxilb-lb-px6th -- loxicmd get lb -o wide
|     EXT IP     | SEC IPS | PORT  | PROTO |             NAME              | MARK | SEL |  MODE   |    ENDPOINT     | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|-------|-------|-------------------------------|------|-----|---------|-----------------|-------|--------|--------|----------|
| 10.0.2.15      |         | 32598 | tcp   | ipvs_10.0.2.15:32598-tcp      |    0 | rr  | fullnat | 192.168.235.161 |  5001 |      1 | active | 0:0      |
| 10.43.0.10     |         |    53 | tcp   | ipvs_10.43.0.10:53-tcp        |    0 | rr  | default | 192.168.182.39  |    53 |      1 | -      | 0:0      |
| 10.43.0.10     |         |    53 | udp   | ipvs_10.43.0.10:53-udp        |    0 | rr  | default | 192.168.182.39  |    53 |      1 | -      | 6:1149   |
| 10.43.0.10     |         |  9153 | tcp   | ipvs_10.43.0.10:9153-tcp      |    0 | rr  | default | 192.168.182.39  |  9153 |      1 | -      | 0:0      |
| 10.43.0.1      |         |   443 | tcp   | ipvs_10.43.0.1:443-tcp        |    0 | rr  | default | 192.168.80.10   |  6443 |      1 | -      | 0:0      |
| 10.43.182.250  |         |   443 | tcp   | ipvs_10.43.182.250:443-tcp    |    0 | rr  | default | 192.168.182.14  |  5443 |      1 | -      | 0:0      |
|                |         |       |       |                               |      |     |         | 192.168.189.75  |  5443 |      1 | -      | 0:0      |
| 10.43.184.155  |         | 55001 | tcp   | ipvs_10.43.184.155:55001-tcp  |    0 | rr  | default | 192.168.235.161 |  5001 |      1 | -      | 0:0      |
| 10.43.78.171   |         |  5473 | tcp   | ipvs_10.43.78.171:5473-tcp    |    0 | rr  | default | 192.168.80.10   |  5473 |      1 | -      | 0:0      |
|                |         |       |       |                               |      |     |         | 192.168.80.102  |  5473 |      1 | -      | 0:0      |
| 10.43.89.40    |         |   443 | tcp   | ipvs_10.43.89.40:443-tcp      |    0 | rr  | default | 192.168.219.68  | 10250 |      1 | -      | 0:0      |
| 192.168.219.64 |         | 32598 | tcp   | ipvs_192.168.219.64:32598-tcp |    0 | rr  | fullnat | 192.168.235.161 |  5001 |      1 | active | 0:0      |
| 192.168.80.10  |         | 32598 | tcp   | ipvs_192.168.80.10:32598-tcp  |    0 | rr  | fullnat | 192.168.235.161 |  5001 |      1 | active | 0:0      |
| 192.168.80.20  |         | 32598 | tcp   | ipvs_192.168.80.20:32598-tcp  |    0 | rr  | fullnat | 192.168.235.161 |  5001 |      1 | active | 0:0      |
| 192.168.80.20  |         | 55001 | tcp   | default_iperf-service         |    0 | rr  | onearm  | 192.168.80.101  | 32598 |      1 | -      | 0:0      |
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
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)           AGE
kubernetes      ClusterIP      10.43.0.1       <none>              443/TCP           3d1h
iperf-service   LoadBalancer   10.43.131.107   llb-192.168.80.20   55001:31181/TCP   9m14s
```

Test the service created (from a host outside the cluster) :
```
## Using service VIP
$ iperf -c 192.168.80.20 -p 55001  -i1 -t3
------------------------------------------------------------
Client connecting to 192.168.80.20, TCP port 55001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  1] local 192.168.80.80 port 58936 connected with 192.168.80.20 port 55001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-1.0000 sec   282 MBytes  2.36 Gbits/sec
[  1] 1.0000-2.0000 sec   276 MBytes  2.31 Gbits/sec
[  1] 2.0000-3.0000 sec   279 MBytes  2.34 Gbits/sec

## Using node-port
$ iperf -c 192.168.80.101 -p 31181  -i1 -t10
------------------------------------------------------------
Client connecting to 192.168.80.101, TCP port 31181
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  1] local 192.168.80.80 port 43208 connected with 192.168.80.101 port 31181
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-1.0000 sec   612 MBytes  5.14 Gbits/sec
[  1] 1.0000-2.0000 sec   598 MBytes  5.02 Gbits/sec
[  1] 2.0000-3.0000 sec   617 MBytes  5.17 Gbits/sec
[  1] 3.0000-4.0000 sec   600 MBytes  5.04 Gbits/sec
[  1] 4.0000-5.0000 sec   630 MBytes  5.28 Gbits/sec
[  1] 5.0000-6.0000 sec   699 MBytes  5.86 Gbits/sec
[  1] 6.0000-7.0000 sec   682 MBytes  5.72 Gbits/sec
```

For more detailed performance comparison with other solutions, kindly follow this [blog](https://www.loxilb.io/post/loxilb-cluster-networking-elevating-k8s-networking-capabilities) and for more detailed information
on incluster deployment of loxilb with bgp in a full-blown cluster, kindly follow this [blog](https://www.loxilb.io/post/k8s-nuances-of-in-cluster-external-service-lb-with-loxilb).   
