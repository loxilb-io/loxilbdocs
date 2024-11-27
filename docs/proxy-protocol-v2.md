# Proxy Protocol v2 with LoxiLB

In network architectures involving load balancers, client connections are routed through these intermediaries before reaching backend servers. While load balancers effectively distribute traffic, they often hide the original client's IP address and connection details, as backend servers see the load balancer as the source.

This lack of transparency creates challenges, such as:

* <b>Accurate Logging</b>: Backend servers cannot log the original client information, leading to incomplete or misleading records.   
* <b>Troubleshooting</b>: Identifying and resolving client-specific issues becomes difficult without visibility into connection metadata.     
* <b>Access Control</b>: Enforcing IP-based rules or geolocation-based policies becomes impossible without knowing the original client’s IP.   

[Proxy Protocol v2](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) addresses these limitations by embedding client connection metadata into the communication stream. This enables backend servers to process requests while retaining visibility into the original client details, enhancing transparency, accuracy, and control in modern network environments.   

## How Proxy protocol works with LoxiLB

When a client establishes a TCP connection with a backend server (after completing the 3-way handshake), LoxiLB appends a Proxy Protocol v2 header containing the client’s metadata to the beginning of the data stream. The request is then forwarded to the backend servers.

![image](https://github.com/user-attachments/assets/5b0f4d9d-a2d2-4a7a-b753-e5fb6a2472b6)

The metadata in the Proxy Protocol v2 header can include:

Client Information: Original IP and port (source/destination).
Protocol Details: TCP/UDP.
Address Family: IPv4/IPv6.
Additional Fields: Checksum, length, or custom data.
By encoding this metadata, LoxiLB ensures transparency, enhances visibility, and enables accurate logging and troubleshooting in complex environments.

LoxiLB leverages [eBPF](ebpf.io) technology to generate Proxy Protocol headers dynamically with minimal performance overhead. However, before enabling Proxy Protocol v2, ensure that server-side applications support this feature to avoid compatibility issues.

## How to create a Service enabling Proxy Protocol v2

One can follows various [guides](https://docs.loxilb.io/latest/kube-loxilb/) about general deployment of loxilb. Below is an example manifest for creating a load-balancer service with Proxy protocol v2 with loxilb :
```
apiVersion: v1
kind: Service
metadata:
  name: tcp-lb-fullnat
  annotations:
    loxilb.io/liveness: "yes"
    loxilb.io/lbmode: "fullnat"
    loxilb.io/useproxyprotov2 : "yes"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: tcp-fullnat-test
  ports:
    - port: 57002
      targetPort: 80
  type: LoadBalancer
```

## How to verify Proxy Protocol v2

Proxy protocol v2 can be checked using tools like [tshark](https://www.wireshark.org/docs/man-pages/tshark.html). For example :

```
tshark --disable-protocol http -VY proxy.v2.protocol==0x01
```
![Screenshot 2024-11-27 at 3 04 11 PM](https://github.com/user-attachments/assets/d58aa3f5-39dc-48aa-992b-36b1a3edf90a)

### Limitations

LoxiLB currently supports Proxy Protocol v2 with IPv4 and TCP payloads. Support for additional protocol combinations is under development.
