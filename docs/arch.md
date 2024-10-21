# loxilb architecture and modules

loxilb consists of the following modules :  

- <b>kube-loxilb</b>

   [kube-loxilb](https://github.com/loxilb-io/kube-loxilb) is loxilb's implementation of kubernetes service load-balancer spec which includes support for load-balancer class, advanced IPAM (shared or 
   exclusive) etc. kube-loxilb runs as a deloyment set in kube-system namespace. It is a control-plane component that always runs inside k8s cluster and watches k8s system for changes to nodes/end- 
   points/reachability/LB services etc. It acts as a K8s [Operator](https://www.cncf.io/blog/2022/06/15/kubernetes-operators-what-are-they-some-examples/) of [loxilb](https://github.com/loxilb-io/loxilb). The <b>loxilb</b> component takes care of doing actual job of providing service connectivity and load-balancing. So, from deployment perspective we need to run <b>kube-loxilb</b> inside K8s 
   cluster but we have option to deploy <b>loxilb</b> in-cluster or external to the cluster.
  
- <b>loxicmd</b>

  [loxicmd](https://github.com/loxilb-io/loxicmd) is command line tool  to configure and dump loxilb information which is based on same foundation as the wildly popular kubectl tool.
  
- <b>loxilb</b>

  loxilb is a modern goLang based framework (process) which mantains information coming in from various sources e.g apiserver and populates the eBPF maps used by the loxilb eBPF kernel. It is also responsible for loading eBPF programs to the interfaces.It also acts as a client to goBGP to exchange routes based on information from loxilb CCM. Last but not the least, it will be finally responsible for maintaining HA state sync with its remote peers. Almost all serious lb implementations need to be deployed as a HA cluster.
  
- <b>loxilb eBPF kernel</b>

  eBPF kernel module implements the data-plane of loxilb which provides complete kernel bypass. It is a fully self contained and feature-rich stack able to process packets from rx to tx without invoking linux native kernel networking.
  
- <b>goBGP</b>

  Although goBGP is a separate project, loxilb has adopted and integrated with goBGP as its routing stack of choice. We have also developed various features for this awesome project which are planned for upstreaming.

- <b>DashBoards</b>

  Grafana based dashboards to provide highly dynamic insight into loxilb state.
  
The following is a typical loxilb deployment topology (HA is omitted here for simplicity) : 

![image](https://github.com/user-attachments/assets/86fa40b0-aee9-48da-82ed-f13b955a7be1)

