# loxilb - How to debug and troubleshoot

## * loxilb docker or pod not coming in <I>Running</I> state ?

* <b>Solution:</b> 

  * Check the host machine kernel version. loxilb requires <b>kernel version 5.8 or above</b>.

  * Make sure you are running the correct image as per your environment.

## * externalIP pending in "kubectl get svc" ?

If this happens:

<b>1.</b> When running loxilb externally, there could be a connectivity issue.
  
  * <b>Solution:</b>
  
    * Check if loxiURL annotation in kube-loxilb.yaml was set correctly.
    * Check for kube-loxilb(master node) connectivity with loxilb node.
  
<b>2.</b> When running loxilb in-cluster mode
  
  * <b>Solution:</b> Make sure loxilb pods were spawned.
  
<b>3.</b> Make sure the annotation <b>"node.kubernetes.io/exclude-from-external-load-balancers"</b> is NOT present in the node's configuration.
  
  * <b>Solution:</b> If present, then the node will not be considered as an endpoint by loxilb. You can remove it by editing "kubectl edit \<node-name\>"

<b>4.</b> Make sure these annotations are present in your service.yaml
  ```
    spec:
      loadBalancerClass: loxilb.io/loxilb
      type: LoadBalancer
  ```

## * SCTP packets dropping ?

Usually, This happens due to SCTP checksum validation by host kernel and the possible scenarios are:

<b>1.</b> When workload and loxilb are scheduled in the same node. 

<b>2.</b> Different CNI creates different types of interfaces i.e. CNIs creates bridges/tunnels/veth pairs and different network policies. These interfaces have different characteristics and implications on loxilb's checksum calculation logic.
  
  * <b>Solution:</b> 

    There are two ways to resolve this issue:

    * Disable checksum calculation.
    
    ```
      echo 1 >  /sys/module/sctp/parameters/no_checksums
      echo 0 >  /proc/sys/net/netfilter/nf_conntrack_checksum
    ```   
    
    * Or, Let loxilb take care of the checksum calculation completely. For that, We need to install a utility(a kernel module) in all the nodes where loxilb is running. It will make sure the correct checksum is applied at the end.   
    
    ```
      curl -sfL https://github.com/loxilb-io/loxilb-ebpf/raw/main/kprobe/install.sh | sh -
    ```

## * ABORT in SCTP ?

SCTP ABORT can be seen in many scenarios:

<b>1.</b> When Service IP is same as loxilb IP and SCTP packets does not match the rules.
  
  * <b>Solution:</b>
  
    * Check if the rule is installed properly
    
      ```
        loxicmd get lb
      ```   

    * Make sure the client is connecting to the same IP and port as per the configured service LB rule.
  
<b>2.</b> In one-arm/fullnat mode, loxilb sends SCTP ABORT after receiving SCTP INIT ACK packet.
  
  * <b>Solution:</b> Check the underlying hypervisor interface driver. 
  Some drivers does not provide enough metadata for ebpf processing which makes the packet to follow fallback path to kernel and kernel
  being unaware of the SCTP connection sends SCTP ABORT.
  Emulated interfaces in bridge mode are preferred for smooth networking.

<b>3.</b> ABORT after few seconds(Heartbeat re-transmisions)

  When initiating the SCTP connection, if the application is not binded with a particular IP then SCTP stack uses all the IPs in the SCTP INIT message.
  After the successful connection, both endpoints start health check for each network path. 
  As loxilb is in between and unaware of all the endpoint IPs, drops all those packets, which leads to sending SCTP ABORT from the endpoint.
  
  * <b>Solution:</b> In SCTP uni-homing case, it is absolutely necessary to make sure the applications are binded to only one IP to avoid this case.

<b>4.</b> ABORT after few seconds(SCTP Multihoming)

  * <b>Solution:</b> Currently, SCTP Multihoming service works only with fullnat mode and externalTrafficPolicy set to <b><I>"Local"</I></b>
  
## * <b>Check loxilb logs</b>

loxilb logs its various important events and logs in the file /var/log/loxilb.log. Users can check it by using tail -f or any other command of choice. 

```
root@752531364e2c:/# tail -f /var/log/loxilb.log 
DBG:  2022/07/10 12:49:27 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:49:37 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:49:47 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:49:57 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:50:07 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:50:17 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:50:27 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:50:37 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:50:47 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
DBG:  2022/07/10 12:50:57 1:dst-10.10.10.1/32,proto-6,dport-2020,,do-dnat:eip-31.31.31.1,ep-5001,w-1,alive|eip-32.32.32.1,ep-5001,w-2,alive|eip-100.100.100.1,ep-5001,w-2,alive| pc 0 bc 0 
```


