## Quick Start Guide with Redhat OCP 4.16 and LoxiLB in-cluster mode

This guide will explain how to:

* Deploy a single-node Redhat OCP 4.16 cluster
* Run loxilb in-cluster mode
* Expose services with loxilb as an external load balancer   

### Pre-requisite

* Single node to run Redhat OCP 4.16 (or later)
* Bastion or Host node to consume the services running in same network subnet as OCP node

### Topology   

For quickly bringing up loxilb with OCP, we will be deploying all components in a single node :   

![image](https://github.com/user-attachments/assets/6b6ff783-33fa-498a-974a-461bb9090b08)

All components of loxilb run as part of the OCP cluster in this scenario. loxilb can be used with various [HA](https://docs.loxilb.io/latest/ha-deploy/) options as well, but skipped here for simplicity.   

## Setup Redhat OpenShift Container Platform (OCP)  4.16

Users can follow the official [documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/installing/installing-on-a-single-node#preparing-to-install-snol) to install a single node OCP cluster

### Deploying loxilb in OCP

Use the following command to get the loxilb manifest:
```
wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/in-cluster/loxilb-nobgp.yaml
```

Change the contents of loxilb.yaml as following:
```
command: [ "/root/loxilb-io/loxilb/loxilb", "--egr-hooks", "--whitelist=enp0s3" ]
```
As one could guess, this is the name of system interface. This should be the interface which is associated with br-ex of OCP node. To confirm one can open the OCP web console and check under "Networking->NodeNetworkState" :

![image](https://github.com/user-attachments/assets/567e6871-b61f-4c42-ad6d-31fc0882e7b0)


Now apply the loxilb manifest file :
```
oc apply -f loxilb-nobgp.yaml
```

### Deploying kube-loxilb in OCP
[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is used as loxilb's operator with Kubernetes. Get the manifest file :
```
wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/in-cluster/kube-loxilb.yaml
```

Change the contents of kube-loxilb.yaml "args" to the following (leaving others commented) : 
```
        args:
          #- --loxiURL=http://192.168.80.10:11111
          - --cidrPools=defaultPool=192.168.80.100/32
          - --setRoles=0.0.0.0
          #- --setBGP=64512
          #- --listenBGPPort=1791
          #- --monitor
          #- --extBGPPeers=50.50.50.1:65101,51.51.51.1:65102
          #- --setLBMode=1
```
The CIDR is set to any unused IP/subnet in the local network.

Apply the yaml contents :
```
oc apply -f kube-loxilb.yaml
```

## Spawn a test pod and create a LB service
```
cat << EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: tcp-lb-onearm
  annotations:
    loxilb.io/liveness: "yes"
    loxilb.io/lbmode: "onearm"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: tcp-onearm-test
  ports:
    - port: 55002
      targetPort: 80 
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: tcp-onearm-test
  labels:
    what: tcp-onearm-test
spec:
  containers:
    - name: tcp-onearm-test
      image: ghcr.io/loxilb-io/nginx:stable
      ports:
        - containerPort: 80
EOF
```
For detailed explanation of various annotations, please check [this](https://docs.loxilb.io/latest/kube-loxilb/#how-to-deploy-kube-loxilb) guide.

## Check the status
### Check Kubernetes services status :
```
oc get svc
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP          PORT(S)           AGE
kubernetes      ClusterIP      10.245.0.1    <none>               443/TCP           31m
tcp-lb-onearm   LoadBalancer   10.245.2.83   llb-192.168.80.100   55002:32057/TCP   45s
```
### Check create LB rules in loxilb pod :
```
$ oc get pods -A |grep loxilb-lb
kube-system    loxilb-lb-sws2g                  1/1     Running   0          4m14s

$ oc exec -it -n kube-system loxilb-lb-sws2g  -- loxicmd get lb -o wide
|     EXT IP     | SEC IPS | HOST | PORT  | PROTO |         NAME          | MARK | SEL |  MODE  |    ENDPOINT    | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|------|-------|-------|-----------------------|------|-----|--------|----------------|-------|--------|--------|----------|
| 192.168.80.100 |         |      | 55002 | tcp   | default_tcp-lb-onearm |    0 | rr  | onearm | 192.168.80.250 | 31640 |      1 | active | 11:818   |
```

## Connect from host/client
```
$ curl http://192.168.80.100:55002
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
