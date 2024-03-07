# Table of Contents

 - [What is loxicmd](./cmd.md#what-is-loxicmd)
 - [How to build](./cmd.md#how-to-build)
 - [How to run and configure loxilb](./cmd.md#how-to-run-and-configure-loxilb)
	 - [Load balancer](./cmd.md#load-balancer)
	 - [Endpoint](./cmd.md#endpoint)
  	 - [BFD](./cmd.md#bfd)
	 - [Session](./cmd.md#session)
	 - [SessionUlCl](./cmd.md#sessionulcl)
	 - [IPaddress](./cmd.md#ipaddress)
	 - [FDB](./cmd.md#fdb)
	 - [Route](./cmd.md#route)
	 - [Neighbor](./cmd.md#neighbor)
	 - [Vlan](./cmd.md#vlan)
	 - [Vxlan](./cmd.md#vxlan)
	 - [Firewall](./cmd.md#firewall)
	 - [Mirror](./cmd.md#mirror)
	 - [Policy](./cmd.md#policy)
	 - [Session Recorder](./cmd.md#session-recorder)

## What is loxicmd

loxicmd is command tool for loxilb's configuration. loxicmd aims to provide all configuation related to loxilb and is based on kubectl's look and feel. When running k8s,  [kube-loxilb](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/kube-loxilb.md) usually takes care of loxilb configuration but nonetheless loxicmd can be used for enhanced config, debugging and observability.         
   
## How to build

Note - loxilb docker has this built-in and there is no need to build it when using loxilb docker

Install package dependencies 

```
go get .
```

Make loxicmd

```
make
```

Install loxicmd

```
sudo cp -f ./loxicmd /usr/local/sbin
```

## How to run and configure loxilb

### Load Balancer
#### Get load-balancer rules

Get basic information
```
loxicmd get lb
```

Get detailed information
```
loxicmd get lb -o wide
```
Get info in json
```
loxicmd get lb -o json
```
#### Configure load-balancer rule
Simple NAT44 tcp (round-robin) load-balancer
```
loxicmd create lb 1.1.1.1 --tcp=1828:1920 --endpoints=2.2.3.4:1
```
***Note:***   
- Round-robin is default mode in loxilb    
- End-point format is specified as &lt;CIDR:weight&gt;. For round-robin, weight(1) has no significance.

NAT66 (round-robin) load-balancer
```
loxicmd create lb  2001::1 --tcp=2020:8080 --endpoints=4ffe::1:1,5ffe::1:1,6ffe::1:1
```

NAT64 (round-robin) load-balancer
```
loxicmd create lb  2001::1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1,33.33.33.1:1
```

WRR (Weighted round-robin) load-balancer (Divide traffic in 40%, 40% and 20% ratio among end-points)
```
loxicmd create lb 20.20.20.1 --select=priority --tcp=2020:8080 --endpoints=31.31.31.1:40,32.32.32.1:40,33.33.33.1:20
```

Sticky end-point selection load-balancer (select end-points based on traffic hash)
```
loxicmd create lb 20.20.20.1 --select=hash --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1,33.33.33.1:1
```

Load-balancer with forceful tcp-reset session timeout after inactivity of 30s
```
loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1,33.33.33.1:1 --inatimeout=30
```

Load-balancer with one-arm mode
```
loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=100.100.100.2:1,100.100.100.3:1,100.100.100.4:1 --mode=onearm
```

Load-balancer with fullnat mode
```
loxicmd create lb 88.88.88.1 --sctp=38412:38412 --endpoints=192.168.70.3:1 --mode=fullnat
```
- For more information on  one-arm and full-nat mode, please check this [post](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/nat.md)

Load-balancer config in DSR(direct-server return) mode
```
loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1 --mode=dsr
```

Load-balancer config with active endpoint monitoring
```
loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1 --monitor
```    
***Note:***    
- By default loxilb does not do active endpoint monitoring i.e it will continue to select end-points which might be inactive   
- This is due to the fact kubernetes also has its own service monitoring mechanism and it can notify loxilb of any such endpoint health state    
- Based on user's requirements, one can specify active endpoint checks using "--monitor" flag   
- loxilb has extensive endpoint monitoring methods. Further details can be found in endpoint [section]("https://github.com/loxilb-io/loxilbdocs/edit/main/docs/cmd.md#endpoint)   


##### Load-balancer yaml example
```
apiVersion: netlox/v1
kind: Loadbalancer
metadata:
  name: test
spec:
  serviceArguments:
    externalIP: 1.2.3.1
    port: 80
    protocol: tcp
    sel: 0
  endpoints:
  - endpointIP: 4.3.2.1
    weight: 1
    targetPort: 8080
  - endpointIP: 4.3.2.2
    weight: 1
    targetPort: 8080
  - endpointIP: 4.3.2.3
    weight: 1
    targetPort: 8080
```
#### Delete a load-balancer rule
```
loxicmd delete lb 1.1.1.1 --tcp=1828 
```
---

### Endpoint
#### Get load-balancer end-point health information
```
loxicmd get ep
```
#### Create end-point for health probing
```
# loxicmd create endpoint IP [--name=<id>] [--probetype=<probetype>] [--probereq=<probereq>] [--proberesp=<proberesp>] [--probeport=<port>] [--period=<period>] [--retries=<retries>]
loxicmd create endpoint 32.32.32.1 --probetype=http --probeport=8080 --period=60 --retries=2
```
IP(string) : Endpoint target IPaddress   
name(string) : Endpoint Identifier   
probetype(string): Probe-type:ping,http,https,udp,tcp,sctp,none   
probereq(string): If probe is http/https, one can specify additional uri path   
proberesp(string): If probe is http/https, one can specify custom response string   
probeport(int): If probe is http,https,tcp,udp,sctp one can specify custom l4port to use   
period(int): Period of probing   
retries(int): Number of retries before marking endPoint inactive   

***Notes:***   
- "name" is not required when endpoint is created initially. loxilb will allocate the name which can be checked with "loxicmd get ep". "name" can be given as an Identifier when user wants to modify endpoint probe parameters    
- Initial state of endpoint will be decided within 15 seconds of rule addition (We cant be sure if service is immediately up so this is the init liveness check timeout. It is not configurable at this time)    
- After init liveness check, probes will be done as per default (60s) or whatever value is set by the user    
- When endpoint is inactive we have internal logic and timeouts to minimize blocking calls and maintain stability. Only when endpoint is active, we use probe timeout given by user    
- For UDP end-points and probe-type, there are two ways to check end-point health currently:    
  - If the service can respond to probe requests with pre-defined responses sent over UDP, we can use the following :    
    ```
    loxicmd create endpoint 172.1.217.133 --name="udpep1" --probetype=udp --probeport=32031 --period=60 --retries=2 --probereq="probe" --proberesp="hello"
    ```    
  - If the services cannot support the above mechanism, loxilb will try to check for "ICMP Port unreachable" after sending UDP probes. If an "ICMP Port unreachable" is received, it means the endpoint is not up. 

##### Examples :   
```
loxicmd create endpoint 32.32.32.1 --probetype=http --probeport=8080 --period=60 --retries=2

loxicmd get ep
|    HOST    |         NAME         | PTYPE | PORT | DURATION | RETRIES | MINDELAY | AVGDELAY | MAXDELAY | STATE |
|------------|----------------------|-------|------|----------|---------|----------|----------|----------|-------|
| 32.32.32.1 | 32.32.32.1_http_8080 | http: | 8080 |       60 |       2 | 0s       | 0s       | 0s       | ok    |

# Modify duration and retry count using name

loxicmd create endpoint 32.32.32.1 --name=32.32.32.1_http_8080 --probetype=http --probeport=8080 --period=40 --retries=4

loxicmd get ep
|    HOST    |         NAME         | PTYPE | PORT | DURATION | RETRIES | MINDELAY | AVGDELAY  | MAXDELAY  | STATE |
|------------|----------------------|-------|------|----------|---------|----------|-----------|-----------|-------|
| 32.32.32.1 | 32.32.32.1_http_8080 | http: | 8080 |       40 |       4 | 0s       | 0s        | 0s        | ok    |

```

#### Create end-point with https probing information
```
# loxicmd create endpoint IP [--name=<id>] [--probetype=<probetype>] [--probereq=<probereq>] [--proberesp=<proberesp>] [--probeport=<port>] [--period=<period>] [--retries=<retries>]
loxicmd create endpoint 32.32.32.1 --probetype=https --probeport=8080 --probereq="health" --proberesp="OK" --period=60 --retries=2
```
***Note:*** loxilb requires CA certificate for TLS connection and private certificate and private key for mTLS connection. Admin can keep a common(default) CA certificate for all the endpoints at "/opt/loxilb/cert/rootCA.crt" or per-endpoint certificates can be kept as "/opt/loxilb/cert/\<IP\>/rootCA.crt", private key must be kept at "/opt/loxilb/cert/server.key" and private certificate at "/opt/loxilb/cert/server.crt".
Please see [Minica](https://github.com/jsha/minica) or [Certstrap](https://github.com/square/certstrap) or [this](https://github.com/loxilb-io/loxilb/tree/main/cicd/httpsep) CICD test case to know how to generate certificates.
	
##### Endpoint yaml example
```
apiVersion: netlox/v1
kind: Endpoint
metadata:
  name: test
spec:
  hostName: "Test"
  description: string
  inactiveReTries: 0
  probeType: string
  probeReqUrl: string
  probeDuration: 0
  probePort: 0

```
#### Delete end-point informtion
```
loxicmd delete endpoint 31.31.31.31
```
---
### BFD
#### Get BFD Session information
```
loxicmd get bfd
```
#### Create BFD Session
```
#loxicmd create bfd <remoteIP> --sourceIP=<sourceIP> --interval=<time in usecs> --retryCount=<count>
loxicmd create bfd 192.168.80.253 --sourceIP=192.168.80.252 --interval=500000 --retryCount=3
```
remoteIP(string): Remote IP address   
sourceIP(string): Source IP address for binding   
interval(int): BFD packet Tx Interval Time in microseconds   
retryCount(int): Number of retry counts to detect failure.   
#### Delete BFD Session
```
#loxicmd delete bfd <remoteIP>
loxicmd delete bfd 192.168.80.253
```
remoteIP(string): Remote IP address   
sourceIP(string): Source IP address for binding   
interval(int): BFD packet Tx Interval Time in microseconds   
retryCount(int): Number of retry counts to detect failure.
#### BFD yaml example
```
apiVersion: netlox/v1
kind: BFD
metadata:
  name: test
spec:
  instance: "default"
  remoteIp: "192.168.80.253"
  sourceIp: "192.168.80.252"
  interval: 300000
  retryCount: 4

```

### Session
#### Get Session information
```
loxicmd get session
```
#### Create Session information
```
#loxicmd create session <userID> <sessionIP> --accessNetworkTunnel=<TeID>:<TunnelIP> --coreNetworkTunnel=<TeID>:<TunnelIP>
loxicmd create session user1 192.168.20.1 --accessNetworkTunnel=1:1.232.16.1 coreNetworkTunnel=1:1.233.16.1
```
userID(string): User Identifier   
sessionIP(string): Session IP address   
accessNetworkTunnel(string): accessNetworkTunnel has pairs that can be specified as 'TeID:IP'   
coreNetworkTunnel(string): coreNetworkTunnel has pairs that can be specified as 'TeID:IP'   

##### Session yaml example
```
apiVersion: netlox/v1
kind: Session
metadata:
  name: test
spec:
  ident: user1
  sessionIP: 88.88.88.88
  accessNetworkTunnel:
    TeID: 1
    tunnelIP: 11.11.11.11
  coreNetworkTunnel:
    TeID: 1
    tunnelIP: 22.22.22.22

```
#### Delete Session information
```
loxicmd delete session user1
```
---
### SessionUlCl
#### Get SessionUlCl information
```
loxicmd get sessionulcl
```
#### Create SessionUlCl information
```
#loxicmd create sessionulcl <userID> --ulclArgs=<QFI>:<ulclIP>,...
loxicmd create sessionulcl user1 --ulclArgs=16:192.33.125.1
```
userID(string): User Identifier   
ulclArgs(string): Port pairs can be specified as 'QFI:UlClIP'   

##### SessionUlCl yaml example
```
apiVersion: netlox/v1
kind: SessionULCL
metadata:
  name: test
spec:
  ulclIdent: user1
  ulclArgument:
    qfi: 11
    ulclIP: 8.8.8.8
```
#### Delete SessionUlCl information
```
loxicmd delete sessionulcl --ulclArgs=192.33.125.1
```
ulclArgs(string): UlCl IP address can be specified as 'UlClIP'. It don't need QFI.


---
### IPaddress
#### Get IPaddress information
```
loxicmd get ip
```
#### Create IPaddress information
```
#loxicmd create ip <DeviceIPNet> <device>
loxicmd create ip 192.168.0.1/24 eno7
```
DeviceIPNet(string): Actual IP address with mask   
device(string): name of the related device   

##### IPaddress yaml example
```
apiVersion: netlox/v1
kind: IPaddress
metadata:
  name: test
spec:
  dev: eno8
  ipAddress: 192.168.23.1/32

```
#### Delete IPaddress information
```
#loxicmd delete ip <DeviceIPNet> <device>
loxicmd delete ip 192.168.0.1/24 eno7
```
---
### FDB
#### Get FDB information
```
loxicmd get fdb
```
#### Create FDB information
```
#loxicmd create fdb <MacAddress> <DeviceName>
loxicmd create fdb aa:aa:aa:aa:bb:bb eno7
```
MacAddress(string): mac address   
DeviceName(string): name of the related device   

##### FDB yaml example
```
apiVersion: netlox/v1
kind: FDB
metadata:
  name: test
spec:
  dev: eno8
  macAddress: aa:aa:aa:aa:aa:aa
```
#### Delete FDB information
```
#loxicmd delete fdb <MacAddress> <DeviceName>
loxicmd delete fdb aa:aa:aa:aa:bb:bb eno7
```

---
### Route
#### Get Route information
```
loxicmd get route
```
#### Create Route information
```
#loxicmd create route <DestinationIPNet> <gateway>
loxicmd create route 192.168.212.0/24 172.17.0.254
```
DestinationIPNet(string): Actual IP address route with mask   
gateway(string): gateway information if any   

##### Route yaml example
```
apiVersion: netlox/v1
kind: Route
metadata:
  name: test
spec:
  destinationIPNet: 192.168.30.0/24
  gateway: 172.17.0.1

```
#### Delete Route information
```
#loxicmd delete route <DestinationIPNet>
loxicmd delete route 192.168.212.0/24 
```
---
### Neighbor
#### Get Neighbor information
```
loxicmd get neighbor
```
#### Create Neighbor information
```
#loxicmd create neighbor <DeviceIP> <DeviceName> [--macAddress=aa:aa:aa:aa:aa:aa]
loxicmd create neighbor 192.168.0.1 eno7 --macAddress=aa:aa:aa:aa:aa:aa
```
DeviceIP(string): The IP address   
DeviceName(string): name of the related device   
macAddress(string): resolved hardware address if any   

##### Neighbor yaml example
```
apiVersion: netlox/v1
kind: Neighbor
metadata:
  name: test
spec:
  dev: eno8
  macAddress: aa:aa:aa:aa:aa:aa
  ipAddress: 192.168.23.21
```

#### Delete Neighbor information
```
#loxicmd delete neighbor <DeviceIP> <device>
loxicmd delete neighbor 192.168.0.1 eno7
```

---
### Vlan
#### Get Vlan and Vlan Member information
```
loxicmd get vlan
```
```
loxicmd get vlanmember
```
#### Create Vlan and Vlan Member information
```
#loxicmd create vlan <Vid>
loxicmd create vlan 100
```
Vid(int): vlan identifier
```

#loxicmd create vlanmember <Vid> <DeviceName> --tagged=<Tagged>
loxicmd create vlanmember 100 eno7 --tagged=true
loxicmd create vlanmember 100 eno7
```
Vid(int): vlan identifier   
DeviceName(string): name of the related device   
tagged(boolean): tagged or not (default is false)   

##### Vlan yaml example
```
apiVersion: netlox/v1
kind: Vlan
metadata:
  name: test
spec:
  vid: 100
```
##### Vlan Member yaml example
```
apiVersion: netlox/v1
kind: VlanMember
metadata:
  name: test
  vid: 100
spec:
  dev: eno8
  Tagged: true
```
#### Delete Vlan and Vlan Member information
```
#loxicmd delete vlan <Vid>
loxicmd delete vlan 100
```
```
#loxicmd delete vlanmember <Vid> <DeviceName> --tagged=<Tagged>
loxicmd delete vlanmember 100 eno7 --tagged=true
loxicmd delete vlanmember 100 eno7
```
---
### Vxlan
#### Get Vxlan and Vxlan Peer information
```
loxicmd get vxlan
```
```
loxicmd get vxlanpeer
```
#### Create Vxlan and Vxlan Peer information
```
#loxicmd create vxlan <VxlanID> <EndpointDeviceName>
loxicmd create vxlan 100 eno7
```
VxlanID(int): Vxlan Identifier   
EndpointDeviceName(string): VTEP Device name(It must have own IP address for peering)   


```
#loxicmd create vxlanpeer <VxlanID> <PeerIP>
loxicmd create vxlan-peer 100 30.1.3.1
```
VxlanID(int):  Vxlan Identifier   
PeerIP(string): Vxlan peer device IP address   

##### Vxlan yaml example
```
apiVersion: netlox/v1
kind: Vxlan
metadata:
  name: test
spec:
  epIntf: eno8
  vxlanID: 100
```

##### Vxlan Peer yaml example
```
apiVersion: netlox/v1
kind: VxlanPeer
metadata:
  name: test
  vxlanID: 100
spec:
  peerIP: 21.21.21.1
```
#### Delete Vxlan and Vxlan Peer  information
```
#loxicmd delete vxlan <VxlanID>
loxicmd delete vxlan 100
```
```
#loxicmd delete vxlanpeer <VxlanID> <PeerIP>
loxicmd delete vxlan-peer 100 30.1.3.1
```
---
### Firewall
#### Get Firewall information
```
loxicmd get firewall
```
#### Create Firewall information
```
#loxicmd create firewall --firewallRule=<ruleKey>:<ruleValue>, [--allow] [--drop] [--trap] [--redirect=<PortName>] [--setmark=<FwMark>
loxicmd create firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200" --allow
loxicmd create firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200" --allow --setmark=10
loxicmd create firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200" --drop
loxicmd create firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200" --trap
loxicmd create firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200" --redirect=ensp0
```
**firewallRule**
sourceIP(string) - Source IP in CIDR notation   
destinationIP(string) - Destination IP in CIDR notation   
minSourcePort(int) - Minimum source port range   
maxSourcePort(int) - Maximum source port range   
minDestinationPort(int) - Minimum destination port range   
maxDestinationPort(int) - Maximum source port range   
protocol(int) - the protocol   
portName(string) - the incoming port   
preference(int) - User preference for ordering   

##### Firewall yaml example
```
apiVersion: netlox/v1
kind: Firewall
metadata:
  name: test
spec:
  ruleArguments:
    sourceIP: 192.169.1.2/24
    destinationIP: 192.169.2.1/24
    preference: 200
  opts:
    allow: true
```

#### Delete Firewall information
```
#loxicmd delete firewall --firewallRule=<ruleKey>:<ruleValue>
loxicmd delete firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200
```
---
### Mirror
#### Get Mirror information
```
loxicmd get mirror
```
#### Create Mirror information
```
#loxicmd create mirror <mirrorIdent> --mirrorInfo=<InfoOption>:<InfoValue>,... --targetObject=attachement:<port1,rule2>,mirrObjName:<ObjectName>
loxicmd create mirror mirr-1 --mirrorInfo="type:0,port:ensp0" --targetObject="attachement:1,mirrObjName:ensp1
```
mirrorIdent(string): Mirror identifier   
type(int) : Mirroring type as like 0 == SPAN, 1 == RSPAN, 2 == ERSPAN    
port(string) : The port where mirrored traffic needs to be sent   
vlan(int) : for RSPAN we may need to send tagged mirror traffic   
remoteIP(string) : For ERSPAN we may need to send tunnelled mirror traffic   
sourceIP(string): For ERSPAN we may need to send tunnelled mirror traffic   
tunnelID(int): For ERSPAN we may need to send tunnelled mirror traffic   

##### Mirror yaml example
```
apiVersion: netlox/v1
kind: Mirror
metadata:
  name: test
spec:
  mirrorIdent: mirr-1
  mirrorInfo:
    type: 0
    port: eno1
  targetObject:
    attachment: 1
    mirrObjName: eno2

```

#### Delete Mirror information
```
#loxicmd delete mirror <mirrorIdent>
loxicmd delete mirror mirr-1
```

---
### Policy
#### Get Policy information
```
loxicmd get policy
```
#### Create Policy information
```
#loxicmd create policy IDENT --rate=<Peak>:<Commited> --target=<ObjectName>:<Attachment> [--block-size=<Excess>:<Committed>] [--color] [--pol-type=<policy type>]
loxicmd create policy pol-0 --rate=100:100 --target=ensp0:1
loxicmd create policy pol-1 --rate=100:100 --target=ensp0:1 --block-size=12000:6000
loxicmd create policy pol-1 --rate=100:100 --target=ensp0:1 --color
loxicmd create policy pol-1 --rate=100:100 --target=ensp0:1 --color --pol-type 0
```
rate(string): Rate pairs can be specified as 'Peak:Commited'. *rate unit : Mbps   
block-size(string): Block Size pairs can be specified as 'Excess:Committed'. *block-size unit : bps   
target(string): Target Interface pairs can be specified as 'ObjectName:Attachment'   
color(boolean): Policy color enbale or not   
pol-type(int): Policy traffic control type. 0 : TrTCM, 1 : SrTCM   

##### Policy yaml example
```
apiVersion: netlox/v1
kind: Policy
metadata:
  name: test
spec:
  policyIdent: pol-eno8
  policyInfo:
    type: 0
    colorAware: false
    committedInfoRate: 100
    peakInfoRate: 100
  targetObject:
    attachment: 1
    polObjName: eno8

```
#### Delete Policy information
```
#loxicmd delete policy <Polident>
loxicmd delete policy pol-1
```
---   
### Session Recorder
#### Set n-tuple policy for recording
```
loxicmd create firewall --firewallRule="destinationIP:31.31.31.0/24,preference:200" --allow --record
```
loxilb will record any connection track entry which matches this policy (even for reverse direction) as a way to provide extended visibility for debugging   
#### Check or record with tcpdump
```
tcpdump -i llb0 -n
```
Any valid tcpdump option can be used including saving to a pcap file

---


### Get live connection-track information
```
loxicmd get conntrack
```

### Get port-dump information
```
loxicmd get port
```
### Save all loxilb's operational information in DBStore
```
loxicmd save -a
```
** This will ensure that whenever loxilb restarts, it will start with last saved state from DBStore

### Configure loxicmd with yaml(Beta)
The loxicmd support yaml based configuration. The format is same as Kubernetes. This beta version support only one configuraion per one file. That means "Do not use `---` in yaml file." . It will be supported at next release.

#### Command 
```
#loxicmd apply -f <file.yaml>
#loxicmd delete -f <file.yaml>
loxicmd apply -f lb.yaml
loxicmd delete -f lb.yaml
```
#### File example(lb.yaml)
```
apiVersion: netlox/v1
kind: Loadbalancer
metadata:
  name: load
spec:
  serviceArguments:
    externalIP: 123.123.123.1
    port: 80
    protocol: tcp
    sel: 0
  endpoints:
  - endpointIP: 4.3.2.1
    weight: 1
    targetPort: 8080
  - endpointIP: 4.3.2.2
    weight: 1
    targetPort: 8080
  - endpointIP: 4.3.2.3
    weight: 1
    targetPort: 8080
```
It reuse API's json body as a "Spec". If the API URL has no param, it don't need to use "metadata". For example, The body of load Balancer rule is shown below.

```
{
  "serviceArguments": {
    "externalIP": "123.123.123.1",
    "port": 80,
    "protocol": "tcp",
    "sel": 0
  },
  "endpoints": [
    {
      "endpointIP": "4.3.2.1",
      "weight": 1,
      "targetPort": 8080
    },
    {
      "endpointIP": "4.3.2.2",
      "weight": 1,
      "targetPort": 8080
    },
    {
      "endpointIP": "4.3.2.3",
      "weight": 1,
      "targetPort": 8080
    }
}
```
This json format can be converted Yaml format as shown below.
```
  serviceArguments:
    externalIP: 123.123.123.1
    port: 80
    protocol: tcp
    sel: 0
  endpoints:
  - endpointIP: 4.3.2.1
    weight: 1
    targetPort: 8080
  - endpointIP: 4.3.2.2
    weight: 1
    targetPort: 8080
  - endpointIP: 4.3.2.3
    weight: 1
    targetPort: 8080
```
Finally, this is located in the Spec of the entire configuration file as [File example(lb.yaml)](https://github.com/loxilb-io/loxilbdocs/edit/main/docs/cmd.md#file-examplelbyaml)

If you want to add Vlan bridge, IPaddress or something else. Just change the Kind value from Loadbalancer to VlanBridge, IPaddress as like below example. 
```
apiVersion: netlox/v1
kind: IPaddress
metadata:
  name: test
spec:
  dev: eno8
  ipAddress: 192.168.23.1/32
```

If the URL has param such as adding vlan-member, it must have `metadata`. 
```
apiVersion: netlox/v1
kind: VlanMember
metadata:
  name: test
  vid: 100
spec:
  dev: eno8
  Tagged: true
```
The example of all the settings below, so please refer to it.

### More information

There are tons of other commands, use help option!
```
loxicmd help
```   
