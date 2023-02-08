# Table of Contents

 - [What is loxicmd](./cmd.md#what-is-loxicmd)
 - [How to build](./cmd.md#how-to-build)
 - [How to run and configure loxilb](./cmd.md#how-to-run-and-configure-loxilb)
	 - [Load balancer](./cmd.md#load-balancer)
	 - [Endpoint](./cmd.md#endpoint)
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
	 - [PCAP Recorder](./cmd.md#pcap-recording)
- [loxicmd development guide](./cmd.md#loxicmd-development-guide)

## What is loxicmd

loxicmd is command tool for loxilb's configuration. loxicmd aims to provide all configuation related to loxilb and is based on kubectl's look and feel. 

## How to build

Note - loxilb docker has this built-in

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
** Please note that round-robin is default mode in loxilb    
** End-point format is specified as &lt;CIDR:weight&gt;. For round-robin, weight(1) has no significance.

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
** For more information on  one-arm and full-nat mode, please check this [post](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/nat.md)

Load-balancer config in DSR(direct-server return) mode
```
loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1 --mode=dsr
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
#### Create end-point information
```
# loxicmd create endpoint IP [--desc=<desc>] [--probetype=<probetype>] [--probereq=<probereq>] [--proberesp=<proberesp>] [--probeport=<port>] [--period=<period>] [--retries=<retries>]
loxicmd create endpoint 32.32.32.1 --desc=zone1host --probetype=http --probeport=8080 --period=60 --retries=2
```
IP(string) : Endpoint target IPaddress\
desc(string) : Description of and end-point\
probetype(string): Probe-type:ping,http,https,connect-udp,connect-tcp,connect-sctp,none\
probereq(string): If probe is http/https, one can specify additional uri path\
proberesp(string): If probe is http/https, one can specify custom response string\
probeport(int): If probe is http,https,tcp,udp,sctp one can specify custom l4port to use\
period(int): Period of probing\
retries(int): Number of retries before marking endPoint inactive\
#### Create end-point with https probing information
```
# loxicmd create endpoint IP [--desc=<desc>] [--probetype=<probetype>] [--probereq=<probereq>] [--proberesp=<proberesp>] [--probeport=<port>] [--period=<period>] [--retries=<retries>]
loxicmd create endpoint 32.32.32.1 --desc=zone1host --probetype=https --probeport=8080 --probereq="health" --proberesp="OK" --period=60 --retries=2
```
***Note:*** loxilb requires CA certificate for HTTPS connection. Admin can keep a common(default) CA certificate for all the endpoints at "/opt/loxilb/cert/rootCACert.pem" or per-endpoint certificates can be kept as "/opt/loxilb/cert/\<IP\>/rootCACert.pem"
Please see [Minica](https://github.com/jsha/minica) or [Certstrap](https://github.com/square/certstrap) to know how to generate certificates.
	
#### Delete end-point informtion
```
loxicmd delete endpoint 31.31.31.31
```
---
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
userID(string): User Ident\
sessionIP(string): Session IPaddress\
accessNetworkTunnel(string): accessNetworkTunnel has pairs that can be specified as 'TeID:IP'\
coreNetworkTunnel(string): coreNetworkTunnel has pairs that can be specified as 'TeID:IP'\

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
userID(string): User Ident\
ulclArgs(string): Port pairs can be specified as 'QFI:UlClIP'\

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
DeviceIPNet(string): Actual IP address with mask\
device(string): name of the related device\

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
MacAddress(string): mac address\
DeviceName(string): name of the related device\

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
DestinationIPNet(string): Actual IP address route with mask\
gateway(string): gateway information if any\

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
DeviceIP(string): The IP address\
DeviceName(string): name of the related device\
macAddress(string): resolved hardware address if any\
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
Vid(int): vlan identifier\
DeviceName(string): name of the related device\
tagged(boolean): tagged or not (default is false)\

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
VxlanID(int): Vxlan Identifier\
EndpointDeviceName(string): VTEP Device name(It must have own IP address for peering.)\
```
#loxicmd create vxlanpeer <VxlanID> <PeerIP>
loxicmd create vxlan-peer 100 30.1.3.1
```
VxlanID(int):  Vxlan Identifier\
PeerIP(string): Vxlan peer device IP address\

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
loxicmd create firewall --firewallRule="sourceIP:1.2.3.2/32,destinationIP:2.3.1.2/32,preference:200" --redirect=hs1
```
**firewallRule**
sourceIP(string) - Source IP in CIDR notation\
destinationIP(string) - Destination IP in CIDR notation\
minSourcePort(int) - Minimum source port range\
maxSourcePort(int) - Maximum source port range\
minDestinationPort(int) - Minimum destination port range\
maxDestinationPort(int) - Maximum source port range\
protocol(int) - the protocol\
portName(string) - the incoming port\
preference(int) - User preference for ordering\
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
loxicmd create mirror mirr-1 --mirrorInfo="type:0,port:hs0" --targetObject="attachement:1,mirrObjName:hs1
```
mirrorIdent(string): Mirror identifier\
type(int) : Mirroring type as like 0 == SPAN, 1 == RSPAN, 2 == ERSPAN\
port(string) : The port where mirrored traffic needs to be sent\
vlan(int) : for RSPAN we may need to send tagged mirror traffic\
remoteIP(string) : For ERSPAN we may need to send tunnelled mirror traffic\
sourceIP(string): For ERSPAN we may need to send tunnelled mirror traffic\
tunnelID(int): For ERSPAN we may need to send tunnelled mirror traffic\
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
loxicmd create policy pol-hs0 --rate=100:100 --target=hs0:1
loxicmd create policy pol-hs1 --rate=100:100 --target=hs0:1 --block-size=12000:6000
loxicmd create policy pol-hs1 --rate=100:100 --target=hs0:1 --color
loxicmd create policy pol-hs1 --rate=100:100 --target=hs0:1 --color --pol-type 0
```
rate(string): Rate pairs can be specified as 'Peak:Commited'. *rate unit : Mbps\
block-size(string): Block Size pairs can be specified as 'Excess:Committed'. *block-size unit : bps\
target(string): Target Interface pairs can be specified as 'ObjectName:Attachment'\
color(boolean): Policy color enbale or not\
pol-type(int): Policy traffic control type. 0 : TrTCM, 1 : SrTCM\

#### Delete Policy information
```
#loxicmd delete policy <Polident>
loxicmd delete policy pol-hs1
```
---   
### PCAP Recording
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

There are tons of other commands, use help option!
```
loxicmd help
```
## Configure loxicmd with yaml
TBD

## loxicmd development guide
It will provide a guide for development. Please develop it according to the guidelines. The guide is divided into three main stages: design, development, and testing, and the details are as follows.

1. API check and command design
Before developing Command, we need to check if the API of the necessary functions is provided. Check the official API document of LoxiLB to see if the required API is provided. Afterwards, the GET, POST, and DELETE methods are designed with get, create, and delete commands according to the API provided.

```
loxicmd$ tree
.
├── AUTHORS
├── cmd
│   ├── create
│   │   ├── create.go
│   │   └── create_loadbalancer.go
│   ├── delete
│   │   ├── delete.go
│   │   └── delete_loadbalancer.go
│   ├── get
│   │   ├── get.go
│   │   ├── get_loadbalancer.go
│   └── root.go
├── go.mod
├── go.sum
├── LICENSE
├── main.go
├── Makefile
├── pkg
│   └── api
│       ├── client.go
│       ├── common.go
│       ├── loadBalancer.go
│       └── rest.go
└── README.md

```
Add the code in the ./cmd/get, ./cmd/delete, ./cmd/create, and ./pkg/api directories to add functionality.

2. Add structure in pkg/api and register method (example of connection track API)
  * CommonAPI embedding
Using embedding the CommonAPI for the Methods and variables, to use in the Connecttrack structure.
```
type Conntrack struct {
    CommonAPI
}
```

* Add Structure Configuration and JSON Structure
Define the structure for JSON Unmashal.
```
type CtInformationGet struct {
    CtInfo []ConntrackInformation `json:"ctAttr"`
}

type ConntrackInformation struct {
    Dip    string `json:"destinationIP"`
    Sip    string `json:"sourceIP"`
    Dport  uint16 `json:"destinationPort"`
    Sport  uint16 `json:"sourcePort"`
    Proto  string `json:"protocol"`
    CState string `json:"conntrackState"`
    CAct   string `json:"conntrackAct"`
}
```
* Define Method Functions in pkg/api/client.go
Define the URL in the Resource constant.
Defines the function to be used in the command.
```
const (
    …
    loxiConntrackResource    = "config/conntrack/all"
)


func (l *LoxiClient) Conntrack() *Conntrack {
    return &Conntrack{
        CommonAPI: CommonAPI{
            restClient: &l.restClient,
            requestInfo: RequestInfo{
                provider:   loxiProvider,
                apiVersion: loxiApiVersion,
                resource:   loxiConntrackResource,
            },
        },
    }
}

```
3. Add get, create, delete functions within cmd
Use the Cobra library to define commands, Alise, descriptions, options, and callback functions, and then create a function that returns.
Create a function such as PrintGetCTReturn and add logic when the status code is 200.
```
func NewGetConntrackCmd(restOptions *api.RESTOptions) *cobra.Command {
	var GetctCmd = &cobra.Command{
		Use:     "conntrack",
		Aliases: []string{"ct", "conntracks", "cts"},
		Short:   "Get a Conntrack",
		Long:    `It shows connection track Information`,
		Run: func(cmd *cobra.Command, args []string) {
			client := api.NewLoxiClient(restOptions)
			ctx := context.TODO()
			var cancel context.CancelFunc
			if restOptions.Timeout > 0 {
				ctx, cancel = context.WithTimeout(context.TODO(), time.Duration(restOptions.Timeout)*time.Second)
				defer cancel()
			}
			resp, err := client.Conntrack().Get(ctx)
			if err != nil {
				fmt.Printf("Error: %s\n", err.Error())
				return
			}
			if resp.StatusCode == http.StatusOK {
				PrintGetCTResult(resp, *restOptions)
				return
			}

		},
	}

	return GetctCmd
}

```
4. Register command in cmd
Register Cobra as defined in 3.
```
func GetCmd(restOptions *api.RESTOptions) *cobra.Command {
    var GetCmd = &cobra.Command{
        Use:   "get",
        Short: "A brief description of your command",
        Long: `A longer description that spans multiple lines and likely contains examples
    and usage of using your command. For example:
   
    Cobra is a CLI library for Go that empowers applications.
    This application is a tool to generate the needed files
    to quickly Get a Cobra application.`,
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Println("Get called")
        },
    }
    GetCmd.AddCommand(NewGetLoadBalancerCmd(restOptions))
    GetCmd.AddCommand(NewGetConntrackCmd(restOptions))
    return GetCmd
}
```

5. Build & Test
```
make
```
Test the command as you want!
