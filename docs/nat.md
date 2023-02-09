## NAT modes in loxilb 

loxilb implements a variety of NAT modes to achieve load-balancing for different scenarios as far as L4 load-balancing is concerned. These NAT modes have subtle differences and this guide will shed light on these details

### 1. Normal NAT 

This is basic NAT mode used by loxilb. In this mode, loxilb employs simple DNAT for incoming requests i.e destination IP (which is also the service IP) is changed to the chosen end-point IP. For the outgoing responses it does the exactly opposite(SNAT). Since loxilb relies on statefulness for this mode, it is necessary that return packets also traverse through loxilb. The following figure illustrates this operation -

![normal nat](photos/dnat1.png)

In this mode, original source IP is preserved till the end-point and provides best visibility for anyone needing it. Finally, this also means the end-points should know how to reach the source.

### 2. One-ARM 

Traditionally one-arm NAT mode meant that the LB node used to have a single arm (or connection) to the LAN instead of separate ingress and egress networks. loxilb's one-arm NAT mode is a little extended version of the traditional one-arm mode. In one-arm mode, loxilb chooses its LAN IP as source-IP when sending incoming requests towards end-points nodes. Even if the originating source is not on the same LAN, this is loxilb's default behaviour for one arm mode.

![normal nat](photos/onearm.png)

### 3. Full-NAT    

In the full-NAT mode, loxilb replaces the source-IP of an incoming request to a special instance IP. This instance IP is associated with each instance in a cluster deployment and maintained internally by loxilb. In this mode, various instances of loxilb cluster will have unique instance IPs and each of them will be advertised by BGP towards the end-point to set the return PATH accordingly. This helps in optimal distribution and spread of traffic in case an active-active clustering mode is desired.

![normal nat](photos/fullnat.png)

### 4. L2-DSR mode

In L2-DSR (direct server return) mode, loxilb performs load-balancing operation but without changing any IP addresses. It just updates the layer2 header as per selected end-point. Also in DSR mode, loxilb does not need statefulness and end-point can choose a different return path not involving loxilb. This maybe advantageous for certain scenarios where there is a need to reduce load in LB nodes by allowing return traffic to bypass the LB.

![l2dsr](photos/l2dsr.png)

### 5. L3-DSR mode

In L3-DSR (direct server return) mode, loxilb performs load-balancing operation but encapsulates the original payload with an IPinIP tunnel towards the end-points. Also like L2-DSR mode, loxilb does not need statefulness and end-point can choose a different/direct return path not involving loxilb.

![l3dsr](photos/l3dsr.png)





