In Kubernetes, there are two key concepts - <b><i>Service and Endpoint</b></i>.

## What is Service?
A <b><i>"Service"</i></b> is a method that exposes an application running in one or more pods.

## What is an Endpoint?
An <b><i>"Endpoint"</b></i> defines a list of network endpoints(IP address and port), typically referenced by a Service to define which Pods the traffic can be sent to.

When we create a service in Kubernetes, usually we do not have to worry about the Endpoints' management as it is taken care by Kubernetes itself. But, sometimes your service is hosted outside your cluster, may be in other cluster(s).
In that case, your cloud-native apps needs to connect to the external services with external endpoints.

![External Endpoint](photos/ext-ep.png)

For this, You can simply create an Endpoint Object:

```
apiVersion: v1
kind: Endpoints
metadata:
  name: ext-tcp-lb
subsets:
  - addresses:
    - ip: 192.168.82.2
    ports:
    - port: 80
```

And, create a service using that endpoint:
```
apiVersion: v1
kind: Service
metadata:
  name: ext-tcp-lb
spec:
 loadBalancerClass: loxilb.io/loxilb
 type: LoadBalancer 
 ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
```
