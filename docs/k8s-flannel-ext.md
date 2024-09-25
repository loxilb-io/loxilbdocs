## Quick Start Guide with K8s and LoxiLB ext-mode

This guide will explain how to:

* Deploy a single-node K8s cluster with flannel networking
* Run loxilb in ext-mode
* Expose services with loxilb as provider of serviceLB      

### Pre-requisite

* Two nodes with Linux - one for running K8s and another for running loxilb
* For loxilb, the linux kernel version should be >= 5.15
* Bastion or Host node to consume the services

### Topology   

For quickly bringing up loxilb with K8s, we will be deploying all components in a single node :   

![image](https://github.com/user-attachments/assets/96dd024d-7480-42be-8272-be047485aa43)

loxilb is run as a docker in its own node in external mode. loxilb can be used in more complex [in-cluster](https://www.loxilb.io/post/k8s-nuances-of-in-cluster-external-service-lb-with-loxilb) mode as well with various [HA](https://docs.loxilb.io/latest/ha-deploy/) options as well, but skipped here for simplicity.   

## Setup loxilb node

### Install docker runtime

```
apt-get update
apt-get install -y software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository -y "deb [arch=amd64] https://download.docker.com/linux/ubuntu  $(lsb_release -cs)  stable"
apt-get update
apt-get install -y docker-ce
```

### Run loxilb
```
sudo docker run -u root --cap-add SYS_ADMIN --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --net=host --name loxilb ghcr.io/loxilb-io/loxilb:latest
```

## Setup K8s node

We  will setup the single node Kubernetes setup using kubeadm. 

### Prepare the node
```
# disable swap
sudo swapoff -a

# keeps the swap off during reboot
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
sudo apt-get update -y

# Kernel Forwarding 
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

# Install docker to be used as container-runtime by Kubernetes
sudo apt install -y docker.io
sudo usermod -aG docker vagrant
```

### Install docker container runtime (or any runtime as per user preference)
```
# cri-docker Install
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
echo $VER
wget -q https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz --retry-connrefused  --waitretry=1 --read-timeout=20 --timeout=15 --tries=5 --continue
tar xvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

wget -q https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service --retry-connrefused  --waitretry=1 --read-timeout=20 --timeout=15 --tries=5 --continue
wget -q https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket --retry-connrefused  --waitretry=1 --read-timeout=20 --timeout=15 --tries=5 --continue
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket

# cri-docker Active Check
sudo systemctl restart docker && sudo systemctl restart cri-docker
sudo systemctl status cri-docker.socket --no-pager

# Docker cgroup Change Require to Systemd
sudo mkdir -p /etc/docker || true
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl restart docker && sudo systemctl restart cri-docker
```

### Install Kubernetes dependencies
```
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29.2/deb/Release.key | sudo gpg --no-tty --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29.2/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubectl kubeadm
sudo apt-get update -y
sudo apt-get install -y jq
sudo apt-get install -y ipvsadm
```
### Install Kubernetes with kubeadm

#### First create a kubeadm-config.yaml with following contents : 
(Note that 192.168.80.250 is the nodeIP of the Kubernetes node)
```
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.80.250
  bindPort: 6443
nodeRegistration:
  imagePullPolicy: IfNotPresent
  name: master
  taints: null
  kubeletExtraArgs:
    node-ip: 192.168.80.250
  criSocket: unix:///var/run/cri-dockerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
kind: ClusterConfiguration
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:
  - 192.168.80.250
controlPlaneEndpoint: 192.168.80.250:6443
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kubernetesVersion: v1.29.2
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.245.0.0/18
scheduler: {}
--
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
  qps: 5
clusterCIDR: ""
configSyncPeriod: 15m0s
#featureGates: "SupportIPVSProxyMode=true"
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```
#### Run kubeadm to complete installation: 
```
sudo kubeadm init --ignore-preflight-errors Swap --config kubeadm-config.yaml
```

#### Setup kubeconfig for running kubectl conveniently (optional)
```
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
```

### Label the node and remove taints to make sure pods are scheduled properly (since it is a single node setup)
```
kubectl label node $(hostname -s) node-role.kubernetes.io/worker=''
kubectl label node $(hostname -s) node.kubernetes.io/exclude-from-external-load-balancers-
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Deploying kube-loxilb in K8s
[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is used to deploy loxilb with Kubernetes.
```
wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/main/manifest/ext-cluster/kube-loxilb.yaml
```

Change the contents of kube-loxilb.yaml to the following (leaving others commented) : 
```
        args:
            - --loxiURL=http://192.168.80.10:11111
            - --cidrPools=defaultPool=192.168.80.100/32
            - --setRoles=0.0.0.0
```
In the above snippet, loxiURL uses nodeIP of loxilb node, which can be different for each setup. The CIDR is set to any unused IP/subnet in the local network.

Apply the yaml contents :
```
kubectl apply -f kube-loxilb.yaml
```

## Spawn a test pod and create a LB service
```
cat << EOF | kubectl apply -f -
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
### In Kubernetes node:
```
kubectl get svc
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP          PORT(S)           AGE
kubernetes      ClusterIP      10.245.0.1    <none>               443/TCP           31m
tcp-lb-onearm   LoadBalancer   10.245.2.83   llb-192.168.80.100   55002:32057/TCP   45s
```
### In loxilb node:
```
$ sudo docker exec -it loxilb loxicmd get lb -o wide
|     EXT IP     | SEC IPS | HOST | PORT  | PROTO |         NAME          | MARK | SEL |  MODE  |    ENDPOINT    | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|------|-------|-------|-----------------------|------|-----|--------|----------------|-------|--------|--------|----------|
| 192.168.80.100 |         |      | 55002 | tcp   | default_tcp-lb-onearm |    0 | rr  | onearm | 192.168.80.250 | 32057 |      1 | active | 0:0      |
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
