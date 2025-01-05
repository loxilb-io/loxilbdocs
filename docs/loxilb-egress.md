### What is Kubernetes egress

In Kubernetes, egress refers to outbound network traffic originating from pods within a cluster. This traffic is sent to external destinations, such as other services, APIs, databases, or the internet.

### Why is Kubernetes Egress Important?

Kubernetes egress is needed to ensure the reliability, scalability, and consistency of outbound traffic from a Kubernetes cluster. Here are the key reasons why it is essential:

* External Connectivity: Pods often need to interact with resources outside the cluster (e.g., APIs, external databases, or other cloud services).
* Stable IP Representation: HA egress allows the use of static egress IPs for consistent communication with external systems, making external firewalls, whitelists, or identity-based access control more reliable.
* Regulated Outbound Traffic: HA egress solutions can enforce policies and logging for egress traffic, ensuring compliance with regulatory requirements (e.g., GDPR, PCI-DSS).
* Controlled IP Ranges: By centralizing and managing egress IPs, HA ensures that only specific, authorized IPs are exposed externally, reducing the attack surface.
* Traffic Insights: HA egress solutions often provide monitoring tools that track traffic patterns, allowing for proactive scaling or optimization.

Kubernetes egress needs to have High Availability (HA) to ensure reliability, scalability, and control for outbound traffic.

![image](https://github.com/user-attachments/assets/ce4144d9-36d3-4022-a1b9-c27ecb312c14)

### Kubernetes Egress with LoxiLB

LoxiLB already  provides support for Kubernetes ServiceType: LoadBalancer, enabling direct integration with Kubernetes to manage external traffic for services efficiently. But, ServiceType: LoadBalancer is usually used for traffic coming into the Kubernetes cluster (reverse proxy). With Kubernetes egress support, LoxiLB provides a HA enabled solution to manage outgoing traffic from Kubernetes pods( Forward proxy).

![image](https://github.com/user-attachments/assets/654e9409-2ff3-456d-8af2-8c26ea4561d3)

### Deploy HA egress with LoxiLB for secondary services

LoxiLB supports high-availability (HA) egress for a wide range of Kubernetes scenarios, whether deployed in in-cluster or external mode. This guide elaborates on how to configure LoxiLB as an HA egress solution for pods using secondary interfaces. These deployments are particularly popular for multi-homed pods or applications requiring high performance.

This guide builds upon the [feature](https://docs.loxilb.io/latest/loxilb-incluster-multus/) introduced by LoxiLB to support in-cluster load balancer services for secondary interfaces. A similar deployment setup can be followed when implementing the steps in this guide. The following shows a sample deployment which uses a egress VIP for secondary services : 

![image](https://github.com/user-attachments/assets/da4d499d-aa58-4a24-b58e-485744f9a6a8)


If a loxilb pod or node hosting loxilb fails, the VIP is reassociated with an active node without any disruption of egress traffic :

![image](https://github.com/user-attachments/assets/c7df6fe2-ded4-40fb-96d3-c9ac2f5d8dec)

As  mentioned, the basic deployment steps can be found in our existing [guide]([feature]((https://docs.loxilb.io/latest/loxilb-incluster-multus/))).  Here we will touch upon the necessary changes and additional steps to enable egress feature. 

##### Enable loxilb specific egress CRD

```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/crds/egress-crd.yaml
```

##### Use kube-loxilb with egress CRD permissions  

```
kubectl apply -f https://github.com/loxilb-io/kube-loxilb/blob/main/manifest/in-cluster/kube-loxilb-nobgp.yaml
```

Please note that there are new ClusterRoles introduced in the above yaml for egress CRD support :

```
- apiGroups:
      - egress.loxilb.io
    resources:
      - egresses
    verbs: ["get", "watch", "list", "patch", "update"]
```
##### Create egress CRD rules

One can use the following yaml to create an egress object for the above scenario :

```
apiVersion: "egress.loxilb.io/v1"
kind: Egress
metadata:
  name: loxilb-egress-svc2
spec:
  #address: IP list of the Pod on which you want the egress rule applied
  addresses:
  - 123.123.123.17
  - 123.123.123.18
  #vip: Corresponding VIP for forward-proxy.
  vip: 123.123.123.200
```

**Note>** :  When using Multus-based secondary interfaces, you may need to configure the default route to utilize the secondary services. To achieve this, include the ```default-route``` annotation in the Pod YAML as shown below:

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  labels:
    app: pod-01
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{
      "name": "vlan5",
      "default-route": ["123.123.123.200"]
    }]'
spec:
  containers:
    - name: testpod
      image: ghcr.io/loxilb-io/nettest:latest
      command: ["sleep"]
      args: [ "infinity" ]
      ports:
        - containerPort: 8181
```
Routes can also be selectively configured to utilize this egress as needed. With the above setup, pods can take advantage of LoxiLB's high-performance HA egress, ensuring uninterrupted service and minimal downtime.
