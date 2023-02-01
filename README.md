# loxilb Documentation
[![eBPF Emerging Project](https://img.shields.io/badge/ebpf.io-Emerging--Project-success)](https://ebpf.io/projects#loxilb) ![build workflow](https://github.com/loxilb-io/loxilb/actions/workflows/docker-image.yml/badge.svg) ![sanity workflow](https://github.com/loxilb-io/loxilb/actions/workflows/basic-sanity.yml/badge.svg) ![sanity workflow](https://github.com/loxilb-io/loxilbdocs/actions/workflows/documentation.yml/badge.svg)    

![apache](https://img.shields.io/badge/license-Apache-blue.svg) ![bsd](https://img.shields.io/badge/license-BSD-blue.svg) ![gpl](https://img.shields.io/badge/license-GPL-blue.svg)

## loxilb Background 
loxilb started as a project to ease deployments of cloud-native/kubernetes workloads for the edge. When we deploy services in public clouds like AWS/GCP, the services becomes easily accessible or exported to the outside world. The public cloud providers, usually by default, associate load-balancer instances for incoming requests to these services to ensure everything is quite smooth. 

However, for on-prem and edge deployments, there is no *service type - external load balancer* provider by default. For a long time, [MetalLB](https://metallb.universe.tf/) was the only choice for the needy. But edge services are a different ball game altogether due to the fact that there are so many exotic protocols in play like GTP, SCTP, SRv6 etc and integrating everything into a seamlessly working solution has been quite difficult. These services are also increasingly in demand in public clouds as well. 

loxilb dev team was approached by many people who wanted to solve this problem. As a first step to solve the problem, it became apparent that networking stack provided by Linux kernel, although very solid,  really lacked the development process agility to quickly provide support for a wide variety of permutations and combinations of protocols and stateful load-balancing on them. Our search led us to the awesome tech developed by the Linux community - eBPF. The flexibility to introduce new functionality into Kernel as a sandbox program was a complete fit to our design philosophy. Although we did consider DPDK for a while, but the fact that it needs dedicated cores/CPUs really defeats the whole purpose of making energy-efficient edge architectures.

During the journey of loxilb's development, we developed many other generic networking/security/visibility features in it using eBPF which can be used for various other purposes not specific to load-balancer only. But we decided to stick to our original name *loxilb* as load-balancing will continue to be its main purpose in the forseeable future. loxilb team hopes the open-source community finds it helpful.

## loxilb Aim/Goals

loxilb aims to provide the following :

-  Service type external load-balancer for kubernetes
-  L4/NAT stateful loadbalancer
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
-  GoLang based control plane components (Apache license)
-  Seamless integration with goBGP based routing stack
-  Easy to use APIs/Interfaces for developers
-  Cloud-Native Network Function (CNF) form-factor by default

## Documentation

- [What is eBPF](docs/ebpf.md)
- [What is k8s service type - external load-balancer](docs/lb.md)
- [Architecture in brief](docs/arch.md)
- [Code organization](docs/code.md)
- [eBPF internals of loxilb](docs/loxilbebpf.md)
- [Howto - build/run](docs/run.md)
- [Howto - kube-loxilb](docs/kube-loxilb.md)
- [Howto - ccm plugin](docs/ccm.md)
- [Howto - debug](docs/debugging.md)
- [Howto - loxilb with calico bgp](docs/integrate_bgp_eng.md)
- [What are loxilb NAT Modes](docs/nat.md)
- [Cmd/Config guide](docs/cmd.md)
- [Api usage/dev guide](docs/api.md)
- [Performance](docs/perf.md)
- [Development Roadmap](docs/roadmap.md)
- [Contribute](docs/contribute.md)
- [FAQs](docs/faq.md)

## Blogs
- [5G SCTP LoadBalancer Using LoxiLB](https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103)
- [5G Uplink Classifier Using Loxilb](https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c)
- [K3s: Using loxilb as external service lb](https://cloudybytes.medium.com/k3s-using-loxilb-as-external-service-lb-2ea4ce61e159)
- [K8s - Deploying "hitless" Load-balancing](https://www.loxilb.io/post/k8s-deploying-hitless-and-ha-load-balancing)

### Host OS requirements
To install LoxiLB software packages, you need the 64-bit version of one of these OS versions:
* Ubuntu 20.04(LTS)
* Ubuntu 22.04(LTS)
* Fedora 36
* RockyOS
* Enterprise Redhat (Planned)
* Windows Server(Planned)

### Linux Kernel Requirements
* Linux Kernel Version >= 5.4.0

### Compatible Kubernetes Versions
* Kubernetes 1.19
* Kubernetes 1.20
* Kubernetes 1.21
* Kubernetes 1.22

### Hardware Requirements
* None as long as above criteria are met
