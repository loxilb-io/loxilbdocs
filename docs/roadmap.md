# Release Notes

## 0.7.0 beta (Aug, 2022)

Initial release of loxilb 

- **Major functions**:  
    - Two-Arm Load-Balancer (NAT+Routed mode)
        - Upto 16 end-points support
    - Load-balancer selection policy
        -  Round-robin, traffic-hash (fallback to RR if hash fails)
    - Conntrack support in eBPF - TCP/UDP/ICMP profiles
    - GTP with QFI extension support
        - UL/CL classifier support for MEC
    - Extended QoS support (SRTCM/TRTCM)
    - Support for GoBGP
    - Support for Calico CNI
    - Extended visibility and statistics 

- **CCM Support**: 
    - IP allocation policy
    - Kubernetes 1.20 base support
 
- **Utilities**:  
    - loxicmd support : Configuration utlity with the look and feel of kubectl

## 0.8.0 (Nov, 2022) - Planned  

- **Major functions**: 
    - Enhanced load-balancer support upto 32 end-points
    - Integrated Firewall support
    - Extended conntrack - SCTP support
    - One-ARM LB mode support
    - SRv6 support
    - HA support
    - Grafana based dashboard
  
- **CCM Support**: 
    - OpenShift Integration
  
- **DPU Support**:  
    - Nvidia BF2 Support (Depends on community/public demand)

