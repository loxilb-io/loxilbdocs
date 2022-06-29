# loxilb eBPF implementation details

In this section, we will look into details of loxilb ebpf implementation in little details and try to check what goes on under the hood. When loxilb is build, it builds two object files as follows :

```
llb@nd2:~/loxilb$ ls -l /opt/loxilb/
total 396
drwxrwxrwt 3 root root    0  6?? 20 11:17 dp
-rw-rw-r-- 1 llb llb 305536  6?? 29 09:39 llb_ebpf_main.o
-rw-rw-r-- 1 llb llb  95192  6?? 29 09:39 llb_xdp_main.o
```

As the name suggests and based on hook point, *xdp* version does XDP packet processing while *ebpf* version is used at TC layer for TC eBPF processing. Interesting enough, the packet forwarding code is largely agnostic of its final hook point due to usage of a light abstraction layer to hide differences between eBPF and XDP layer.

Now this beckons the question why separate hook points and how does it all work together ? loxilb does bulk of its processing at TC eBPF layer as this layer is most optimized for doing L4+ processing needed for loxilb operation. XDP's frame format is different than what is used by skb (linux kernel's generic socket buffer). This makes in very difficult (if not impossible) to do tcp checksum offload and other such features used by linux networking stack for ages. In short if we need to do such operations XDP performance will be inherently slow. XDP as such is perfect for quick operations at l2 layer. loxilb uses XDP to do certain operations like mirroring. Due to how TC eBPF works, it is difficult to work with multiple packet copies and loxilb's TC eBPF offloads some functinality to XDP layer in such special cases.

## Loading of loxilb eBPF program

loxilb's go agent by default loads the loxilb ebpf programs to all the interfaces(only physical/real/bond/wireguard) available  in the system. As loxilb is designed to run in its own container, this is convienient for users who dont want to have to manually load/unload eBPF programs. However, it is still possible to do so manually :

To load :
```
ntc filter add dev eth1 ingress bpf da obj /opt/loxilb/llb_ebpf_main.o sec tc_packet_parser
```

To unload:
```
ntc filter del dev eth1 ingress
```
-- Please not that ntc is the customized tc tool from iproute2 package which can be found in loxilb's repository

## Entry points of loxilb eBPF

loxilb's eBPF code is usually divided into two program sections with the following entry functions :

- tc_packet_func\
  This alongwith the consequent code does majority of the packet processing. If conntrack entries are in established state, this is also responsible for packet tx. However if conntrack entry for a particular packet flow is not established, it makes a bpf tail call to the *tc_packet_func_slow*
  
- tc_packet_func_slow\
  This is responsible mainly for doing NAT lookup and stateful conntrack implementation. Once conntrack entry transitions to established state, the forwarding then can happen directly from tc_packet_fn
  
- xdp_packet_func\
  This is the entry point for packet processing when hook point is XDP instead of TC eBPF
  
  ## Pinned Maps of loxilb eBPF
  
  All maps used by loxilb eBPF are mounted as below :
  





