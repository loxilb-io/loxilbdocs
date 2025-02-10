# Setting up LoxiLB In-Cluster Mode and Multus in a Kubernetes Cluster Using Calico CNI

This document introduces one of the methods to deploy LoxiLB in in-cluster mode within a Kubernetes cluster using Calico and expose load balancer services externally through LoxiLB.

The default networking in Kubernetes is managed entirely by Calico. However, traffic accessing the load balancer service via an external IP is delivered to LoxiLB through a Multus network. LoxiLB then load balances the traffic and forwards it to the appropriate pods.

## **Prerequisite**

This document uses a K3s single-node setup for Kubernetes. However, this is not a requirement; K3s was simply chosen for ease of installation. The contents of this document can be applied to other Kubernetes configurations as well.

## Topology

Overall we will deploy a topology as follows :

![loxilb-k8s-arch-loxilb-calico-multus.jpg](photos/loxilb-k8s-arch-loxilb-calico-multus.drawio.svg)

## Install K3s

```
# Install IPVS
sudo apt-get -y install ipset ipvsadm

# Install K3s with Calico and kube-proxy in IPVS mode
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik,metrics-server,servicelb --disable-cloud-controller --kubelet-arg cloud-provider=external --flannel-backend=none --disable-network-policy" K3S_KUBECONFIG_MODE="644" sh -s - server --kube-proxy-arg proxy-mode=ipvs

# Remove taints in k3s if any (usually happens if started without cloud-manager)
sudo kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized=false:NoSchedule-
```

## Install Calico

There are two methods to install Calico: using an Operator or using a Manifest. In this document, we will install it using a Manifest.

```
# Download calico manifest file
wget https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml

```

Open the calico.yaml file and locate the `cni_network_config` section. Then, add the `container_settings` entry inside the `plugins` section as follows. (I referred to the following link: [K3s Basic Network Options](https://docs.k3s.io/networking/basic-network-options?_highlight=container_settings&cni=Calico#custom-cni))

```
"ipam": {
    "type": "calico-ipam"
},
"container_settings": {
    "allow_ip_forwarding": true
}
```

## Deploy multus

Deploy the Multus DaemonSet using the following command.

```
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
	
```

Then, create the `multus-macvlan.yaml` file as follows. In my case, I used `enp0s8` as the master interface for Multus Macvlan. Since the IP assigned to `enp0s8` is `192.168.219.103`, I will configure the network range for the Macvlan interface as `192.168.219.0/24`. Please adjust the IP and `master` interface according to your environment.

```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan1
spec:
  config: '{
            "cniVersion": "0.3.1",
            "type": "macvlan",
            "master": "enp0s8",
            "mode": "bridge",
            "ipam": {
                "type": "host-local",
                "ranges": [
                    [ {
                         "subnet": "192.168.219.0/24",
                         "rangeStart": "192.168.219.251",
                         "rangeEnd": "192.168.219.254",
                         "routes": [
                            {
                              "dst": "0.0.0.0/0"
                            }
                         ],
                         "gateway": "192.168.219.1"
                    } ]
                ]
            }
        }'
```

Apply it to Kubernetes using the following command.

```
kubectl apply -f multus-macvlan.yaml 
```

## Deploy LoxiLB in-cluster mode

Typically, LoxiLB uses the host’s network by enabling the `hostNetwork: true` option. However, in this document, we will remove this option to allow Calico to fully control the default Kubernetes network. Instead, we will use a Multus network to configure an external access path.

The following is the manifest file for deploying LoxiLB in in-cluster mode. We have specified annotations to use the Multus network (`macvlan1`) created earlier.

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

## deploy kube-loxilb

To integrate LoxiLB with Kubernetes, we also need to deploy `kube-loxilb`. First, download the manifest file using the following command.

```
wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/in-cluster/kube-loxilb-nobgp.yaml

```

In the `kube-loxilb-nobgp.yaml` file, locate the `cidrPools` section and modify the IP address according to your environment. The IP set here will be assigned as the external IP for the LoadBalancer service. In my case, I will configure it to allow external access via `192.168.219.249`.

```
 - --cidrPools=defaultPool=192.168.219.249/32  
```

Once the modifications are complete, deploy `kube-loxilb` using the following command.

```
kubectl apply -f kube-loxilb-nobgp.yaml
```

## Testing

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
kubectl apply -f nginx.yaml 
```

We can check if services have been created as expected :

```
kubectl get svc
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP          PORT(S)                       AGE
kubernetes          ClusterIP      10.245.0.1      <none>               443/TCP                       47h
loxilb-lb-service   ClusterIP      None            <none>               11111/TCP,179/TCP,50051/TCP   22m
tcp-lb              LoadBalancer   10.245.23.116   llb-192.168.219.249  56002:32093/TCP               3m7s
```

Finally we can check the service accessibility from outside :

```
curl http://192.168.219.249:56002
```

# Troubleshooting

## If the LoxiLB pod remains stuck in the `ContainerCreating` state when being created

```
kubectl describe pods {loxilb pod name}
```

Some common errors include the following:

### If the macvlan CNI cannot be found:

You can build the Macvlan CNI using the following command:

```
cd /tmp
mkdir -p cni-plugins
cd cni-plugins
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
tar xvfz cni-plugins-linux-amd64-v1.6.2.tgz
sudo cp macvlan /opt/cni/bin
```

### If the Created Multus Interface Cannot Be Found:

Even though the Multus interface has been created, an error message appears indicating that the pod cannot find the interface. In such cases, check whether the correct interface name is set as the master for the Multus interface.

## If an external host cannot send packets to LoxiLB's Multus interface IP

The master interface may need to be set to promiscuous mode. Enable promiscuous mode using the following command.

```
# 1. turn on promisc mode
sudo ip link set eth1 promisc on

# 2. set interface to down and up
sudo ip link set eth1 down
sudo ip link set eth1 up
```

If you are in a VM environment, you also need to enable promiscuous mode in the VM network or interface settings for it to work.
