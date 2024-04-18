# loxilb - How to build/run

## 1. Right from code (difficult)

* Install GoLang > v1.17

```
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz && sudo tar -xzf go1.22.0.linux-amd64.tar.gz --directory /usr/local/
export PATH="${PATH}:/usr/local/go/bin"
```

* Install standard packages
```
sudo apt install -y clang llvm libelf-dev gcc-multilib libpcap-dev vim net-tools linux-tools-$(uname -r) elfutils dwarves git libbsd-dev bridge-utils wget unzip build-essential bison flex iproute2
```
* Build and run loxilb 

```
git clone --recurse-submodules https://github.com/loxilb-io/loxilb.git
cd loxilb
./loxilb-ebpf/utils/mkllb_bpffs.sh
make
cd loxilb-ebpf/libbpf/src
sudo make install
cd -
sudo mkdir -p /opt/loxilb/cert/
sudo cp api/certification/server.* /opt/loxilb/cert/
sudo ./loxilb 
```
* Build and use loxicmd 

```
git clone https://github.com/loxilb-io/loxicmd.git
cd loxicmd
go get .
make
sudo cp -f loxicmd /usr/local/sbin/
```
loxicmd usage guide can be found [here](https://loxilb-io.github.io/loxilbdocs/cmd/)

## 2. From docker (easy)

Get the loxilb official docker image    

* Latest build image (multi-arch amd64/arm64)   
```
docker pull ghcr.io/loxilb-io/loxilb:latest
```   

* Release build Image   
```
docker pull ghcr.io/loxilb-io/loxilb:v0.9.2
``` 

To run loxilb docker, we can use the following commands :

```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit --pid=host --cgroupns=host -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest
```

To drop in to a shell of loxilb doker :

```
docker exec -it loxilb bash
```

For load-balancing to effectively work in a bare-metal environment, we need multiple interfaces assigned to the docker (external and internal connectivitiy) 

  loxilb docker relies on docker's macvlan driver for achieving this. The following is an example of creating macvlan network and using with loxilb

```
# Create a mac-vlan (on an underlying interface e.g. enp0s3).
# Subnet used for mac-vlan is usually the same as underlying interface
docker network create -d macvlan -o parent=enp0s3   --subnet 172.30.1.0/24   --gateway 172.30.1.254 --aux-address 'host=172.30.1.193’ llbnet

# Run loxilb docker with the created macvlan 
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --net=llbnet --ip=172.30.1.195 --name loxilb ghcr.io/loxilb-io/loxilb:latest

# If we still want to connect loxilb docker additionally to docker's default "bridge" network or more macvlan networks
docker network connect bridge loxilb
docker network connect llbnet2 loxilb --ip=172.30.2.195
```

<b>Note:</b>    

* While working with macvlan interfaces, the parent/underlying interface should be put in promiscous mode     
* One can further use docker-compose to automate attaching multiple networks to loxilb docker or use --net=host as per requirement    
* To create a simple and self-contained topology for testing loxilb, users can follow this [guide](simple_topo.md)   
* If loxilb docker is in the same node as the app/workload docker, it is advised that "tx checksum offload" inside app/workload docker is turned off for sctp load-balancing to work properly   
```
docker exec -dt <app-docker-name> ethtool -K <app-docker-interface> tx off
```   

## 3. Running in Kubernetes   
* For running in K8s environment, kindly follow [kube-loxilb](https://loxilb-io.github.io/loxilbdocs/kube-loxilb/) guide     

