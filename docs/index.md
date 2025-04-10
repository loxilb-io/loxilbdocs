<link rel="stylesheet" href="stylesheets/extra.css" />

## [![loxilb logo](photos/loxilb-logo-small.png)](https://github.com/loxilb-io/loxilb) Welcome to loxilb documentation

[![eBPF Emerging App](https://img.shields.io/badge/ebpf.io-Emerging--App-success)](https://ebpf.io/projects#loxilb) [![Go Report Card](https://goreportcard.com/badge/github.com/loxilb-io/loxilb)](https://goreportcard.com/report/github.com/loxilb-io/loxilb)  ![apache](https://img.shields.io/badge/license-Apache-blue.svg) ![gpl](https://img.shields.io/badge/license-BSD-blue.svg)  [![Stargazers][stars-shield]][stars-url]   
<!--[![eBPF Emerging Project](photos/ebpflogo.png)](https://ebpf.io/projects#loxilb)-->
   
[stars-shield]: https://img.shields.io/github/stars/loxilb-io??style=for-the-badge&logo=appveyor
[stars-url]: https://github.com/loxilb-io/loxilb/stargazers

## Background 

**loxilb** began as a project to simplify the deployment of cloud-native and Kubernetes workloads at the edge. In public cloud environments like AWS or GCP, exposing services to the outside world is often effortless. These platforms typically provide external load balancers by default, ensuring smooth and efficient handling of inbound traffic.

In contrast, on-premise and edge deployments face a different challenge. There’s no native equivalent of the `ServiceType=LoadBalancer` in these environments. For a long time, **MetalLB** was the only practical option. However, edge workloads come with their own unique set of complexities. These workloads frequently rely on specialized protocols like QUIC, SCTP, and SRv6, and they operate under tighter latency budgets and stricter resource constraints. Building a seamless and reliable solution in such an environment requires deep protocol awareness and careful design.

As more users approached us to solve these challenges, it became clear that the traditional Linux networking stack, while mature and reliable, lacked the agility to rapidly support the wide variety of protocols and stateful load-balancing scenarios needed at the edge.

This led us to adopt **eBPF**, a powerful technology from the Linux community. eBPF enables safely running sandboxed programs within the kernel, allowing us to extend system functionality without altering kernel source code or introducing performance overhead. It provides the flexibility and performance essential for modern networking use cases — without requiring dedicated CPU cores — making it an ideal foundation for lightweight, energy-efficient edge architectures.


## What is loxilb

**loxilb** is an open-source, cloud-native load balancer built using **Go** and **eBPF**, designed to offer seamless cross-compatibility across on-premise, public cloud, and hybrid Kubernetes environments.

Its primary goal is to enable efficient and flexible load balancing for next-generation workloads, especially in complex deployment scenarios. Whether it's telecom, mobility, or edge computing, loxilb is built to support the growing adoption of cloud-native technologies in these demanding sectors.

By leveraging the power of eBPF, loxilb delivers high-performance networking with deep protocol awareness, minimal resource overhead, and a modern, extensible design. This makes it an ideal fit for both traditional and emerging infrastructure needs.

## Kubernetes with loxilb

Kubernetes defines several service constructs such as `ClusterIP`, `NodePort`, `LoadBalancer`, and `Ingress` to handle different types of communication — from pod-to-pod and pod-to-service, to service access from the outside world.

![loxilb cover](photos/loxilb-cover.png)

All of these service types rely on load balancers or proxies operating at Layer 4 or Layer 7. Thanks to Kubernetes’ modular architecture, these services can be delivered by different software components. For instance, `kube-proxy` is commonly used to provide `ClusterIP` and `NodePort` functionality. However, for services like `LoadBalancer` and `Ingress`, Kubernetes typically does not include a default provider.

The `LoadBalancer` service type is usually managed by public cloud providers as part of their infrastructure. In on-premise or self-managed clusters, there are limited options available. Even in managed Kubernetes environments like Amazon EKS, many users prefer to bring their own load balancer for more control, customization, or advanced protocol support.

This need becomes even more critical in domains like 5G telco and edge computing, where protocols such as **GTP**, **SCTP**, **SRv6**, **SEPP**, and **DTLS** are involved. These use cases introduce complex requirements that demand flexibility and high performance. loxilb is designed to meet these challenges, with support for the `LoadBalancer` service type as a core capability. It can be deployed either within the cluster or externally, depending on user requirements.

By default, loxilb acts as a Layer 4 load balancer and service proxy. While L4 load balancing offers high performance and broad protocol coverage, Kubernetes also benefits from robust Layer 7 capabilities. loxilb supports L7 load balancing through its Ingress implementation, which is enhanced using eBPF `sockmap` helpers. This makes it a strong option for users who need both L4 and L7 load balancing in a unified solution.

### Current Kubernetes Integrations

- [x] service type LoadBalancer   
- [x] kube-proxy replacement with eBPF   
- [x] Ingress Support   
- [x] Kubernetes Gateway API    
- [x] HA capable Egress for Kubernetes    
- [ ] Kubernetes Network Policies (in-progress)    

## Telco-Cloud with loxilb

When deploying telco-cloud environments using cloud-native network functions (CNFs), **loxilb** can serve as a **Service Communication Proxy (SCP)**. SCP is a component defined by [3GPP](https://www.etsi.org/deliver/etsi_ts/129500_129599/129500/16.04.00_60/ts_129500v160400p.pdf) to optimize communication between telco microservices in cloud-native deployments. You can read more about this use case [here](https://dev.to/nikhilmalik/5g-service-communication-proxy-with-loxilb-4242).

![loxilb svc](photos/scp.svg)

Telco-cloud workloads require load balancing and protocol-aware communication across a wide range of interfaces and standards such as **N2**, **N4**, **E2 (ORAN)**, **S6x**, **5GLAN**, **GTP**, and others. Each interface brings its own set of challenges, which loxilb is designed to address. Examples include:

- **N4**: Requires session-aware intelligence at the PFCP layer
- **N2**: Requires NGAP protocol parsing  
  <details>
    <summary>Related Blogs</summary>
    <ul>
      <li><a href="https://www.loxilb.io/post/ngap-load-balancing-with-loxilb">NGAP LB with loxilb</a></li>
      <li><a href="https://futuredon.medium.com/5g-sctp-loadbalancer-using-loxilb-b525198a9103">SCTP LB</a></li>
      <li><a href="https://medium.com/@ben0978327139/5g-sctp-loadbalancer-using-loxilb-applying-on-free5gc-b5c05bb723f0">Free5GC integration</a></li>
    </ul>
  </details>
- **S6x**: Requires Diameter protocol load balancing with SCTP multi-homing  
  <details>
    <summary>Related Blog</summary>
    <ul>
      <li><a href="https://www.loxilb.io/post/k8s-introducing-sctp-multihoming-functionality-with-loxilb">SCTP Multi-Homing Support</a></li>
    </ul>
  </details>
- **MEC (Multi-access Edge Computing)**: May require uplink classifier (UL-CL) support  
  <details>
    <summary>Related Blog</summary>
    <ul>
      <li><a href="https://futuredon.medium.com/5g-uplink-classifier-using-loxilb-7593a4d66f4c">5G UL-CL with loxilb</a></li>
    </ul>
  </details>
- **Mission-critical apps**: May require hitless failover and zero-downtime behavior
- **E2 (ORAN)**: May require SCTP load balancing integrated with OpenVPN
- **SIP (Session Initiation Protocol)**: Needed for enabling cloud-native VoIP
- **N32**: Requires support for SEPP (Security Edge Protection Proxy)
  
loxilb’s deep protocol awareness and ability to operate efficiently in Kubernetes environments make it a strong fit for modern telco-cloud deployments.

## Why choose loxilb?

- **Performs** exceptionally well across different architectures and environments  
  * [Single-Node Performance](https://loxilb-io.github.io/loxilbdocs/perf-single/)  
  * [Multi-Node Performance](https://loxilb-io.github.io/loxilbdocs/perf-multi/)  
  * [Performance on ARM](https://www.loxilb.io/post/running-loxilb-on-aws-graviton2-based-ec2-instance)  
  * [Short Demo on Performance](https://www.youtube.com/watch?v=MJXcM0x6IeQ)
- Utilizes **eBPF**, making it both **flexible** and **customizable**
- Offers advanced **Quality of Service** controls  (per load balancer, per endpoint, or per client)
- Compatible with **any** Kubernetes distribution or CNI  
  (K8s / K3s / K0s / KIND / OpenShift + Calico, Flannel, Cilium, Weave, Multus, etc.)
- Extensive support for **SCTP workloads** (including multi-homing) on Kubernetes
- Built-in dual-stack support with **NAT66** and **NAT64** for Kubernetes environments
- Planned support for **multi-cluster Kubernetes**
- Deployable in **any** environment - public cloud, on-prem, or **standalone**

## Overall features of loxilb
- **L4/NAT Stateful Load Balancer**
  - Supports NAT44, NAT66, NAT64 with One-ARM, FullNAT, DSR, and more
  - Protocol support includes TCP, UDP, SCTP (with multi-homing), QUIC, FTP, TFTP, and others
- **High Availability**
  - Built-in clustering with support for hitless failover, Maglev, and CGNAT-style redundancy
- **Scalable Endpoint Health Probes**
  - Extensive and cloud-native-friendly liveness checks for service endpoints
- **Integrated Security**
  - Stateful firewall capabilities with support for IPsec and WireGuard tunnels
- **Optimized Kernel Features**
  - High-performance implementations for Conntrack, QoS, and related networking functions  
    - Learn more: [Connection Tracking in Linux](https://thermalcircle.de/doku.php?id=blog:linux:connection_tracking_1_modules_and_hooks)
- **IPVS Compatibility**
  - Full support for IPVS-based policies, with automatic policy inheritance
- **Layer 7 Proxy Support**
  - Policy-driven HTTP proxying with planned support for HTTP/1.0, 1.1, and 2.0

## Components of loxilb 
- **Control Plane (Go)**
  - Control plane built in Go for reliability and modularity
- **Data-Path (eBPF)**
  - High-performance, scalable datapath powered by [eBPF](https://ebpf.io/)
- **Routing Stack**
  - Integrated BGP support using a built-in [GoBGP](https://github.com/osrg/gobgp) based implementation
- **Kubernetes Integration**
  - A native Kubernetes agent, [kube-loxilb](https://github.com/loxilb-io/kube-loxilb), written in Go for seamless service discovery and synchronization

## Architectural Considerations   
- [Understanding loxilb modes and deployment in Kubernetes with kube-loxilb](kube-loxilb.md)  
  A deep dive into how loxilb operates in different deployment modes, and how the `kube-loxilb` agent enables seamless integration with Kubernetes.

- [Understanding High-Availability with loxilb](ha-deploy.md)  
  Covers clustering options, failover strategies, and how loxilb ensures resilient service delivery in high-availability environments.

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
- [OAuth2 guide for loxilb API](oauth2.md)
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
- [A Scalable and Fault-Tolerant 5G Core on Kubernetes](https://www.cse.iitb.ac.in/~mythili/research/papers/2025-5gcore-k8s.pdf)   

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

