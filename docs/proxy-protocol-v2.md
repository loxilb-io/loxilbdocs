# Proxy Protocol v2 with LoxiLB

Proxy Protocol v2 plays a critical role in enabling transparency between clients, load balancers, and backend servers. 
Load balancers typically act as intermediaries, and sometimes making it impossible to preserve the source IP address of the original client. 
[Proxy Protocol v2](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) addresses this limitation by forwarding the client’s connection metadata to the backend servers, allowing them to receive and process the original client details.

## How Proxy protocol works with LoxiLB

When a client establishes a TCP connection with the backend server after the completion of a 3-way handshake, LoxiLB appends a Proxy Protocol v2 header containing the client’s metadata, and forwards the request to the backend servers. The Proxy Protocol v2 header, containing the client's connection details, is added to the beginning of the data stream.

![image](https://github.com/user-attachments/assets/5b0f4d9d-a2d2-4a7a-b753-e5fb6a2472b6)

The Client's metadata may include original L3/L4 information, for example:
- Client's IP and Port(Source/Destination)
- Protocol(TCP/UDP)
- Address Family(IPv4/IPv6)
- Checksum
- Length
- Custom Data

By encoding essential Client's connection metadata, it preserves transparency, extends visibility and facilitates accurate logging and troubleshooting in complex environments. Furthermore, loxilb uses eBPF to implement proxy protocol headers on-the-fly. Hence there is hardly any performance impact on enabling proxy protocol. However one should confirm if the server-side applications deployed are compatible with proxy protocol v2.

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

Currently, loxilb supports proxy protocol v2 with only Ipv4 and TCP payloads. Support for more combinations of protocols is in progress.



