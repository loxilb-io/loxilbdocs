## Quick Start Guide with K0s and LoxiLB in-cluster mode

This document will explain how to install a [K0s](https://k0sproject.io/) cluster with loxilb as a serviceLB provider running in-cluster mode.     

### Prerequisite(s)

* Single node with Linux   

### Topology   

For quickly bringing up loxilb in-cluster and K0s, we will be deploying all components in a single node :   

![loxilb topology](photos/loxilb-incluster.png)

loxilb and kube-loxilb components run as pods managed by kubernetes  in this scenario.

### Setup k0s in a single-node
```
# k0s installation steps
curl -sSLf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start
```

### Check k0s status
```
$ sudo k0s status
Version: v1.29.2+k0s.0
Process ID: 2631
Workloads: true
SingleNode: true
Kube-api probing successful: true
Kube-api probing last error:  
```

### How to deploy loxilb ?
loxilb can be deloyed by using the following command in the K3s node
```
sudo k0s kubectl apply -f https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/k0s-incluster/loxilb.yml
```

### How to deploy kube-loxilb ?
[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is used as an operator to manage loxilb.
```
wget https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/k0s-incluster/kube-loxilb.yml
```
kube-loxilb.yaml
```
       args:
         #- --loxiURL=http://172.17.0.2:11111
         - --externalCIDR=192.168.82.100/32
         - --setRoles=0.0.0.0
         #- --monitor
         #- --setBGP

```
In the above snippet, loxiURL is commented out which denotes to utilize in-cluster mode to discover loxilb pods automatically. External CIDR represents the IP pool from where serviceLB VIP will be allocated.

Apply after making changes (if any) :
```
sudo k0s kubectl apply -f kube-loxilb.yaml
```

### Create the service
```
sudo k0s kubectl apply -f https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/k0s-incluster/tcp-svc-lb.yml
```

### Check status of various components in k0s node  
In k0s node:
```
## Check the pods created
$ sudo k0s kubectl get pods -A
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   kube-proxy-vczxm                  1/1     Running   0          4m48s
kube-system   kube-router-gjp7g                 1/1     Running   0          4m48s
kube-system   metrics-server-7556957bb7-25hsk   1/1     Running   0          4m50s
kube-system   coredns-6cd46fb86c-xllg2          1/1     Running   0          4m50s
kube-system   loxilb-lb-4fmdp                   1/1     Running   0          3m43s
kube-system   kube-loxilb-6f44cdcdf5-ffdcv      1/1     Running   0          2m22s
default       tcp-onearm-test                   1/1     Running   0          92s


## Check the services created
$ sudo k0s kubectl get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP          PORT(S)           AGE
kubernetes      ClusterIP      10.96.0.1       <none>               443/TCP           5m28s
tcp-lb-onearm   LoadBalancer   10.96.108.109   llb-192.168.82.100   56002:32033/TCP   111s
```
In loxilb pod, we can check internal LB rules:
```
$ sudo k0s kubectl exec -it -n kube-system loxilb-lb-4fmdp -- loxicmd get lb -o wide
|     EXT IP     | SEC IPS | PORT  | PROTO |         NAME          | MARK | SEL |  MODE  | ENDPOINT  | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|-------|-------|-----------------------|------|-----|--------|-----------|-------|--------|--------|----------|
| 192.168.82.100 |         | 56002 | tcp   | default_tcp-lb-onearm |    0 | rr  | onearm | 10.0.2.15 | 32033 |      1 | active | 25:1842  |
```

### Connect from host/client
```
$ curl http://192.168.82.100:56002
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
For more detailed information on incluster deployment of loxilb with bgp in a full-blown cluster, kindly follow this [blog](https://www.loxilb.io/post/k8s-nuances-of-in-cluster-external-service-lb-with-loxilb).   

All of the above steps are also available as part of loxilb CICD workflow. Follow the steps below to replicate the above (please note that you will need [vagrant tool](https://developer.hashicorp.com/vagrant/docs/installation) installed to run:
```
$ git clone https://github.com/loxilb-io/loxilb.git
$ cd cicd/k0s-incluster/

# To setup the single node k0s setup with kube-router networking and loxilb as external load balancer
$ ./config.sh

# To validate the results
$ ./validation.sh

# To login to the node and check the installation
$ vagrant ssh k0s-node1

# Cleanup
$ ./rmconfig.sh
```
