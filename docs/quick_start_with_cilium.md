## LoxiLB Quick Start Guide with Cilium

This document will explain how to install a K3s cluster with cilium as a CNI and loxilb as an external load balancer.

### Pre-requisite

[Install](https://docs.docker.com/engine/install/ubuntu/) docker runtime to manage loxilb.   

### Topology   

For quickly bringing up loxilb with cilium CNI, we will be deploying all components in a single node :   

![loxilb topology](photos/qs_single.png)

loxilb and cilium both uses ebpf technology for load balancing and implementing policies. So, to avoid the conflict we have to run them in separate network space. 
This is reason we are going to run loxilb in a docker and use macvlan for the incoming traffic. Also, this is to mimic a topology close to cloud-hosted k8s where LB nodes run outside a cluster.

## Install loxilb docker
```
## Set promisc mode for mac-vlan to work
sudo ifconfig eth1 promisc

sudo docker run -u root --cap-add SYS_ADMIN --restart unless-stopped --privileged --entrypoint /root/loxilb-io/loxilb/loxilb -dit -v /dev/log:/dev/log  --name loxilb ghcr.io/loxilb-io/loxilb:latest

# Create mac-vlan on top of underlying eth1 interface
sudo docker network create -d macvlan -o parent=eth1 --subnet 192.168.82.0/24   --gateway 192.168.82.1 --aux-address 'host=192.168.82.252' llbnet

# Assign mac-vlan to loxilb docker with specified IP (which will be used as LB VIP)
sudo docker network connect llbnet loxilb --ip=192.168.82.100

# Add iptables rule to allow traffic from source IP(192.168.82.1) to loxilb
sudo iptables -A DOCKER -s 192.168.82.1 -j ACCEPT

```

## Setup K3s with cilium
```
#K3s installation
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb --disable-cloud-controller  \
--flannel-backend=none \
--disable-network-policy" sh -

#Install Cilium
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
mkdir -p ~/.kube/
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
cilium install

echo $MASTER_IP > /vagrant/master-ip
sudo cp /var/lib/rancher/k3s/server/node-token /vagrant/node-token
sudo cp /etc/rancher/k3s/k3s.yaml /vagrant/k3s.yaml
sudo sed -i -e "s/127.0.0.1/${MASTER_IP}/g" /vagrant/k3s.yaml

```

## How to deploy kube-loxilb ?
[kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is used to deploy loxilb with Kubernetes.
```
wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/main/manifest/ext-cluster/kube-loxilb.yaml
```

kube-loxilb.yaml
```
        args:
            - --loxiURL=http://172.17.0.2:11111
            - --externalCIDR=192.168.82.100/32
            - --setMode=1
```

Apply in k8s:
```
kubectl apply -f kube-loxilb.yaml
```

## Create the service
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/loxilb/main/cicd/docker-k3s-cilium/tcp-svc-lb.yml
```

## Check the status
In k3s:
```
kubectl get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP          PORT(S)           AGE
kubernetes      ClusterIP      10.43.0.1       <none>               443/TCP           80m
tcp-lb-onearm   LoadBalancer   10.43.183.123   llb-192.168.82.100   56002:30001/TCP   6m50s
```
In loxilb docker:
```
$ sudo docker exec -it loxilb loxicmd get lb -o wide
|   EXT IP       | SEC IPS | PORT  | PROTO |         NAME          | MARK | SEL |  MODE  | ENDPOINT  | EPORT | WEIGHT | STATE  | COUNTERS |
|----------------|---------|-------|-------|-----------------------|------|-----|--------|-----------|-------|--------|--------|----------|
| 192.168.82.100 |         | 56002 | tcp   | default_tcp-lb-onearm |    0 | rr  | onearm | 10.0.2.15 | 30001 |      1 | active | 12:880   |
```

## Connect from client
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

All of the above steps are also available as part of loxilb CICD workflow. Follow the steps below to replicate the above:
```
$ cd cicd/docker-k3s-cilium/

# To setup the single node k3s setup with cilium as CNI and loxilb as external load balancer
$ ./config.sh

# To validate the results
$ ./validation.sh

# Cleanup
$ ./rmconfig.sh
```
