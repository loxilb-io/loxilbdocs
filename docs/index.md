<link rel="stylesheet" href="stylesheets/extra.css" />

## [![loxilb logo](photos/loxilb-logo-small.png)](https://github.com/loxilb-io/loxilb) Welcome to loxilb documentation

[![eBPF Emerging App](https://img.shields.io/badge/ebpf.io-Emerging--App-success)](https://ebpf.io/projects#loxilb) [![Go Report Card](https://goreportcard.com/badge/github.com/loxilb-io/loxilb)](https://goreportcard.com/report/github.com/loxilb-io/loxilb)  ![apache](https://img.shields.io/badge/license-Apache-blue.svg) ![gpl](https://img.shields.io/badge/license-BSD-blue.svg)  [![Stargazers][stars-shield]][stars-url]   
<!--[![eBPF Emerging Project](photos/ebpflogo.png)](https://ebpf.io/projects#loxilb)-->
   
[stars-shield]: https://img.shields.io/github/stars/loxilb-io??style=for-the-badge&logo=appveyor
[stars-url]: https://github.com/loxilb-io/loxilb/stargazers

## Background 
loxilb started as a project to ease deployments of cloud-native/kubernetes workloads for the edge. When we deploy services in public clouds like AWS/GCP, the services becomes easily accessible or exported to the outside world. The public cloud providers, usually by default, associate load-balancer instances for incoming requests to these services to ensure everything is quite smooth.    

However, for on-prem and edge deployments, there is no *service type - external load balancer* provider by default. For a long time, MetalLB was the only choice for the needy. But edge services are a different ball game altogether due to the fact that there are so many exotic protocols in play like GTP, SCTP, SRv6 etc and integrating everything into a seamlessly working solution has been quite difficult.   

loxilb dev team was approached by many people who wanted to solve this problem. As a first step to solve the problem, it became apparent that networking stack provided by Linux kernel, although very solid,  really lacked the development process agility to quickly provide support for a wide variety of permutations and combinations of protocols and stateful load-balancing on them. Our search led us to the awesome tech developed by the Linux community - eBPF. The flexibility to introduce new functionality into the OS Kernel as a safe sandbox program was a complete fit to our design philosophy. It also does not need any dedicated CPU cores which makes it perfect for designing energy-efficient edge architectures.    

## What is loxilb

loxilb is an open source cloud-native load-balancer based on GoLang/eBPF with the goal of achieving cross-compatibity across a wide range of on-prem, public-cloud or hybrid K8s environments. loxilb is being developed to support the adoption of cloud-native tech in telco, mobility, and edge computing.   

## Kubernetes with loxilb

Kubernetes defines many service constructs like cluster-ip, node-port, load-balancer, ingress etc for pod to pod, pod to service and outside-world to service communication.    

![loxilb cover](photos/loxilb-cover.png)

All these services are provided by load-balancers/proxies operating at Layer4/Layer7. Since Kubernetes's is highly modular, these services can be provided by different software modules. For example, kube-proxy is used by default to provide cluster-ip and node-port services. For some services like LB and Ingress, no default is usually provided.

Service type load-balancer is usually provided by public cloud-provider(s) as a managed entity. But for on-prem and self-managed clusters, there are only a few good options available. Even for provider-managed K8s like EKS, there are many who would want to bring their own LB to clusters running anywhere. Additionally, Telco 5G and edge services introduce unique challenges due to the variety of exotic protocols involved, including GTP, SCTP, SRv6, SEPP and DTLS, making seamless integration particularly challenging. loxilb provides service type load-balancer as its main use-case. loxilb can be run in-cluster or ext-to-cluster as per user need.

loxilb works as a L4 load-balancer/service-proxy by default. Although L4 load-balancing provides great performance and functionality, an equally performant L7 load-balancer is also necessary in K8s for various use-cases. loxilb also supports L7 load-balancing in the form of Kubernetes Ingress implementation which is enhanced with eBPF sockmap helpers. This also benefit users who need L4 and L7 load-balancing under the same hood.

Additionally, loxilb also supports:   
- [x] kube-proxy replacement with eBPF(full cluster-mesh implementation for Kubernetes)   
- [x] Ingress Support   
- [x] Kubernetes Gateway API
- [x] HA capable Egress for Kubernetes
- [ ] Kubernetes Network Policies (in-progress)  

## Telco-Cloud with loxilb
For deploying telco-cloud with cloud-native functions, loxilb can be used as a SCP(service communication proxy). SCP is a communication proxy defined by [3GPP](https://www.etsi.org/deliver/etsi_ts/129500_129599/129500/16.04.00_60/ts_129500v160400p.pdf) and aimed at optimizing telco micro-services running in cloud-native environment. Read more about it [here](https://dev.to/nikhilmalik/5g-service-communication-proxy-with-loxilb-4242).    

![loxilb svc](photos/scp.svg)   

Telco-cloud requires load-balancing and communication across various interfaces/standards like N2, N4, E2(ORAN), S6x, 5GLAN, GTP etc. Each of these present its own unique challenges which loxilb aims to solve e.g.:

- N4 requires PFCP level session-intelligence   
- N2 requires NGAP parsing capability(Related Blogs - [Blog-1](https://www.loxilb.io/post/ngap-load-balancing-with-loxilb), [Blog-2](https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103), [Blog-3](https://medium.com/@ben0978327139/5g-sctp-loadbalancer-using-loxilb-applying-on-free5gc-b5c05bb723f0))   
- S6x requires Diameter/SCTP multi-homing LB support(Related [Blog](https://www.loxilb.io/post/k8s-introducing-sctp-multihoming-functionality-with-loxilb))   
- MEC use-cases might require UL-CL understanding(Related [Blog](https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c))   
- Hitless failover support might be essential for mission-critical applications   
- E2 might require SCTP-LB with OpenVPN bundled together   
- SIP support is needed to enable cloud-native VOIP
- N32 requires support for Security Edge Protection Proxy(SEPP)   

## Why choose loxilb?
   
- ```Performs``` much better compared to its competitors across various architectures   
    * [Single-Node Performance](https://loxilb-io.github.io/loxilbdocs/perf-single/)  
    * [Multi-Node Performance](https://loxilb-io.github.io/loxilbdocs/perf-multi/) 
    * [Performance on ARM](https://www.loxilb.io/post/running-loxilb-on-aws-graviton2-based-ec2-instance)
    * [Short Demo on Performance](https://www.youtube.com/watch?v=MJXcM0x6IeQ)
- Utitlizes ebpf which makes it ```flexible``` as well as ```customizable```
- Advanced ```quality of service``` for workloads (per LB, per end-point or per client)
- Works with ```any``` Kubernetes distribution/CNI - k8s/k3s/k0s/kind/OpenShift + Calico/Flannel/Cilium/Weave/Multus etc
- Extensive support for ```SCTP workloads``` (with multi-homing) on k8s
- Dual stack with ```NAT66, NAT64``` support for k8s
- k8s ```multi-cluster``` support (planned 🚧)
- Runs in ```any``` cloud (public cloud/on-prem) or ```standalone``` environments

## Overall features of loxilb
- L4/NAT stateful loadbalancer
    * NAT44, NAT66, NAT64 with One-ARM, FullNAT, DSR etc
    * Support for TCP, UDP, SCTP (w/ multi-homing), QUIC, FTP, TFTP etc
- High-availability support with hitless/maglev/cgnat clustering
- Extensive and scalable end-point liveness probes for cloud-native environments
- Stateful firewalling and IPSEC/Wireguard support
- Optimized implementation for features like [Conntrack](https://thermalcircle.de/doku.php?id=blog:linux:connection_tracking_1_modules_and_hooks), QoS etc
- Full compatibility for ipvs (ipvs policies can be auto inherited)
- Policy oriented L7 proxy support - HTTP1.0, 1.1, 2.0 etc (planned 🚧)   

## Components of loxilb 
- GoLang based control plane components
- A scalable/efficient [eBPF](https://ebpf.io/) based data-path implementation
- Integrated goBGP based routing stack
- A kubernetes agent [kube-loxilb](https://github.com/loxilb-io/kube-loxilb) written in Go
  
## Architectural Considerations   
- [Understanding loxilb modes and deployment in K8s with kube-loxilb](kube-loxilb.md)
- [Understanding High-availability with loxilb](ha-deploy.md)

## Getting started with different K8s distributions & tools   

#### loxilb as ext-cluster pod   
- [K8s : loxilb ext-mode](k8s-flannel-ext.md)
- [K3s : loxilb with default flannel](k3s_quick_start_flannel.md)
- [K3s : loxilb with calico](k3s_quick_start_calico.md)
- [K3s : loxilb with cilium](quick_start_with_cilium.md)
- [K0s : loxilb with default kube-router networking](k0s_quick_start.md)
- [EKS : loxilb ext-mode](eks-external.md)    

#### loxilb as in-cluster pod   
- [K8s : loxilb in-cluster mode](k8s-flannel-incluster.md)
- [K8s : loxilb in-cluster mode with cilium](cilium-incluster.md)
- [K3s : loxilb in-cluster mode](k3s_quick_start_incluster.md)
- [K0s : loxilb in-cluster mode](k0s_quick_start_incluster.md)
- [MicroK8s : loxilb in-cluster mode](microk8s_quick_start_incluster.md)
- [EKS : loxilb in-cluster mode](eks-incluster.md)
- [RedHat OCP : loxilb in-cluster mode](rhocp-quickstart-incluster.md)
- [K8s : loxilb in-cluster with calico/multus](calico-incluster-multus.md)

#### loxilb as service-proxy
- [loxilb service-proxy plugin with flannel](service-proxy-flannel.md)
- [loxilb service-proxy plugin with calico](service-proxy-calico.md)

#### loxilb as Kubernetes Ingress
- [K3s: How to run loxilb-ingress](loxilb-ingress.md)

#### loxilb as Kubernetes Egress
- [How-To : HA egress with loxilb](loxilb-egress.md)
  
#### loxilb in standalone mode
- [Run loxilb standalone](standalone.md)

## Advanced Guides   
- [How-To : Service-group zones with loxilb](service-zones.md)
- [How-To : Access end-points outside K8s](ext-ep.md)
- [How-To : Deploy multi-server K3s HA with loxilb](k3s-multi-master.md)
- [How-To : Deploy loxilb with multi-AZ HA support in AWS](aws-multi-az.md)
- [How-To : Deploy loxilb with multi-cloud HA support](multi-cloud-ha.md)
- [How-To : Deploy loxilb with ingress-nginx](loxilb-nginx-ingress.md)
- [How-To : Run loxilb in-cluster with secondary networks](loxilb-incluster-multus.md)
- [How-To : Kubernetes virtual cluster setup with k3k and loxilb](k3k-virtual-cluster.md)
- [How-To : Kubernetes service sharding with loxilb](service-sharding.md)
- [How-To : loxilb L4/L7 Load-Balancing with Kubernetes Gateway API](gw-api.md)
- [How-To : Use proxy protocol v2 with loxilb](proxy-protocol-v2.md)

## Knowledge-Base   
- [What is eBPF](ebpf.md)
- [What is k8s service - load-balancer](lb.md)
- [Architecture in brief](arch.md)
- [Code organization](code.md)
- [eBPF internals of loxilb](loxilbebpf.md)
- [What are loxilb NAT Modes](nat.md)
- [loxilb load-balancer algorithms](lb-algo.md)
- [Manual steps to build/run](run.md)
- [Debugging loxilb](debugging.md)
- [loxicmd command-line tool usage](cmd.md)
- [Developer's guide to loxicmd](cmd-dev.md)
- [Developer's guide to loxilb API](api-dev.md)
- [HTTPS guide for loxilb API](https.md)
- [API Reference - loxilb web-Api](api.md)
- [Performance Reports](perf.md)
- [Development Roadmap](roadmap.md)
- [Contribute](contribute.md)
- [System Requirements](requirements.md)
- [Frequenctly Asked Questions- FAQs](faq.md)

## Blogs
- [Building a resilient EKS Cluster with in-cluster auto-scaled LoxiLB](https://www.loxilb.io/post/build-a-high-performance-eks-cluster-using-auto-scaled-loxilb)
- [K8s - Elevating cluster networking](https://www.loxilb.io/post/loxilb-cluster-networking-elevating-k8s-networking-capabilities)    
- [eBPF - Map sync using Go](https://www.loxilb.io/post/state-synchronization-of-ebpf-maps-using-go-a-tale-of-two-frameworks)    
- [K8s in-cluster service LB with LoxiLB](https://www.loxilb.io/post/k8s-nuances-of-in-cluster-external-service-lb-with-loxilb)   
- [K8s - Introducing SCTP Multihoming with LoxiLB](https://www.loxilb.io/post/k8s-introducing-sctp-multihoming-functionality-with-loxilb)   
- [Load-balancer performance comparison on Amazon Graviton2](https://www.loxilb.io/post/running-loxilb-on-aws-graviton2-based-ec2-instance)   
- [Hyperscale anycast load balancing with HA](https://www.loxilb.io/post/loxilb-anycast-service-load-balancing-with-high-availability)   
- [Getting started with loxilb on Amazon EKS](https://www.loxilb.io/post/loxilb-load-balancer-setup-on-eks)   
- [K8s - Deploying "hitless" Load-balancing](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing)   
- [Oracle Cloud - Hitless HA load balancing](https://www.loxilb.io/post/oracle-cloud-bring-your-own-lb-for-self-managed-clusters)
- [Ipv6 migration in Kubernetes made easy](https://www.loxilb.io/post/k8s-exposing-ipv4-services-externally-as-ipv6)
- [Measuring Performance in L4-L7 Load Balancing](https://dev.to/nikhilmalik/l4-l7-performance-comparing-loxilb-metallb-nginx-haproxy-1eh0)
- [Setup K8s with HA capable egress](https://dev.to/trekkiecoder/setup-a-multi-node-k3s-setup-with-ha-capable-egress-1h4m)

## Community Posts
- [5G SCTP LoadBalancer Using loxilb](https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103)   
- [5G Uplink Classifier Using loxilb](https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c)   
- [K3s - Using loxilb as external service lb](https://cloudybytes.medium.com/k3s-using-loxilb-as-external-service-lb-2ea4ce61e159)   
- [K8s - Bringing load-balancing to multus workloads with loxilb](https://cloudybytes.medium.com/k8s-bringing-load-balancing-to-multus-workloads-with-loxilb-a0746f270abe)
- [5G SCTP LoadBalancer Using LoxiLB on free5GC](https://medium.com/@ben0978327139/5g-sctp-loadbalancer-using-loxilb-applying-on-free5gc-b5c05bb723f0)
- [Kubernetes Services: Achieving optimal performance is elusive](https://cloudybytes.medium.com/kubernetes-services-achieving-optimal-performance-is-elusive-5def5183c281)
- [LoxiLB: Another Load Balancer](https://blog.devgenius.io/loxilb-another-load-balancer-aabf4a29d810)
- [LoxiLB eBPF deep-dive](https://free5gc.org/blog/20241203/20241203)

## Research Papers (featuring loxilb)
- [Mitigating Spectre-PHT using Speculation Barriers in Linux BPF](https://arxiv.org/pdf/2405.00078)   

## Latest CICD Status
<div class="grid cards" markdown>

-   __Features(Ubuntu20.04)__

    ---
    ![build workflow](https://github.com/loxilb-io/loxilb/actions/workflows/docker-image.yml/badge.svg)    
    ![simple workflow](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity.yml/badge.svg)    
    [![tcp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity.yml)    
    [![udp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity.yml)    
    [![sctp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity.yml)    
    ![extlb workflow](https://github.com/loxilb-io/loxilb/actions/workflows/advanced-lb-sanity.yml/badge.svg)    
    ![nat66-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/nat66-sanity.yml/badge.svg)   
    ![ipsec-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/ipsec-sanity.yml/badge.svg)    
    [![liveness-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity.yml)       
    ![scale-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity.yml/badge.svg)     
    [![perf-CI](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml)      

-   __Features(Ubuntu22.04)__

    ---
    [![Docker-Multi-Arch](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml)    
    [![Sanity-CI-Ubuntu-22](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-ubuntu-22.yml)    
    [![tcp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-ubuntu-22.yml)    
    [![udp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-ubuntu-22.yml)    
    [![SCTP-LB-Sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity-ubuntu-22.yml)     
    ![extlb workflow](https://github.com/loxilb-io/loxilb/actions/workflows/advanced-lb-sanity-ubuntu-22.yml/badge.svg)       
    ![nat66-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/nat66-sanity-ubuntu-22.yml/badge.svg)     
    ![ipsec-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/ipsec-sanity-ubuntu-22.yml/badge.svg)     
    [![liveness-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity-ubuntu-22.yml)     
    [![Scale-Sanity-CI-Ubuntu-22](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity-ubuntu-22.yml)     
    [![perf-CI](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml)      
    [![k3s-calico-ubuntu22-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico-ubuntu-22.yml)    

-   __Features(Ubuntu24.04)__

    ---
    [![Docker-Multi-Arch](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml)    
    [![Sanity-CI-Ubuntu-24](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-ubuntu-24.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-ubuntu-24.yml)    
    [![tcp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-ubuntu-24.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-ubuntu-24.yml)    
    [![udp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-ubuntu-24.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-ubuntu-24.yml)       
    [![SCTP-LB-Sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity-ubuntu-24.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity-ubuntu-24.yml)     
    ![extlb workflow](https://github.com/loxilb-io/loxilb/actions/workflows/advanced-lb-sanity-ubuntu-24.yml/badge.svg)      
    ![nat66-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/nat66-sanity-ubuntu-24.yml/badge.svg)     
    ![ipsec-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/ipsec-sanity-ubuntu-24.yml/badge.svg)     
    [![liveness-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity-ubuntu-24.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity-ubuntu-24.yml)     
    [![Scale-Sanity-CI-Ubuntu-22](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity-ubuntu-24.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity-ubuntu-24.yml)      
    [![perf-CI](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml)      

-   __Features(RedHat9)__

    ---
    [![Docker-Multi-Arch](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml)      
    [![Sanity-CI-RH9](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-rh9.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-rh9.yml)      
    [![tcp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-rh9.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-rh9.yml)      
    [![udp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-rh9.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-rh9.yml)      
    [![SCTP-LB-Sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity-rh9.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity-rh9.yml)      
    ![extlb workflow](https://github.com/loxilb-io/loxilb/actions/workflows/advanced-lb-sanity-rh9.yml/badge.svg)      
    ![nat66-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/nat66-sanity-rh9.yml/badge.svg)      
    ![ipsec-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/ipsec-sanity-rh9.yml/badge.svg)      
    [![liveness-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity-rh9.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity-rh9.yml)       
    
-   __K3s Tests__

    ---
    [![K3s-Base-Sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-base-sanity.yml/badge.svg?branch=main)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-base-sanity.yml)     
    [![k3s-flannel-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel.yml)     
    [![k3s-flannel-ubuntu22-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-ubuntu-22.yml)     
    [![k3s-flannel-cluster-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-cluster.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-cluster.yml)     
    [![k3s-flannel-incluster-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster.yml)     
    [![k3s-flannel-incluster-l2-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster-l2.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster-l2.yml)     
    [![k3s-calico-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico.yml)     
    [![k3s-cilium-cluster-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-cilium-cluster.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-cilium-cluster.yml)      
    [![k3s-sctpmh-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh.yml)     
    [![k3s-sctpmh-ubuntu22-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-ubuntu22.yml)      
    [![k3s-sctpmh-2-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-2.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-2.yml)     

-   __K8s Cluster Tests__

    ---
    [![K8s-Calico-Cluster-IPVS-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs.yml)     
    [![K8s-Calico-Cluster-IPVS2-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs2.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs2.yml)     
    [![K8s-Calico-Cluster-IPVS3-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3.yml)      
    [![K8s-Calico-Cluster-IPVS3-HA-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3-ha.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3-ha.yml)     
     

-   __EKS Test__

    ---
    ![EKS](https://github.com/loxilb-io/loxilb/actions/workflows/eks.yaml/badge.svg?branch=main)   
     
    
</div>

