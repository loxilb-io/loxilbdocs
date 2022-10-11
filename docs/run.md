# loxilb - How to build/run

## Right from code (difficult)

* Install GoLang > v1.17

```
wget https://go.dev/dl/go1.18.linux-amd64.tar.gz && tar -xzf go1.18.linux-amd64.tar.gz --directory /usr/local/
export PATH="${PATH}:/usr/local/go/bin"
```

* Install standard packages
```
apt install -y clang llvm libelf-dev gcc-multilib libpcap-dev vim net-tools linux-tools-$(uname -r) elfutils dwarves git libbsd-dev bridge-utils wget unzip build-essential bison flex iproute2
```

* Build custom iproute2 package. (loxilb  requires a special version of [iproute2](https://github.com/shemminger/iproute2) tool for its operation. The customized repository can be found [here](https://github.com/loxilb-io/iproute2). loxilb needs some patches to iproute's tc module to properly load/unload its ebpf modules

```
git clone https://github.com/loxilb-io/iproute2.git
cd iproute2
cd libbpf/src/
mkdir build
DESTDIR=build make install
cd ../../
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:`pwd`/libbpf/src/
LIBBPF_FORCE=on LIBBPF_DIR=`pwd`/libbpf/src/build ./configure
make
sudo cp -f tc/tc /usr/local/sbin/ntc
```

* Build and run loxilb 

```
git --recurse-submodules clone https://github.com/loxilb-io/loxilb.git
cd loxilb
./loxilb-ebpf/utils/mkllb_bpffs.sh
make
cd loxilb-ebpf/libbpf/src
sudo make install
cd -
sudo ./loxilb 
```
* Build and use loxicmd 

```
git clone https://github.com/loxilb-io/loxicmd.git
cd loxicmd
go get .
make
```
loxicmd usage guide can be found [here](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/cmd.md)

## From docker (easy)

* Get the latest loxilb official docker image 

```
docker pull ghcr.io/loxilb-io/loxilb:latest
```

* To run loxilb docker, we can use the following commands :

```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest
```

* To drop in to a shell of loxilb doker :

```
docker exec -it loxilb bash
```

* For load-balancing to effectively work in a bare-metal environment, we need multiple interfaces assigned to the docker (external and internal connectivitiy) 

  loxilb docker relies on docker's macvlan driver for achieving this. The following is an example of creating macvlan network and using with loxilb

```
# Create a mac-vlan (on an underlying interface e.g. enp0s3)
docker network create -d macvlan -o parent=enp0s3   --subnet 172.30.1.0/24   --gateway 172.30.1.254 --aux-address 'host=172.30.1.193’ llbnet

# Run loxilb docker with the created macvlan 
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --net=llbnet --ip=172.30.1.193 --name loxilb ghcr.io/loxilb-io/loxilb:latest

# If we still want to connect loxilb docker additionally to docker's default network or more macvlan networks
docker network connect bridge loxilb
```
  *Note - While working with macvlan interfaces, the parent/underlying interface should be put in promiscous mode*
  
<b>To create a simple and self-contained topology for testing loxilb, users can follow this [guide](simple_topo.md)</b>

