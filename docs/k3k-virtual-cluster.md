###  Kubernetes virtual cluster setup with k3k and loxilb

This guide will provide users with step-by-step instructions to provision a Kubernetes virtual cluster, run workloads in an isolated manner, and expose them via a load balancer without sacrificing performance or security.

For background, virtual cluster is a Kubernetes-native solution that allows you to create virtual clusters within a host Kubernetes cluster. Essentially, each virtual cluster runs its own Kubernetes control plane while sharing the same worker nodes as the host cluster. This setup provides an isolated environment for each virtual cluster without the overhead of creating separate physical clusters.

This guide will use [k3k](https://github.com/rancher/k3k) (kubernetes in kubernetes) as the virtual cluster provider and [loxilb](https://github.com/loxilb-io/loxilb) as the LB solution. k3k is in beta hence this guide can be used as a template for any other virtual cluster solutions out there.

### Prerequisite
We need to have a kubernetes cluster setup already. This base cluster will host other virtual cluster inside it. Let's name it ```host cluster``` for remainder of this guide.  The state of this host cluster to start with is as follows :

```
$ kubectl get pods -A
NAMESPACE      NAME                             READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-94dd4            1/1     Running   0          3h4m
kube-system    coredns-76f75df574-drz2d         1/1     Running   0          3h4m
kube-system    coredns-76f75df574-rlr7c         1/1     Running   0          3h4m
kube-system    etcd-master                      1/1     Running   0          3h4m
kube-system    kube-apiserver-master            1/1     Running   0          3h4m
kube-system    kube-controller-manager-master   1/1     Running   0          3h4m
kube-system    kube-proxy-bxnkd                 1/1     Running   0          3h4m
kube-system    kube-scheduler-master            1/1     Running   0          3h4m

$ kubectl get nodes -A -o wide
NAME     STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master   Ready    control-plane   3h13m   v1.29.9   192.168.80.250   <none>        Ubuntu 22.04.2 LTS   5.15.0-67-generic   docker://24.0.7
```

### Topology

Overall we will deploy a topology as follows :

![loxilb-virtual-cluster](https://github.com/user-attachments/assets/34c571e9-6889-48c8-9f40-eef7f9d392a5)

The overall idea is to install a virtual cluster, run kube-loxilb and workloads in the virtual clusters while running a dedicated/isolated LB per virtual cluster in the host cluster. 

### Installing k3k

First we need to get helm tool using [guidelines](https://helm.sh/docs/intro/install/) or just use the following steps :

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

Install k3k using helm

```
$ helm repo add k3k https://rancher.github.io/k3k
$ helm install my-k3k k3k/k3k --devel
```

Check the running kubernetes pods to confirm k3k is running

```
$ kubectl get pods -A
NAMESPACE      NAME                             READY   STATUS    RESTARTS   AGE
k3k-system     my-k3k-858565699d-kn96m          1/1     Running   0          3m36s
kube-flannel   kube-flannel-ds-94dd4            1/1     Running   0          3h23m
kube-system    coredns-76f75df574-drz2d         1/1     Running   0          3h23m
kube-system    coredns-76f75df574-rlr7c         1/1     Running   0          3h23m
kube-system    etcd-master                      1/1     Running   0          3h23m
kube-system    kube-apiserver-master            1/1     Running   0          3h23m
kube-system    kube-controller-manager-master   1/1     Running   0          3h23m
kube-system    kube-proxy-bxnkd                 1/1     Running   0          3h23m
kube-system    kube-scheduler-master            1/1     Running   0          3h23m
```

To manage virtual cluster using k3k, we need to get and use k3kcli tool

```
$ wget https://github.com/rancher/k3k/releases/download/v0.2.0/k3kcli
$ chmod +x k3kcli
$ sudo mv k3kcli /usr/local/sbin/
```

Now we can create a test virtual cluster
```
$ k3kcli cluster create --name example-cluster --token test  --kubeconfig ~/.kube/config
```
Sometimes, cluster creation fails with the error :
```
E1014 07:58:19.244710       7 node_container_manager_linux.go:61] "Failed to create cgroup" err="cannot enter cgroupv2 \"/sys/fs/cgroup/kubepods\" with domain controllers -- it is in an invalid state" cgroupName=[kubepods]
E1014 07:58:19.244745       7 kubelet.go:1466] "Failed to start ContainerManager" err="cannot enter cgroupv2 \"/sys/fs/cgroup/kubepods\" with domain controllers -- it is in an invalid state"
```

This is due to the fact there is some problems with k3k not able to work with cgroup v2, so in this case one needs to disable cgroup-v2 support in the OS kernel :

```
$ sudo su
$ echo 'GRUB_CMDLINE_LINUX=systemd.unified_cgroup_hierarchy=false' > /etc/default/grub.d/cgroup.cfg
$ update-grub
$ reboot
```

Double check if the new virtual cluster is created in host cluster:

```
$ kubectl get pods -A
NAMESPACE             NAME                             READY   STATUS    RESTARTS        AGE
k3k-example-cluster   example-cluster-k3k-server-0     1/1     Running   0               2d16h
k3k-system            my-k3k-858565699d-kxh8h          1/1     Running   0               2d16h
kube-flannel          kube-flannel-ds-8rn5s            1/1     Running   0               2d16h
kube-system           coredns-76f75df574-87j4d         1/1     Running   1 (2d16h ago)   2d16h
kube-system           coredns-76f75df574-trb9m         1/1     Running   1 (2d16h ago)   2d16h
kube-system           etcd-master                      1/1     Running   3 (2d16h ago)   2d16h
kube-system           kube-apiserver-master            1/1     Running   2 (2d16h ago)   2d16h
kube-system           kube-controller-manager-master   1/1     Running   2 (37h ago)     2d16h
kube-system           kube-proxy-pn4t2                 1/1     Running   1 (2d16h ago)   2d16h
kube-system           kube-scheduler-master            1/1     Running   2 (37h ago)     2d16h
```

To manage the cluster, it is convenient to generate the kubeconfig for the created virtual cluster : 
```
$ k3kcli kubeconfig generate --name example-cluster  --config-name kubeconfig-cluster1 --kubeconfig ~/.kube/config
```

We can then use ```export KUBECONFIG``` and run commands for the new virtual cluster as follows :
```
$ export KUBECONFIG=<PATH-TO-DIR>/kubeconfig-cluster1
$ kubectl get nodes
NAME                           STATUS   ROLES                       AGE   VERSION
example-cluster-k3k-server-0   Ready    control-plane,etcd,master   37h   v1.26.1+k3s1
```
We will use this config file as a ConfigMap and mount in to loxilb later in the guide. For this we need to make sure the server address in the kubeconfig is not set to some host proxy address but pod address of the virtual cluster, which is the podIP of k3k-example-cluster pod.

```
# kubeconfig-cluster1 contents (Double check server in virtual cluster kube-config)
 server: https://10.244.0.7:6443
```

Create a config-map for this virtual cluster's kubeconfig (to be used later) :
```
$ kubectl create configmap kubeconfig-cluster1 --from-file=kubeconfig-cluster1
```

### Install multus

We will be using multus to direct incoming traffic directly to each loxilb pod handling traffic towards a particular virtual cluster. Multus can be installed by using the following command in the host cluster:

```
$ export KUBECONFIG=~/.kube/config
$ kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml  # thin deployment
```
Further usage and installation guides can be found [here](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md)

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
Apply and verify it (in host Cluster) :
```
$ kubectl apply -f multus-macvlan.yaml
$ kubectl describe network-attachment-definitions.k8s.cni.cncf.io 
Name:         macvlan1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  Creation Timestamp:  2024-10-11T14:23:32Z
  Generation:          1
  Resource Version:    13944
  UID:                 32dbd769-f89f-4035-8c21-ac351842bba2
Spec:
  Config:  { "cniVersion": "0.3.1", "type": "macvlan", "master": "eth1", "mode": "bridge", "ipam": { "type": "host-local", "ranges": [ [ { "subnet": "192.168.80.0/24", "rangeStart": "192.168.80.251", "rangeEnd": "192.168.80.254", "routes": [ { "dst": "0.0.0.0/0" } ], "gateway": "192.168.80.1" } ] ] } }
Events:    <none>
```

### Setting up kube-loxilb in each vCLuster 

We will run a kube-loxilb instance for each virtual cluster. Run kube-loxilb in the the vcluster using the following yaml (use the kubeconfig for the vcluster in kubectl) :
```
$ wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/virtual-cluster/kube-loxilb.yaml
```

Change the following line in the manifest as per the service VIP needed in this file :
```
 - --cidrPools=defaultPool=192.168.80.250/32  
```

Then we can apply this config:

```
$ export KUBECONFIG=<PATH-TO-CLUSTER>/kubeconfig-cluster1
$ kubectl apply -f kube-loxilb.yaml
```

We can check if kube-loxilb works inside the virtual cluster now :
```
$ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS      AGE
kube-system   coredns-5c6b6c5476-fm8fw                  1/1     Running     1 (37h ago)   2d16h
kube-system   helm-install-traefik-crd-6w982            0/1     Completed   0             2d16h
kube-system   helm-install-traefik-trsh7                0/1     Completed   1             2d16h
kube-system   kube-loxilb-6c9d9b75cf-8ptpp              1/1     Running     0             21s
kube-system   local-path-provisioner-5d56847996-4pc8m   1/1     Running     1 (37h ago)   2d16h
kube-system   metrics-server-7b67f64457-8v2dz           1/1     Running     1 (37h ago)   2d16h
kube-system   svclb-traefik-3f3876ec-k97dw              2/2     Running     2 (37h ago)   2d16h
kube-system   traefik-56b8c5fb5c-stdtb                  1/1     Running     1 (37h ago)   2d16h
```

In this use-case, we will use URL CRDs exposed by kube-loxilb (running in virtual cluster) for allowing dynamic registration of loxilb instances (running in host cluster). To do so we need to create the CRDs by applying the following yaml inside virtual cluster :

```
$ kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/crds/loxilb-url-crd.yaml 
customresourcedefinition.apiextensions.k8s.io/loxiurls.loxiurl.loxilb.io created
$ kubectl get crds
NAME                                    CREATED AT
addons.k3s.cattle.io                    2024-10-11T12:04:09Z
helmchartconfigs.helm.cattle.io         2024-10-11T12:04:09Z
helmcharts.helm.cattle.io               2024-10-11T12:04:09Z
ingressroutes.traefik.containo.us       2024-10-11T12:04:53Z
ingressroutetcps.traefik.containo.us    2024-10-11T12:04:53Z
ingressrouteudps.traefik.containo.us    2024-10-11T12:04:53Z
loxiurls.loxiurl.loxilb.io              2024-10-13T14:29:31Z
middlewares.traefik.containo.us         2024-10-11T12:04:53Z
middlewaretcps.traefik.containo.us      2024-10-11T12:04:53Z
serverstransports.traefik.containo.us   2024-10-11T12:04:53Z
tlsoptions.traefik.containo.us          2024-10-11T12:04:53Z
tlsstores.traefik.containo.us           2024-10-11T12:04:53Z
traefikservices.traefik.containo.us     2024-10-11T12:04:53Z
```

### Setup host cluster to run loxilb instances

Run loxilb in host cluster with the following loxilb.yaml (with any modificiations needed) :
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: loxilb-lb
  #namespace: kube-system
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
        #- key: "node-role.kubernetes.io/master"
        #operator: Exists
      - key: "node-role.kubernetes.io/control-plane"
        operator: Exists
      initContainers:
      - name: mkllb-joinurl
        command:
          - sh
          - -ec
          - |
            /usr/local/sbin/mkllb-url -a $(MY_POD_IP) -z llb -t default;
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        volumeMounts:
          - name: kubeconfig-cluster1-vol
            mountPath: /root/.kube/config
            subPath: kubeconfig-cluster1
        env:
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
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
        volumeMounts:
          - name: kubeconfig-cluster1-vol
            mountPath: /root/.kube/config
            subPath: kubeconfig-cluster1
      volumes:
      - name: kubeconfig-cluster1-vol
        configMap:
          name: kubeconfig-cluster1
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
In the above yaml file, we run an ```initContainer``` which joins this loxilb instance to the respective kube-loxilb running in the virtual cluster. Also, kindly note that we are using the annotation ```k8s.v1.cni.cncf.io/networks``` which assigns an additional interface to the loxilb pod.

We can also double check if loxilb is running in the host cluster :

```
$ export KUBECONFIG=~/.kube/config
$ kubectl apply -f loxilb.yaml 
daemonset.apps/loxilb-lb created
service/loxilb-lb-service created
$ kubectl get pods -A
NAMESPACE             NAME                             READY   STATUS    RESTARTS        AGE
default               loxilb-lb-kjq5v                  1/1     Running   0               95m
k3k-example-cluster   example-cluster-k3k-server-0     1/1     Running   0               2d17h
k3k-system            my-k3k-858565699d-kxh8h          1/1     Running   0               2d17h
kube-flannel          kube-flannel-ds-8rn5s            1/1     Running   0               2d17h
kube-system           coredns-76f75df574-87j4d         1/1     Running   1 (2d18h ago)   2d18h
kube-system           coredns-76f75df574-trb9m         1/1     Running   1 (2d18h ago)   2d18h
kube-system           etcd-master                      1/1     Running   3 (2d18h ago)   2d18h
kube-system           kube-apiserver-master            1/1     Running   2 (2d18h ago)   2d18h
kube-system           kube-controller-manager-master   1/1     Running   2 (38h ago)     2d18h
kube-system           kube-loxilb-7bc7865f76-6jm6n     1/1     Running   0               26h
kube-system           kube-multus-ds-r8qm9             1/1     Running   0               2d15h
kube-system           kube-proxy-pn4t2                 1/1     Running   1 (2d18h ago)   2d18h
kube-system           kube-scheduler-master            1/1     Running   2 (38h ago)     2d18h
```

There are many [ways](https://medium.com/@ahmetb/mastering-kubeconfig-4e447aa32c75) to deal with multiple kube config files for multi-clusters. Users can choose whatever works the best for their needs.

### Testing 

For testing, we would first create a test pod in virtual cluster and expose it via loxilb using the following yaml (save it as tcp.yaml) :

```
apiVersion: v1
kind: Service
metadata:
  name: tcp-lb
  annotations:
    loxilb.io/lbmode: "onearm"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: tcp-test
  ports:
    - port: 56002
      targetPort: 80
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: tcp-test
  labels:
    what: tcp-test
spec:
  containers:
    - name: tcp-test
      image: ghcr.io/loxilb-io/nginx:stable
      ports:
        - containerPort: 80
```

And apply it inside the virtual cluster :

```
$ export KUBECONFIG=<PATH-TO-DIR>/kubeconfig-cluster1
$ kubectl apply -f tcp.yaml 
service/tcp-lb created
pod/tcp-test created
```

We can check if services have been created as expected :
```
$ kubectl get svc -A
NAMESPACE     NAME             TYPE           CLUSTER-IP     EXTERNAL-IP          PORT(S)                      AGE
default       kubernetes       ClusterIP      10.45.0.1      <none>               443/TCP                      2d16h
default       tcp-lb           LoadBalancer   10.45.225.96   llb-192.168.80.250   56002:32145/TCP              2s
kube-system   kube-dns         ClusterIP      10.45.0.10     <none>               53/UDP,53/TCP,9153/TCP       2d16h
kube-system   metrics-server   ClusterIP      10.45.89.141   <none>               443/TCP                      2d16h
```

Finally we can check the service accessibility from outside :

```
curl http://192.168.80.250:56002
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

In this guide, we have illustrated how to deploy a single virtual cluster with a single loxilb pod. But the same set of steps (with minor variations) can be followed to created multiple virtual clusters each with its own HA pair of loxilb instances. 
