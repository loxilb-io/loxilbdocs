## Quick Start Guide with MicroK8s and LoxiLB in-cluster mode

This document will explain how to install a [MicroK8s](https://microk8s.io/) cluster with loxilb as a serviceLB provider running in-cluster mode.     

### Prerequisite(s)

* Single node with Linux   

### Topology   

For quickly bringing up loxilb in-cluster and MicroK8s, we will be deploying all components in a single node :   

![loxilb topology](photos/loxilb-incluster.png)

loxilb and kube-loxilb components run as pods managed by kubernetes(MicroK8s) in this scenario.

### Setup MicroK8s in a single-node
```
# MicroK8s installation steps
sudo apt-get update
sudo apt install -y snapd
sudo snap install microk8s --classic --channel=1.28/stable
```

### Check MicroK8s status
```
$ sudo microk8s status --wait-ready
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                # (core) CoreDNS
    ha-cluster         # (core) Configure high availability on the current node
    helm               # (core) Helm - the package manager for Kubernetes
    helm3              # (core) Helm 3 - the package manager for Kubernetes
  disabled:
    cert-manager       # (core) Cloud native certificate management
    cis-hardening      # (core) Apply CIS K8s hardening
    community          # (core) The community addons repository
    dashboard          # (core) The Kubernetes dashboard
    gpu                # (core) Automatic enablement of Nvidia CUDA
    host-access        # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage   # (core) Storage class; allocates storage from host directory
    ingress            # (core) Ingress controller for external access
    kube-ovn           # (core) An advanced network fabric for Kubernetes
    mayastor           # (core) OpenEBS MayaStor
    metrics-server     # (core) K8s Metrics Server for API access to service metrics
    minio              # (core) MinIO object storage
    observability      # (core) A lightweight observability stack for logs, traces and metrics
    prometheus         # (core) Prometheus operator for monitoring and logging
    rbac               # (core) Role-Based Access Control for authorisation
    registry           # (core) Private image registry exposed on localhost:32000
    rook-ceph          # (core) Distributed Ceph storage using Rook
    storage            # (core) Alias to hostpath-storage add-on, deprecated
```

### How to deploy loxilb ?
loxilb can be deloyed by using the following command in the MicroK8s node
```
sudo microk8s kubectl apply -f https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/microk8s-incluster/loxilb.yml
```

### How to deploy kube-loxilb ?
[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is used as an operator to manage loxilb.
```
wget https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/microk8s-incluster/kube-loxilb.yml
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
sudo microk8s kubectl apply -f kube-loxilb.yaml
```

### Create the service
```
sudo microk8s kubectl apply -f https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/microk8s-incluster/tcp-svc-lb.yml
```

### Check status of various components in MicroK8s node  
In MicroK8s node:
```
## Check the pods created
$ sudo microk8s kubectl get pods -A
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   calico-node-fjfvz                        1/1     Running   0          10m
kube-system   coredns-864597b5fd-xtmt4                 1/1     Running   0          10m
kube-system   calico-kube-controllers-77bd7c5b-4kldr   1/1     Running   0          10m
kube-system   loxilb-lb-7xctp                          1/1     Running   0          9m11s
kube-system   kube-loxilb-6f44cdcdf5-4864j             1/1     Running   0          7m40s
default       tcp-onearm-test                          1/1     Running   0          6m49s

## Check the services created
$ sudo microk8s kubectl get svc
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP          PORT(S)           AGE
kubernetes      ClusterIP      10.152.183.1     <none>               443/TCP           18m
tcp-lb-onearm   LoadBalancer   10.152.183.216   llb-192.168.82.100   56002:32186/TCP   14m
```
In loxilb pod, we can check internal LB rules:
```
$ sudo microk8s kubectl exec -it -n kube-system loxilb-lb-7xctp -- loxicmd get lb -o wide
|     EXT IP     | SEC IPS | PORT  | PROTO |         NAME          | MARK | SEL |  MODE  | ENDPOINT  | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|-------|-------|-----------------------|------|-----|--------|-----------|-------|--------|--------|----------|
| 192.168.82.100 |         | 56002 | tcp   | default_tcp-lb-onearm |    0 | rr  | onearm | 10.0.2.15 | 32186 |      1 | active | 25:1842  |
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
$ cd cicd/microk8s-incluster/

# To setup the single node microk8s setup and loxilb in-cluster
$ ./config.sh

# To validate the results
$ ./validation.sh

# To login to the node and check the installation
$ vagrant ssh k8s-node1

# Cleanup
$ ./rmconfig.sh
```
