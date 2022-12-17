## What is loxicmd

loxicmd is command tool for loxilb's configuration. loxicmd aims to provide all configuation related to loxilb and is based on kubectl's look and feel. 

## How to build

1. Install package dependencies 

```
go get .
```

2. Make loxicmd

```
make
```

## How to run

1. Run loxicmd with getting lb information

- Get basic information
```
./loxicmd get lb
```

- Get detailed information
```
./loxicmd get lb -o wide
```

2. Run loxicmd with getting lb information in the different API server(ex. 192.168.18.10) and ports(ex. 8099).
```
./loxicmd get lb -s 192.168.18.10 -p 8099
```

3. Run loxicmd with getting lb information as json output format
```
./loxicmd get lb -o json
```

4. Run loxicmd to configure load-balancer

**- Simple NAT44 tcp (round-robin) load-balancer**
```
./loxicmd create lb 1.1.1.1 --tcp=1828:1920 --endpoints=2.2.3.4:1
```
** Please note that round-robin is default mode in loxilb    
** End-point format is specified as &lt;CIDR:weight&gt;. For round-robin, weight has no significance.

**- NAT66 (round-robin) load-balancer**
```
./loxicmd create lb  2001::1 --tcp=2020:8080 --endpoints=4ffe::1:1,5ffe::1:1,6ffe::1:1
```

**- NAT64 (round-robin) load-balancer**
```
./loxicmd create lb  2001::1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1,33.33.33.1:1
```

**- WRR (Weighted round-robin) load-balancer (Divide traffic in 40%, 40% and 20% ratio among end-points)**
```
./loxicmd create lb 20.20.20.1 --select=priority --tcp=2020:8080 --endpoints=31.31.31.1:40,32.32.32.1:40,33.33.33.1:20
```

**- Hash based load-balancer (select end-points based on traffic hash)**
```
./loxicmd create lb 20.20.20.1 --select=hash --tcp=2020:8080 --endpoints=31.31.31.1:40,32.32.32.1:40,33.33.33.1:20
```

**- Load-balancer with forceful tcp-reset session timeout after inactivity of 30s**
```
./loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=31.31.31.1:1,32.32.32.1:1,33.33.33.1:1 --inatimeout=30
```

**- Load-balancer with one-arm mode**
```
./loxicmd create lb 20.20.20.1 --tcp=2020:8080 --endpoints=100.100.100.2:1,100.100.100.3:1,100.100.100.4:1 --mode=onearm
```

**- Load-balancer with fullnat mode**
```
./loxicmd create lb 88.88.88.1 --sctp=38412:38412 --endpoints=192.168.70.3:1 --mode=fullnat
```
** For more information on  one-arm and full-nat mode, please check this [post](https://github.com/loxilb-io/loxilb/discussions/107#discussioncomment-4318418
)

5. Run loxicmd with deleting lb information
```
./loxicmd delete lb 1.1.1.1 --tcp=1828 
```

6. Run loxicmd with getting connection track information
```
./loxicmd get conntrack
```

7. Run loxicmd with getting port dumps
```
./loxicmd get port
```

For more information, use help option!
```
./loxicmd help
```

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


