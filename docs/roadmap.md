# Release Notes  (For major release milestones)   

## 0.7.0 beta (Aug, 2022)

Initial release of loxilb     

- **Functional Features**:    
    - Two-Arm Load-Balancer (NAT+Routed mode)     
        - Upto 16 end-points support    
    - Load-balancer selection policy    
        -  Round-robin, traffic-hash (fallback to RR if hash fails)    
    - Conntrack support in eBPF - TCP/UDP/ICMP/SCTP profiles    
    - GTP with QFI extension support    
        - ULCL classifier support    
    - Native QoS-Policer support (SRTCM/TRTCM)    
    - GoBGP Integration          
    - Extended visibility and statistics     

- **LB Spec Support**:     
    - IP allocation policy    
    - Kubernetes 1.20 base support
    - Support for Calico Networking    
 
- **Utilities**:     
    - loxicmd support : Configuration utlity with the look and feel of kubectl    

## 0.8.0 (Dec, 2022)      

- **Functional Features**:    
    - Enhanced load-balancer support including SCTP statefulness, WRR distribution    
    - Integrated Firewall support    
    - Integrated end-point health-checks    
    - One-ARM, FullNAT, DSR LB mode support    
    - NAT66/NAT64 support    
    - Clustering support      
    - Integration with Linux egress TC hooks    
  
- **LB Spec**:    
    - Stand-alone mode to support LB Spec [kube-loxilb](https://github.com/loxilb-io/kube-loxilb)    
    - Load-balancer class support    
    - Advanced IPAM for ipv4/ipv6 with shared/exclusive mode    
    - Kubernetes 1.25 Integration     

- **Utilities**:  
    - loxicmd support : Data-Store support, more commands

## 0.9.0 (Nov, 2023)     

- **Functional Features**:  
    - Hardened NAT Support - CGNAT'ish   
    - L3 DSR mode Support   
    - Https end-point liveness probes   
    - Maglev clustering   
    - SCTP multihoming support   
    - Integration with Linux native QoS   
    - Support for Cilium, Weave Networking   
    - Grafana based dashboard   
    - IPSEC Support (with VTI)
    - Initial support for in-cluster mode    

- **kube-loxilb/LB Spec Support**: 
    - OpenShift Integration    
    - Support for per-service liveness-checks, IPAM type, multi-homing annotations   
    - Kubernetes 1.26 (k0s, k3s, k8s )   
    - Operator support   
    - AWS EKS support
 
## 0.9.3 (May, 2024)   

- **Functional Features**:
    - Kube-proxy replacement support    
    - IPVS compatibility mode    
    - Master-plane HA support   
    - BFD and GARP support for Hitless HA    
    - Enhancements for Multus support    
    - SCTP multi-homing end-to-end support    
    - Cloud Availability zone(s) support    
    - Redhat9 and Ubuntu24 support   
    - Support for upto Linux Kernel 6.8    
    - Full Support for Oracle OCI      
    - SockAddr eBPF for LocalVIP access   
    - Container size enhancements   
    - HA enhancements for multiple cloud-providers and various scenarios (active-active, active-standby, clustered etc)     
    - CICD infra enhancements   
    - Robust secret management for HTTPS apis   
    - Performance enhancements with CT scaling   
    - Enhanced exception handling
    - GoLang Profiling Support
    - Full support for in-cluster mode
    - Better support for virtio environments    
    - Enhanced RSS distribution mode via XDP (especially for SCTP workloads)   
    - Loadbalancer algorithms - LeastConnections and SessionAffinity added    

- **kube-loxilb Support**: 
    - Kubernetes 1.29   
    - BGP (auto) Mesh support    
    - CRD for BGP peers    
    - Kubernetes GWAPI support   
 
- **Utilities**:  
    - N4 pfcp test-tool added   
    - Seagull test tool integrated   
    - Massive updates to documentation    
      
## 0.9.5 (Jul, 2024)   

- **Functional Features**:
    - L7 (Transparent) proxy
    - HTTPS termination  
    - [Native eBPF implementation for Policy based IP Masquerade/SNAT](https://github.com/loxilb-io/loxilb/issues/718)      
    - Kubernetes vCluster support   
    - E2E SCTP multi-homing support with Multus   
    - [Multi-AZ/Region hitless HA support for AWS/EKS](https://github.com/loxilb-io/loxilb/issues/681)   
    - [Service communication proxy support for Telco deployments](https://dev.to/nikhilmalik/5g-service-communication-proxy-with-loxilb-4242)      

- **Kubernetes Support**: 
    - Kubernetes 1.30   
    - [CRD for BGP policies](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/k8s_bgp_policy_crd.md)        
 
## 0.9.6 (Aug, 2024)   

- **Functional Features**:
    - [Support for any host onearm LB rule](https://github.com/loxilb-io/loxilb/pull/694)        
    - [HTTP 2.0 parser](https://github.com/loxilb-io/loxilb/issues/48)     
    - [NGAP protocol parser](https://www.loxilb.io/post/ngap-load-balancing-with-loxilb)       
    - [ECMP Load-balancing support](https://github.com/loxilb-io/loxilb/issues/730)       
    - [Non-privileged Container support](https://github.com/loxilb-io/loxilb/discussions/654)
    - [AWS Local-Zone support](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/eks-external.md#restricting-loxilb-service-for-a-local-zone-node-group)    
    - [Multi-Cloud HA support (AWS+GCP)](https://github.com/loxilb-io/loxilb/issues/697)   
    - Updated CICD workflows     
 
- **Kubernetes Support**: 
    - [Ingress Manager support](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/loxilb-ingress.md)       
    - Enhanced GW API support   

## 0.9.7 (Oct, 2024)   Planned
- **Functional Features**:
    - [SRv6 implementation](https://github.com/loxilb-io/loxilb/issues/14)         
    - [Rolling upgrades](https://github.com/loxilb-io/loxilb/issues/312)        
    - [URL Filtering](https://github.com/loxilb-io/loxilb/issues/133)            
    - [Wireguard support (ingress + egress)](https://github.com/loxilb-io/loxilb/issues/67)        
    - [SIP protocol support](https://github.com/loxilb-io/loxilb-ebpf/issues/45)       
    - [Sockmap support for SCTP](https://github.com/loxilb-io/loxilb/issues/750)     
    - [Support for proxy protocol v2](https://github.com/loxilb-io/loxilb/issues/675)    
    - [SYNProxy support](https://github.com/loxilb-io/loxilb/issues/670)    
    - IPSec service mesh for Telco workloads  (ingress + egress)     

- **Kubernetes Support**: 
    - Kubernetes 1.31   
    - Multi-cluster support
    - Support for Cilium and LoxiLB in-cluster [support](https://github.com/loxilb-io/kube-loxilb/issues/158)    
    - Kubernetes network policy support    
