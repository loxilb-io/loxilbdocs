## Welcome to loxilb-docs

[![eBPF Emerging App](https://img.shields.io/badge/ebpf.io-Emerging--App-success)](https://ebpf.io/projects#loxilb) [![Go Report Card](https://goreportcard.com/badge/github.com/loxilb-io/loxilb)](https://goreportcard.com/report/github.com/loxilb-io/loxilb) ![build workflow](https://github.com/loxilb-io/loxilb/actions/workflows/docker-image.yml/badge.svg) ![sanity workflow](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity.yml/badge.svg) ![apache](https://img.shields.io/badge/license-Apache-blue.svg) ![gpl](https://img.shields.io/badge/license-BSD-blue.svg)  [![Stargazers][stars-shield]][stars-url]
  
[stars-shield]: https://img.shields.io/github/stars/loxilb-io??style=for-the-badge&logo=appveyor
[stars-url]: https://github.com/loxilb-io/loxilb/stargazers

## Background 
loxilb started as a project to ease deployments of cloud-native/kubernetes workloads for the edge. When we deploy services in public clouds like AWS/GCP, the services becomes easily accessible or exported to the outside world. The public cloud providers, usually by default, associate load-balancer instances for incoming requests to these services to ensure everything is quite smooth. 

However, for on-prem and edge deployments, there is no *service type - external load balancer* provider by default. For a long time, [MetalLB](https://metallb.universe.tf/) from Google was the only choice for the needy. But edge services are a different ball game altogether due to the fact that there are so many exotic protocols in play like GTP, SCTP, SRv6 etc and integrating everything into a seamlessly working solution has been quite difficult.

loxilb dev team was approached by many people who wanted to solve this problem. As a first step to solve the problem, it became apparent that networking stack provided by Linux kernel, although very solid,  really lacked the development process agility to quickly provide support for a wide variety of permutations and combinations of protocols and stateful load-balancing on them. Our search led us to the awesome tech developed by the Linux community - eBPF. The flexibility to introduce new functionality into Kernel as a sandbox program was a complete fit to our design philosophy. Although we did consider DPDK for a while, but the fact that it needs dedicated cores/CPUs really defeats the whole purpose of making energy-efficient edge architectures.

During the journey of loxilb's development, we developed many other generic networking/security/visibility features in it using eBPF which can be used for various other purposes not specific to load-balancer only. But we decided to stick our original name *loxilb* as load-balancing will continue to be its main purpose in the forseeable future. loxilb team hopes the open-source community finds it helpful.

## Aim/Goals

loxilb aims to provide the following :

- Service type external load-balancer for kubernetes
- L4/NAT stateful loadbalancer
   - *NAT44, NAT66, NAT64 with One-ARM, FullNAT, DSR etc*
   - *High-availability support*
   - *Full compliance for K8s loadbalancer Spec*
   - *High perf drop-in replacement for iptables/ipvs*
-  Optimized SRv6 implementation in eBPF  
-  L7 Proxy support  
-  Make GTP tunnels first class citizens of the Linux world  
    - *Support for QFI and other extension headers*    
-  eBPF based data-path forwarding (Dual BSD/GPLv2 license)
    - *Complete kernel networking bypass with home-grown stack for advanced features like [Conntrack](https://thermalcircle.de/doku.php?id=blog:linux:connection_tracking_1_modules_and_hooks), QoS etc* 
    - *Highly scalable with low-latency & high througput*    
-  goLang based control plane components (Apache license)  
-  Seamless integration with goBGP based routing stack  
-  Easy to use APIs/Interfaces for developers
-  Cloud-Native Network Function (CNF) form-factor by default  

## How-To Guides

- [How-To : build/run](run.md)
- [How-To : configuration](cmd.md)
- [How-To : kube-loxilb](docs/kube-loxilb.md)
- [How-To : ccm plugin](ccm.md)
- [How-To : debug](debugging.md)
- [How-To : loxilb with calico bgp](integrate_bgp_eng.md)
- [How-To : loxilb in standalone mode](standalone.md)

## Knowledge-Base   
- [What is eBPF](ebpf.md)
- [What is k8s service - load-balancer](lb.md)
- [Architecture in brief](arch.md)
- [Code organization](code.md)
- [eBPF internals of loxilb](loxilbebpf.md)
- [What are loxilb NAT Modes](nat.md)
- [Developer's guide to loxicmd](cmd-dev.md)
- [Developer's guide to loxilb API](api-dev.md)
- [Performance](perf.md)
- [Development Roadmap](roadmap.md)
- [Contribute](contribute.md)
- [System Requirements](requirements.md)
- [Frequenctly Asked Questions- FAQs](faq.md)

## Blogs
- [5G SCTP LoadBalancer Using LoxiLB](https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103)
- [5G Uplink Classifier Using Loxilb](https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c)
- [K3s: Using loxilb as external service lb](https://cloudybytes.medium.com/k3s-using-loxilb-as-external-service-lb-2ea4ce61e159)
- [K8s - Deploying "hitless" Load-balancing](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing)
- [Ipv6 migration in Kubernetes made easy](https://www.loxilb.io/post/k8s-exposing-ipv4-services-externally-as-ipv6)   
