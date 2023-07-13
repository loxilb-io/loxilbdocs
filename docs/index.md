## [![loxilb logo](photos/loxilb-logo-small.png)](https://github.com/loxilb-io/loxilb) Welcome to loxilb-docs

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

loxilb is an open source hyper-scale software load-balancer for cloud-native workloads. It uses eBPF as its core-engine and is based on Golang. It is designed to power on-premise, edge and public-cloud Kubernetes cluster deployments.   

###  🚀 loxilb aims to provide the following :   
- Service type load-balancer for kubernetes    
    * L4/NAT stateful loadbalancer    
    * NAT44, NAT66, NAT64 with One-ARM, FullNAT, DSR etc    
    * Support for TCP, UDP, SCTP (w/ multi-homing), FTP, TFTP etc    
    * High-availability support with hitless/maglev clustering    
    * Full compliance for K8s loadbalancer Spec    
    * Multi-cluster support       
-  Extensive and scalable liveness probes for cloud-native environments    
-  High-perf replacement for the *aging* iptables/ipvs    
-  L7 proxy support    
-  Telco/5G/6G friendly features    
    * GTP tunnels as first class citizens     
    * Optimized SRv6 implementation    
    * Support for UL-CL with LB, QFI and other utility extensions    

### 🧿 loxilb is composed of:        
- Bespoke GoLang based control plane components     
- [eBPF](https://ebpf.io/) based data-path forwarding    
   * Home-grown stack with advanced features like [Conntrack](https://thermalcircle.de/doku.php?id=blog:linux:connection_tracking_1_modules_and_hooks), QoS etc    
   * Complete kernel networking bypass    
   * Highly scalable with low-latency & high-throughput    
- GoLang powered easy to use APIs/Interfaces/CLI infra    
- Seamless integration with goBGP based routing stack    

### 📦 Why choose loxilb?    
   
- ```Performs``` much better compared to its competitors across various architectures    
    * [Single-Node Performance](https://loxilb-io.github.io/loxilbdocs/perf-single/)    
    * [Multi-Node Performance](https://loxilb-io.github.io/loxilbdocs/perf-multi/)    
    * [Performance on ARM](https://www.loxilb.io/post/running-loxilb-on-aws-graviton2-based-ec2-instance)    
    * [Short Demo on Performance](https://www.youtube.com/watch?v=MJXcM0x6IeQ)    
- ebpf makes it ```flexible``` and ```future-proof``` (kernel version agnostic and in future OS agnostic 🚧)      
- Advanced quality of service for workloads (per LB, per end-point or per client)       
- Includes powerful NG ```stateful firewalling``` and ```IPSEC/Wireguard``` support      
- Optimized/Custom end-point ```liveness checks at scale```      
- Support for ```5G/Edge```  cloud-native workloads     
- Works with ```any``` Kubernetes distribution/CNI - k8s/k3s/k0s/kind/OpenShift + Calico/Flannel/Cilium/Weave/Multus etc      
- Extensive support for ```SCTP workloads``` (with multi-homing) on k8s    
- Dual stack with ```NAT66, NAT64``` support for k8s     
- k8s ```multi-cluster``` support 🚧      
- Runs in ```any``` cloud : public cloud (EKS), on-prem or multi-cloud environments        

  (*🚧: *Work in progress*)      

## How-To Guides

- [How-To : build/run](run.md)
- [How-To : configuration](cmd.md)
- [How-To : kube-loxilb](kube-loxilb.md)
- [How-To : ccm plugin](ccm.md)
- [How-To : debug](debugging.md)
- [How-To : loxilb with calico bgp](integrate_bgp_eng.md)
- [How-To : loxilb in standalone mode](standalone.md)
- [How-To : loxilb Web-APIs](api.md)

## Knowledge-Base   
- [What is eBPF](ebpf.md)
- [What is k8s service - load-balancer](lb.md)
- [Architecture in brief](arch.md)
- [Code organization](code.md)
- [eBPF internals of loxilb](loxilbebpf.md)
- [What are loxilb NAT Modes](nat.md)
- [Developer's guide to loxicmd](cmd-dev.md)
- [Developer's guide to loxilb API](api-dev.md)
- [Performance Reports](perf.md)
- [Development Roadmap](roadmap.md)
- [Contribute](contribute.md)
- [System Requirements](requirements.md)
- [Frequenctly Asked Questions- FAQs](faq.md)

## Blogs
- [K8s - Introducing SCTP Multihoming with LoxiLB](https://www.loxilb.io/post/k8s-introducing-sctp-multihoming-functionality-with-loxilb)   
- [Load-balancer performance comparison on Amazon Graviton2](https://www.loxilb.io/post/running-loxilb-on-aws-graviton2-based-ec2-instance)   
- [Hyperscale anycast load balancing with HA](https://www.loxilb.io/post/loxilb-anycast-service-load-balancing-with-high-availability)   
- [Getting started with loxilb on Amazon EKS](https://www.loxilb.io/post/loxilb-load-balancer-setup-on-eks)   
- [K8s - Deploying "hitless" Load-balancing](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing)   
- [Ipv6 migration in Kubernetes made easy](https://www.loxilb.io/post/k8s-exposing-ipv4-services-externally-as-ipv6)   

## Community Posts
- [5G SCTP LoadBalancer Using loxilb](https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103)   
- [5G Uplink Classifier Using loxilb](https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c)   
- [K3s - Using loxilb as external service lb](https://cloudybytes.medium.com/k3s-using-loxilb-as-external-service-lb-2ea4ce61e159)   
- [K8s - Bringing load-balancing to multus workloads with loxilb](https://cloudybytes.medium.com/k8s-bringing-load-balancing-to-multus-workloads-with-loxilb-a0746f270abe)
- [5G SCTP LoadBalancer Using LoxiLB on free5GC](https://medium.com/@ben0978327139/5g-sctp-loadbalancer-using-loxilb-applying-on-free5gc-b5c05bb723f0)
