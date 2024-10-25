## How to run loxilb in standalone mode

This guide will help users to run loxilb in a standalone mode decoupled from kubernetes

### Pre-requisites 

*This guide uses Ubuntu 20.04.5 LTS as the base operating system*

#### Install docker    

One can follow the guide [here](https://docs.docker.com/engine/install/ubuntu/) to install latest docker engine or use snap to install docker.
```
sudo apt update
sudo apt install snapd
sudo snap install docker
```

#### Enable IPv6 (if running NAT64/NAT66)   
```
sysctl net.ipv6.conf.all.disable_ipv6=0
sysctl net.ipv6.conf.default.disable_ipv6=0
```

### Run loxilb 

Get the loxilb official docker image    

* Latest build image (multi-arch amd64/arm64)   
```
docker pull ghcr.io/loxilb-io/loxilb:latest
```   

* Release build image   
```
docker pull ghcr.io/loxilb-io/loxilb:v0.9.7
``` 

* To run loxilb docker, we can use the following commands :   

```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest
```

* To drop in to a shell of loxilb doker :   

```
docker exec -it loxilb bash
```

* For load-balancing to effectively work in a bare-metal environment, we need multiple interfaces assigned to the docker (external and internal connectivitiy). loxilb docker relies on docker's macvlan driver for achieving this. The following is an example of creating macvlan network and using with loxilb:   

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
* One can further use docker-compose to automate attaching multiple networks to loxilb docker or use ```--net=host``` as per requirement
* To use local socket policy or eBPF sockmap related features, we need to use ```--pid=host --cgroupns=host``` as additional arguments to docker run.   
* To create a simple and self-contained topology for testing loxilb, users can follow this [guide](simple_topo.md)   
* If loxilb docker is in the same node as the app/workload docker, it is advised that "tx checksum offload" inside app/workload docker is turned off for sctp load-balancing to work properly   
```
docker exec -dt <app-docker-name> ethtool -K <app-docker-interface> tx off
```   

### Configuration   

loxicmd command line tool can be used to configure loxilb in standalone mode. A simple example of configuration using loxilb is as follows:   

* Drop into loxilb shell   
```
sudo docker exec -it loxilb bash
```
* Create a LB rule inside loxilb docker. Various other options for LB manipulation can be found [here](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/cmd.md#load-balancer)    
```
loxicmd create lb 2001::1 --tcp=2020:8080 --endpoints=33.33.33.1:1
```
* Validate entry is created using the command:    
```
loxicmd get lb -o wide
```
The detailed usage guide of loxicmd can be found [here](https://loxilb-io.github.io/loxilbdocs/cmd/).   

### Working with gobgp

loxilb works in tandem with gobgp when bgp services  are required. As a first step, create a file gobgp.conf in host where loxilb docker will run and add the basic necessary fields :   

```
[global.config]
  as = 64512
  router-id = "10.10.10.1"

[[neighbors]]
  [neighbors.config]
    neighbor-address = "10.10.10.254"
    peer-as = 64512
```

Run loxilb docker with following arguments:   
```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v gobgp.conf:/etc/gobgp/gobgp.conf -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest -b 
```

The gobgp daemon should pick the configuration. The neighbors can be verified by :   

```
sudo docker exec -it loxilb gobgp neighbor
```

At run time, there are two ways to change gobgp configuration. Ephemeral configuration can simply be done using “gobgp” command as detailed [here](https://github.com/osrg/gobgp/blob/master/docs/sources/cli-operations.md). If persistence is required, then one can change the gobgp config file /etc/gobgp/gobgp.conf and apply SIGHUP to gobgpd process for loading the edited configuration.   

```
sudo docker exec -it loxilb pkill -1 gobgpd
```
### Persistent LB entries

To save the created rules across reboots, one can use the following command:    

```
sudo mkdir -p /etc/loxilb/
sudo loxicmd save --lb
```





