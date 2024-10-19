### Kubernetes setup with cilium and loxilb in-cluster mode

This guide will provide users with step-by-step instructions to provision a Kubernetes cluster, with cilium as the default networking and expose services via a loxilb load balancer also running as an incluster component.

Cilium has a great pod networking implementation and loxilb has a good kubernetes-native load-balancer implementation. This guide explores how to get best of the both worlds and make things work smoothly.

### Prerequisite 

This guide will not delve into details of installing a Kubernetes cluster itself and assumes that users have cluster up and ready. Interested readers can follow various [resources](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) on the internet for the same. 

### Topology

Overall we will deploy a topology as follows :

![image](https://github.com/user-attachments/assets/2c838eb2-6f2f-42a1-9f45-abe7628aad6d)

For sake of simplicilty, we will have a single node based deployment for this guide which can be used as a template for more complex scenarios.

### Installing Cilium

The steps to install cilium as as follows :

#### Install helm tool

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

#### Pull cilium helm chart

```
$ helm repo add cilium https://helm.cilium.io/
$ helm pull cilium/cilium  --version=1.16.2 --untar
```

#### Set cni-exclusive flag in cilium is set to false

```
## Change exclusive flag in values.yaml inside the downloaded chart dir "cilium":
  exclusive: false
```

The importance of setting this flag will become evident in subsequent sections.

#### Deploy cilium using helm

```
$ helm install cilium --namespace kube-system  ./cilium
```

Once the installation is done, the state of the cluster appears as below :

```
$ kubectl get pods -A
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   cilium-8rhpw                      1/1     Running   0          90s
kube-system   cilium-envoy-gk7x9                1/1     Running   0          90s
kube-system   cilium-operator-696b7f8c9-psk5z   1/1     Running   0          90s
kube-system   coredns-76f75df574-d6297          1/1     Running   0          46h
kube-system   coredns-76f75df574-vr8vp          1/1     Running   0          46h
kube-system   etcd-master                       1/1     Running   0          46h
kube-system   kube-apiserver-master             1/1     Running   0          46h
kube-system   kube-controller-manager-master    1/1     Running   0          46h
kube-system   kube-proxy-6tqkb                  1/1     Running   0          46h
kube-system   kube-scheduler-master             1/1     Running   0          46h
```

### Challenges of running different eBPF programs together

Cilium uses eBPF TC hooks for implementing much of its fuctionalilty. These hooks are usually set for exclusive use. Even if it is set to a shared mode, it would need a great deal of co-operation between eBPF programs sharing these hook. When cilium is installed we can see the hooks usage :

```
$ tc filter show dev eth1 ingress
filter protocol all pref 1 bpf chain 0 
filter protocol all pref 1 bpf chain 0 handle 0x1 cil_from_netdev-eth1 direct-action not_in_hw id 783 tag 612ae21d69e3648a 
```

loxilb also uses similar hooks in its operation and by default compatiblity of such programs running together in the same OS is not possible at this point of time.

Hence, we take the approach to completely segregrate the traffic destined to loxilb into a separate pod and use secondary interfaces for traffic injection with help from [multus](https://github.com/k8snetworkplumbingwg/multus-cni). For starters, multus can create various type of secondary interfaces and attach them to pods. It is very useful for pods which require multi-homing or multiple-interfaces to work with.

### Deploy multus

To install multus, we will use the following steps :

```
$ kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml  # thin deployment
```

Further usage and installation guides can be found here

Next, we create a macvlan attachment object with multus with the following yaml (save it as multus-macvlan.yaml) :

```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan1
spec:
  config: '{
            "cniVersion": "0.3.1",
            "type": "macvlan",
            "master": "eth1",
            "mode": "bridge",
            "ipam": {
                "type": "host-local",
                "ranges": [
                    [ {
                         "subnet": "192.168.80.0/24",
                         "rangeStart": "192.168.80.251",
                         "rangeEnd": "192.168.80.254",
                         "routes": [
                            {
                              "dst": "0.0.0.0/0"
                            }
                         ],
                         "gateway": "192.168.80.1"
                    } ]
                ]
            }
        }'
```

Apply and verify it:

```
$ kubectl apply -f multus-macvlan.yaml 
networkattachmentdefinition.k8s.cni.cncf.io/macvlan1 created
$ kubectl describe network-attachment-definitions.k8s.cni.cncf.io 
Name:         macvlan1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  Creation Timestamp:  2024-10-19T01:29:25Z
  Generation:          1
  Resource Version:    288889
  UID:                 9cadef96-aab8-45ed-8b52-9df75f22d433
Spec:
  Config:  { "cniVersion": "0.3.1", "type": "macvlan", "master": "eth1", "mode": "bridge", "ipam": { "type": "host-local", "ranges": [ [ { "subnet": "192.168.80.0/24", "rangeStart": "192.168.80.251", "rangeEnd": "192.168.80.254", "routes": [ { "dst": "0.0.0.0/0" } ], "gateway": "192.168.80.1" } ] ] } }
Events:    <none>
```

### Deploy loxilb

Deploy loxilb the cluster with the following "loxilb.yaml" :

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: loxilb-lb
spec:
  selector:
    matchLabels:
      app: loxilb-app
  template:
    metadata:
      name: loxilb-lb
      labels:
        app: loxilb-app
      annotations:
        k8s.v1.cni.cncf.io/networks: macvlan1
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: Exists
      containers:
      - name: loxilb-app
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        command: [ "/root/loxilb-io/loxilb/loxilb" ]
        ports:
        - containerPort: 11111
        - containerPort: 179
        - containerPort: 50051
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
---
apiVersion: v1
kind: Service
metadata:
  name: loxilb-lb-service
  #namespace: kube-system
spec:
  clusterIP: None
  selector:
    app: loxilb-app
  ports:
  - name: loxilb-app
    port: 11111
    targetPort: 11111
    protocol: TCP
  - name: loxilb-app-bgp
    port: 179
    targetPort: 179
    protocol: TCP
  - name: loxilb-app-gobgp
    port: 50051
    targetPort: 50051
    protocol: TCP
```

Also, kindly note that we are using the annotation k8s.v1.cni.cncf.io/networks which assigns an additional interface to the loxilb pod. If the ``cni-exclusive`` flag is not set to false during cilium's installation, cilium usually does not allow meta-plugins like mutlus to work properly.

We can now double check if loxilb is running in the  cluster :
```
$ kubectl get pods -A
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
default       loxilb-lb-wcz6p                   1/1     Running   0          115s
kube-system   cilium-8rhpw                      1/1     Running   0          7m38s
kube-system   cilium-envoy-gk7x9                1/1     Running   0          7m38s
kube-system   cilium-operator-696b7f8c9-psk5z   1/1     Running   0          7m38s
kube-system   coredns-76f75df574-d6297          1/1     Running   0          46h
kube-system   coredns-76f75df574-vr8vp          1/1     Running   0          46h
kube-system   etcd-master                       1/1     Running   0          46h
kube-system   kube-apiserver-master             1/1     Running   0          46h
kube-system   kube-controller-manager-master    1/1     Running   0          46h
kube-system   kube-multus-ds-vvd2k              1/1     Running   0          2m22s
kube-system   kube-proxy-6tqkb                  1/1     Running   0          46h
kube-system   kube-scheduler-master             1/1     Running   0          46h
```

### Deploy kube-loxilb

Now we will deploy kube-loxilb which is loxilb's operator. Get the manifest file :

```
$ wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/in-cluster/kube-loxilb-nobgp.yaml
```

Change the following line in the manifest as per the service VIP needed in this file :

```
 - --cidrPools=defaultPool=192.168.80.249/32  
```

Deploy it:

```
$ kubectl apply -f kube-loxilb-nobgp.yaml
serviceaccount/kube-loxilb created
clusterrole.rbac.authorization.k8s.io/kube-loxilb created
clusterrolebinding.rbac.authorization.k8s.io/kube-loxilb created
deployment.apps/kube-loxilb created
$ kubectl get pods -A
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
default       loxilb-lb-wcz6p                   1/1     Running   0          16m
kube-system   cilium-8rhpw                      1/1     Running   0          21m
kube-system   cilium-envoy-gk7x9                1/1     Running   0          21m
kube-system   cilium-operator-696b7f8c9-psk5z   1/1     Running   0          21m
kube-system   coredns-76f75df574-d6297          1/1     Running   0          47h
kube-system   coredns-76f75df574-vr8vp          1/1     Running   0          47h
kube-system   etcd-master                       1/1     Running   0          47h
kube-system   kube-apiserver-master             1/1     Running   0          47h
kube-system   kube-controller-manager-master    1/1     Running   0          47h
kube-system   kube-loxilb-5df69f45c5-jqq22      1/1     Running   0          8m21s
kube-system   kube-multus-ds-vvd2k              1/1     Running   0          16m
kube-system   kube-proxy-6tqkb                  1/1     Running   0          47h
kube-system   kube-scheduler-master             1/1     Running   0          47h
```

### Testing

For testing, we would first create a test pod in the cluster and expose it via loxilb using the following yaml (save it as nginx.yaml) :

```
apiVersion: v1
kind: Service
metadata:
  name: tcp-lb
  annotations:
    loxilb.io/lbmode: "onearm"
    loxilb.io/usepodnetwork : "yes"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: nginx-test
  ports:
    - port: 56002
      targetPort: 80
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  labels:
    what: nginx-test
spec:
  containers:
    - name: tcp-test
      image: ghcr.io/loxilb-io/nginx:stable
      ports:
        - containerPort: 80
```

And apply it to the cluster:

```
$ kubectl apply -f nginx.yaml 
service/tcp-lb created
pod/nginx-test created
```

We can check if services have been created as expected :

```
$ kubectl get svc
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP          PORT(S)                       AGE
kubernetes          ClusterIP      10.245.0.1      <none>               443/TCP                       47h
loxilb-lb-service   ClusterIP      None            <none>               11111/TCP,179/TCP,50051/TCP   22m
tcp-lb              LoadBalancer   10.245.23.116   llb-192.168.80.249   56002:32093/TCP               3m7s
```

Finally we can check the service accessibility from outside :

```
$ curl http://192.168.80.249:56002
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