## * <b>Check *loxicmd* to debug loxilb's internal state</b>

```
### Spawn a bash shell of loxilb docker 
docker exec -it loxilb bash

root@752531364e2c:/# loxicmd get lb       
| EXTERNALIP | PORT | PROTOCOL | SELECT | # OF ENDPOINTS |
|------------|------|----------|--------|----------------|
| 10.10.10.1 | 2020 | tcp      |      0 |              3 |


root@752531364e2c:/# loxicmd get lb -o wide
| EXTERNALIP | PORT | PROTOCOL | SELECT |  ENDPOINTIP   | TARGETPORT | WEIGHT |
|------------|------|----------|--------|---------------|------------|--------|
| 10.10.10.1 | 2020 | tcp      |      0 | 31.31.31.1    |       5001 |      1 |
|            |      |          |        | 32.32.32.1    |       5001 |      2 |
|            |      |          |        | 100.100.100.1 |       5001 |      2 |


root@0c4f9175c983:/# loxicmd get conntrack
| DESTINATIONIP |  SOURCEIP  | DESTINATIONPORT | SOURCEPORT | PROTOCOL |    STATE    | ACT |
|---------------|------------|-----------------|------------|----------|-------------|-----|
| 127.0.0.1     | 127.0.0.1  |           11111 |      47180 | tcp      | closed-wait |     |
| 127.0.0.1     | 127.0.0.1  |           11111 |      47182 | tcp      | est         |     |
| 32.32.32.1    | 31.31.31.1 |           35068 |      35068 | icmp     | bidir       |     |


root@65ad9b2f1b7f:/# loxicmd get port
| INDEX | PORTNAME |        MAC        | LINK/STATE  |    L3INFO     |    L2INFO     |
|-------|----------|-------------------|-------------|---------------|---------------|
|     1 | lo       | 00:00:00:00:00:00 | true/false  | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3801    |
|       |          |                   |             | IPv6 : []     |               |
|     2 | vlan3801 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3801    |
|       |          |                   |             | IPv6 : []     |               |
|     3 | llb0     | 42:6e:9b:7f:ff:36 | true/false  | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3803    |
|       |          |                   |             | IPv6 : []     |               |
|     4 | vlan3803 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3803    |
|       |          |                   |             | IPv6 : []     |               |
|     5 | eth0     | 02:42:ac:1e:01:c1 | true/true   | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3805    |
|       |          |                   |             | IPv6 : []     |               |
|     6 | vlan3805 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3805    |
|       |          |                   |             | IPv6 : []     |               |
|     7 | enp1     | fe:84:23:ac:41:31 | false/false | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3807    |
|       |          |                   |             | IPv6 : []     |               |
|     8 | vlan3807 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3807    |
|       |          |                   |             | IPv6 : []     |               |
|     9 | enp2     | d6:3c:7f:9e:58:5c | false/false | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3809    |
|       |          |                   |             | IPv6 : []     |               |
|    10 | vlan3809 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3809    |
|       |          |                   |             | IPv6 : []     |               |
|    11 | enp2v15  | 8a:9e:99:aa:f9:c3 | false/false | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3811    |
|       |          |                   |             | IPv6 : []     |               |
|    12 | vlan3811 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3811    |
|       |          |                   |             | IPv6 : []     |               |
|    13 | enp3     | f2:c7:4b:ac:fd:3e | false/false | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3813    |
|       |          |                   |             | IPv6 : []     |               |
|    14 | vlan3813 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3813    |
|       |          |                   |             | IPv6 : []     |               |
|    15 | enp4     | 12:d2:c3:79:f3:6a | false/false | Routed: false | IsPVID: true  |
|       |          |                   |             | IPv4 : []     | VID : 3815    |
|       |          |                   |             | IPv6 : []     |               |
|    16 | vlan3815 | aa:bb:cc:dd:ee:ff | true/true   | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 3815    |
|       |          |                   |             | IPv6 : []     |               |
|    17 | vlan100  | 56:2e:76:b2:71:48 | false/false | Routed: false | IsPVID: false |
|       |          |                   |             | IPv4 : []     | VID : 100     |
|       |          |                   |             | IPv6 : []     |               |
