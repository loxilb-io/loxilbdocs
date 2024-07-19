<link rel="stylesheet" href="stylesheets/extra.css" />

## [![loxilb logo](photos/loxilb-logo-small.png)](https://github.com/loxilb-io/loxilb) Welcome to loxilb documentation

[![eBPF Emerging App](https://img.shields.io/badge/ebpf.io-Emerging--App-success)](https://ebpf.io/projects#loxilb) [![Go Report Card](https://goreportcard.com/badge/github.com/loxilb-io/loxilb)](https://goreportcard.com/report/github.com/loxilb-io/loxilb) ![build workflow](https://github.com/loxilb-io/loxilb/actions/workflows/docker-image.yml/badge.svg) ![sanity workflow](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity.yml/badge.svg)    
![apache](https://img.shields.io/badge/license-Apache-blue.svg) ![gpl](https://img.shields.io/badge/license-BSD-blue.svg)  [![Stargazers][stars-shield]][stars-url]   
<!--[![eBPF Emerging Project](photos/ebpflogo.png)](https://ebpf.io/projects#loxilb)-->
   
[stars-shield]: https://img.shields.io/github/stars/loxilb-io??style=for-the-badge&logo=appveyor
[stars-url]: https://github.com/loxilb-io/loxilb/stargazers

## Background 
loxilb started as a project to ease deployments of cloud-native/kubernetes workloads for the edge. When we deploy services in public clouds like AWS/GCP, the services becomes easily accessible or exported to the outside world. The public cloud providers, usually by default, associate load-balancer instances for incoming requests to these services to ensure everything is quite smooth.    

However, for on-prem and edge deployments, there is no *service type - external load balancer* provider by default. For a long time, **MetalLB** originating from Google was the only choice for the needy. But edge services are a different ball game altogether due to the fact that there are so many exotic protocols in play like GTP, SCTP, SRv6 etc and integrating everything into a seamlessly working solution has been quite difficult.   

loxilb dev team was approached by many people who wanted to solve this problem. As a first step to solve the problem, it became apparent that networking stack provided by Linux kernel, although very solid,  really lacked the development process agility to quickly provide support for a wide variety of permutations and combinations of protocols and stateful load-balancing on them. Our search led us to the awesome tech developed by the Linux community - eBPF. The flexibility to introduce new functionality into the OS Kernel as a safe sandbox program was a complete fit to our design philosophy. It also does not need any dedicated CPU cores which makes it perfect for designing energy-efficient edge architectures.    

## What is loxilb

loxilb is an open source cloud-native load-balancer based on GoLang/eBPF with the goal of achieving cross-compatibity across a wide range of on-prem, public-cloud or hybrid K8s environments.   

## Kubernetes with loxilb

Kubernetes defines many service constructs like cluster-ip, node-port, load-balancer etc for pod to pod, pod to service and service from outside communication.   

<img src="photos/K8sLoxiLB.jpg" width=50% height=50%>

All these services are provided by load-balancers/proxies operating at Layer4/Layer7. Due to Kubernetes's highly modular architecture,  these services can be provided by different software modules. For example, kube-proxy is used to provide cluster-ip and node-port services by default.   

Service type load-balancer is usually provided by public cloud-provider as a managed service. But for on-prem and self-managed clusers, there are only a few good options. <b>loxilb provides service type load-balancer as its main use-case</b>.   

Additionally, loxilb can also support cluster-ip and node-port services and thereby providing end-to-end connectivity for Kubernetes.  

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

## Layer4 vs Layer7
loxilb works as a L4 load-balancer/service-mesh by default. Although it provides great performance, at times, L7 load-balancing/proxy might be necessary in K8s. There are many good L7 proxies already available for k8s. Still, we are working on providing a great L7 solution natively in eBPF. It is a tough endeavor one which should reap great benefits once completed. Please keep an eye for updates on this.

## Telco-Cloud with loxilb
For deploying telco-cloud with cloud-native functions, loxilb can be used as a SCP(service communication proxy). SCP is a communication proxy defined by [3GPP](https://www.etsi.org/deliver/etsi_ts/129500_129599/129500/16.04.00_60/ts_129500v160400p.pdf) and aimed at telco micro-services running in cloud-native environment. Telco-cloud requires load-balancing and communication across various interfaces/standards like N2, N4, E2(ORAN), S6x, 5GLAN, GTP etc. Each of these present its own unique challenges which loxilb aims to solve e.g.:    

- N4 requires PFCP level session-intelligence
- N2 requires NGAP parsing capability   
- S6x requires Diameter/SCTP multi-homing LB support   
- MEC use-cases might require UL-CL understanding   
- Hitless failover support might be essential for mission-critical applications  
- E2 might require SCTP-LB with OpenVPN bundled together
  
## Architectural Considerations   
- [Understanding loxilb modes and deployment in K8s with kube-loxilb](kube-loxilb.md)
- [Understanding High-availability with loxilb](ha-deploy.md)

## Getting started with different K8s distributions & tools   

#### loxilb as ext-cluster pod   
- [K3s : loxilb with default flannel](k3s_quick_start_flannel.md)
- [K3s : loxilb with calico](k3s_quick_start_calico.md)
- [K3s : loxilb with cilium](quick_start_with_cilium.md)
- [K0s : loxilb with default kube-router networking](k0s_quick_start.md)
- [EKS : loxilb ext-mode](eks-external.md)    

#### loxilb as in-cluster pod   
- [K3s : loxilb in-cluster mode](k3s_quick_start_incluster.md)
- [K0s : loxilb in-cluster mode](k0s_quick_start_incluster.md)
- [MicroK8s : loxilb in-cluster mode](microk8s_quick_start_incluster.md)
- [EKS : loxilb in-cluster mode](eks-external.md)     

#### loxilb as service-proxy
- [K3s : loxilb service-proxy with flannel](service-proxy-flannel.md)
- [K3s : loxilb service-proxy with calico](service-proxy-calico.md)

#### loxilb in standalone mode
- [Run loxilb standalone](standalone.md)

## Advanced Guides   
- [How-To : Service-group zones with loxilb](service-zones.md)
- [How-To : Access end-points outside K8s](ext-ep.md)
- [How-To : Deploy multi-server K3s HA with loxilb](k3s-multi-master.md)
- [How-To : Deploy loxilb with Ingress](loxilb-nginx-ingress.md)
- [How-To : Deploy loxilb with multi-AZ HA support in AWS](aws-multi-az.md)
- [How-To : Deploy loxilb with multi-cloud HA support](multi-cloud-ha.md)

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
- [API Reference - loxilb web-Api](api.md)
- [Performance Reports](perf.md)
- [Development Roadmap](roadmap.md)
- [Contribute](contribute.md)
- [System Requirements](requirements.md)
- [Frequenctly Asked Questions- FAQs](faq.md)

## Blogs
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


## Community Posts
- [5G SCTP LoadBalancer Using loxilb](https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103)   
- [5G Uplink Classifier Using loxilb](https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c)   
- [K3s - Using loxilb as external service lb](https://cloudybytes.medium.com/k3s-using-loxilb-as-external-service-lb-2ea4ce61e159)   
- [K8s - Bringing load-balancing to multus workloads with loxilb](https://cloudybytes.medium.com/k8s-bringing-load-balancing-to-multus-workloads-with-loxilb-a0746f270abe)
- [5G SCTP LoadBalancer Using LoxiLB on free5GC](https://medium.com/@ben0978327139/5g-sctp-loadbalancer-using-loxilb-applying-on-free5gc-b5c05bb723f0)
- [Kubernetes Services: Achieving optimal performance is elusive](https://cloudybytes.medium.com/kubernetes-services-achieving-optimal-performance-is-elusive-5def5183c281)

## Latest CICD Status

| Features(Ubuntu20.04) | Features(Ubuntu22.04)|  
|:----------|:-------------|
| ![build workflow](https://github.com/loxilb-io/loxilb/actions/workflows/docker-image.yml/badge.svg)  |  [![Docker-Multi-Arch](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/docker-multiarch.yml) | 
| ![simple workflow](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity.yml/badge.svg)  | [![Sanity-CI-Ubuntu-22](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity-ubuntu-22.yml) | 
| [![tcp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity.yml) | [![tcp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/tcp-sanity-ubuntu-22.yml)   |
| [![udp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity.yml) | [![udp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/udp-sanity-ubuntu-22.yml) |
| [![sctp-lb-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/sctp-sanity.yml)  | ![ipsec-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/ipsec-sanity-ubuntu-22.yml/badge.svg)  |
| ![extlb workflow](https://github.com/loxilb-io/loxilb/actions/workflows/advanced-lb-sanity.yml/badge.svg) | ![nat66-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/nat66-sanity-ubuntu-22.yml/badge.svg)  | 
| ![ipsec-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/ipsec-sanity.yml/badge.svg)   | [![Scale-Sanity-CI-Ubuntu-22](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity-ubuntu-22.yml) |
| ![scale-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/scale-sanity.yml/badge.svg)  | [![perf-CI](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml) |  [![k3s-calico-ubuntu22-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico-ubuntu-22.yml)  | | |
| [![liveness-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/liveness-sanity.yml)  | |
| ![nat66-sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/nat66-sanity.yml/badge.svg)   | | 
| [![perf-CI](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/perf.yml)  | |

| K3s Tests | K8s Cluster Tests | EKS Test |
|:-------------|:-------------|:-------------|
|[![K3s-Base-Sanity-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-base-sanity.yml/badge.svg?branch=main)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-base-sanity.yml) | [![K8s-Calico-Cluster-IPVS-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs.yml) | ![EKS](https://github.com/loxilb-io/loxilb/actions/workflows/eks.yaml/badge.svg?branch=main)  |
| [![k3s-flannel-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel.yml) | [![K8s-Calico-Cluster-IPVS2-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs2.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs2.yml) | |
| [![k3s-flannel-ubuntu22-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-ubuntu-22.yml) | [![K8s-Calico-Cluster-IPVS3-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3.yml) | |
|[![k3s-flannel-cluster-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-cluster.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-cluster.yml) | [![K8s-Calico-Cluster-IPVS3-HA-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3-ha.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k8s-calico-ipvs3-ha.yml) | |
| [![k3s-flannel-incluster-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster.yml)   |  |  |
|[![k3s-flannel-incluster-l2-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster-l2.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-flannel-incluster-l2.yml)  | | |
| [![k3s-calico-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-calico.yml)  | | |
| [![k3s-cilium-cluster-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-cilium-cluster.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-cilium-cluster.yml) | |
| [![k3s-sctpmh-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh.yml)  | | |
| [![k3s-sctpmh-ubuntu22-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-ubuntu-22.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-ubuntu22.yml) | | |
| [![k3s-sctpmh-2-CI](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-2.yml/badge.svg)](https://github.com/loxilb-io/loxilb/actions/workflows/k3s-sctpmh-2.yml)  | | |
