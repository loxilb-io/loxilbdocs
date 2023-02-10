
## loxicmd development guide

This guide should help developers extend and enhance loxicmd. The guide is divided into three main stages: design, development, and testing. Start with cloning the loxicmd git:

```
git clone git@github.com:loxilb-io/loxicmd.git
```

### API check and command design
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

### Add structure in pkg/api and register method (example of connection track API)
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
### Add get, create, delete functions within cmd
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
### Register command in cmd
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

### Build & Test
```
make
```
