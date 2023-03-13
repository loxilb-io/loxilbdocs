## Bare-Metal Performance

The topology for this test is as follows :

```mermaid
graph LR;
    A[100.100.100.1]-->B[loxilb VIP-20.20.20.1];
    B-->C[31.31.31.1];
    B-->D[32.32.32.1];
    B-->E[33.33.33.1];
```

 In this test, all the hosts, end-points and load-balancer run in separate dedicated servers. Server specs used - 
*Intel(R) Xeon(R) Silver 4210R CPU @ 2.40GHz - 40 core RAM 125GB, Kernel 5.15.0-52-generic*. The following command can be used to configure loxilb for the given topology:

```
# loxicmd create lb 20.20.20.1 --tcp=2020:5001 --endpoints=31.31.31.1:1,32.32.32.1:1,33.33.33.1:1
```

The default mode of LoxiLB is RR(round-robin) and the traffic distribution will be as follows:

```mermaid
    
sequenceDiagram
    participant ClientB1
    participant ClientB2
    participant ClientB3
    participant loxilb
    participant ServiceB_PodX
    participant ServiceB_PodY
    participant ServiceB_PodZ
    ClientB1->>loxilb: 100.100.100.1->20.20.20.1
    ClientB2->>loxilb: 101.101.101.1->20.20.20.1
    ClientB3->>loxilb: 102.102.102.1->20.20.20.1
    Note right of loxilb: Distribute the traffic in Round Robin fashion(default)!
    loxilb-->>ServiceB_PodX: 100.100.100.1->31.31.31.1
    loxilb-->>ServiceB_PodY: 101.101.101.1->32.32.32.1
    loxilb-->>ServiceB_PodZ: 102.102.102.1->33.33.33.1
    
```

We run popular tool **[netperf](https://github.com/HewlettPackard/netperf)** for the above topology. A quick explanation of terminologies used :

**RPS** - requests per seconds. Given a fixed number of connections, this denotes how many requests/message per second can be supported    
**CPS** - connections per second. This denotes how many new TCP connection setup/teardowns can be supported per second and hence one of the most important indicators of load-balancer performance     
**CRR** - connect/request/response. This is same as CPS but netperf tool uses this term to refer to CPS as part of its test scenario       
**RR** - request/response. This is another netperf test option. We used it to measure min and avg latency   

We are comparing **loxilb** with **ipvs** and **haproxy**. 

The results are as follows :

### Connections per second (TCP_CRR)
![Connections per second](photos/netperf_cps.png)

### Requests per second (TCP_RR)
![Requests per second](photos/netperf_rps.png)

### Minimum Latency
![Minimum Latency](photos/netperf_min_lat.png)

### Average Latency
![Average Latency](photos/netperf_avg_lat.png)

#### Conclusion/Notes -   

* loxilb provides enhanced performance across the spectrum of tests.  There is a noticeable gain in CPS.
* loxilb's CPS scales linearly with number of cores
* haproxy version used - 2.0.29
* netperf test scripts can be found [here](https://github.com/loxilb-io/loxilb/tree/main/cicd/tcplbcps)
