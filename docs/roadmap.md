# Release Notes

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
    - Support for Calico CNI  
 
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
    - Support for Cilium, Weave CNI   
    - Grafana based dashboard   
    - IPSEC Support (with VTI)   

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
    - Performance enhancements   
    - Enhanced RSS distribution mode via XDP (especially for SCTP workloads)   
    - Loadbalancer algorithms - LeastConnections and SessionAffinity added    

- **kube-loxilb Support**: 
    - Kubernetes 1.29   
    - BGP (auto) Mesh support    
    - CRD for BGP peers    
    - Kubernetes GW-API support   
 
- **Utilities**:  
    - N4 pfcp test-tool added   
    - Seagull test tool integrated   
    - Massive updates to documentation    
      
## 0.9.5 (Jul, 2024) - Planned   

- **Functional Features**:
    - SRv6 support  
    - Rolling upgrades   
    - L7 (Transparent) proxy   
    - HTTPS termination  
    - URL Filtering       
    - DNS caching   
    - Wireguard support  
    - SIP protocol support   
    - Policy based SNAT    
    - Kubernetes vCluster Support    

- **kube-loxilb Support**: 
    - Kubernetes 1.30        
    - Multi-cluster support   
    - CRD for BGP policies   
    - Kubernetes network policy support    
