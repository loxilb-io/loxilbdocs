# LoxiLB - Mastering L4/L7 Load Balancing with Kubernetes Gateway API

The Kubernetes Gateway API is a significant advancement in managing traffic load balancing at both L4 and L7. It provides a flexible and extensible framework that allows developers to define how external traffic should be directed to their services, enabling fine-grained control over routing decisions and security policies. This comprehensive guide aims to demystify L4 and L7 load balancing within the context of the Kubernetes Gateway API.

In this guide, we will explore the fundamental concepts and provide some examples to help you leverage this powerful toolset.

![image](https://github.com/user-attachments/assets/62b24c2a-19d5-4d92-a15c-0b1cfbba74b7)

### Getting Started
This guide assumes that users have a Kubernetes cluster pre-installed. If not, they can follow various [resources](https://kubernetes.io/docs/setup/) available in the web.

### Install loxilb as a L4 service LB
You may follow any of the [loxilb getting started guides](https://github.com/loxilb-io/loxilb?tab=readme-ov-file#getting-started) as per requirement. In this example, we will run loxilb-lb in external mode. 

## Getting started with Gateway API

Before getting into creating gateway resource, let's delve into some of them.

### GatewayClass
The actual implementation of the Gateway API varies depending on which controller is used. Users can use a variety of controllers, and when creating a Gateway API, they must specify the controller.

[*GatewayClass*](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/?h=gatewayclass) is a resource that specifies the name of the controller that provides the Gateway API implementation. 

This is similar to the *spec.LoadBalancerClass* of the *LoadBalancer* service. Just as users can use LoadBalancerClass to set a specific provider to control the LoadBalancer function, they can create a GatewayClass and then set other Gateway API Resources to be controlled by a specific provider.

Users must create at least one *GatewayClass* to use the Gateway API.

### Gateway
[*Gateway*](https://gateway-api.sigs.k8s.io/api-types/gateway/) is a resource that defines external IP, port, and protocol that can be accessed from the outside.

When creating a gateway, kube-loxilb is assigned an external IP from the IP Pool and provides it to the gateway. (You can also specify the IP statically when creating a gateway.)

Just creating a gateway does not connect the path from the outside to the Pod. You must create TCPRoute, UDPRoute, etc. connected to the gateway.

### TCPRoute/UDPRoute/HTTPRoute
TCPRoute, and UDPRoute define routing for TCP and UDP traffic, respectively. HTTPRoute resource can be used to define routing for HTTP and HTTPs traffic as well.

It is linked to the listener defined in the gateway (ports, protocols, etc).

When creating a Route resource, a new LoadBalancer Type service is created using listener information and Route resource information.

### First, create the CRDs:

For using the Gateway API, you must first install the Gateway API CRD on K8s. 

Installation is divided into **Standard Channel** and **Experimental Channel**. The Standard side is the official release, and the Experimental side includes Standard + experimental CRDs. Since TCPRoute and UDPRoute CRDs corresponding to Gateway API L4 functions are provided in Experimental Channel, we must install Experimental Channel.

For the relationship between each channel, please refer to the [this page](https://gateway-api.sigs.k8s.io/concepts/versioning/).

For an explanation of CRD version management, see [CRD Management Page](https://gateway-api.sigs.k8s.io/guides/crd-management/).

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
```

### kube-loxilb(loxilb's operator for Kubernetes)
In order for kube-loxilb to use the Gateway API, ClusterRole must have permission to access the gateway API resource *(apiGroup: [”gateway.networking.k8s.io”])* as follows:
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-loxilb
rules:
  - apiGroups: [""]
    resources: ["nodes", ["pods", "endpoints"]]
    verbs: ["get", "watch", "list", "patch"]
  - apiGroups: [""]
    resources: ["services", "services/status"]
    verbs: ["*"]
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["gatewayclasses", "gatewayclasses/status", "gateways", "tcproutes", "udproutes"]
    verbs: ["get", "watch", "list", "patch", "update"]
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["authentication.k8s.io"]
    resources: ["tokenreviews"]
    verbs: ["create"]
  - apiGroups: ["authorization.k8s.io"]
    resources: ["subjectaccessreviews"]
    verbs: ["create"]
```
 
#### Install kube-loxilb
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/kube-loxilb.yaml
```

Since, Gateway API's HTTPRoute for https will be handled via loxilb-ingress module, we must prepare SSL certificates.

### Prepare TLS/SSL certificates for Ingress
Self-signed TLS/SSL certificates and private keys can be built using various tools like OpenSSL or Minica. Basically, one will need to have two files - server.crt and server.key for loxilb-ingress usage. Once these files are in place, a Kubernetes secret can be created using the following yaml:
```
apiVersion: v1
data:
  server.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUI3RENDQVhPZ0F3SUJBZ0lJU.....
  server.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JRzJBZ0VBTUJBR0J5cUdTTTQ5Q.....
kind: Secret
metadata:
  creationTimestamp: null
  name: loxilb-ssl
  namespace: kube-system
type: Opaque
```
The above values are just dummy values but it is important to note that they need to be in base64 format not in pem format. How do we get the base64 values from server.crt and server.key files ?
```
$ base64 server.crt
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUI3RENDQVhPZ0F3SUJBZ0lJU.....
$ base64 server.key
LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JRzJBZ0VBTUJBR0J5cUdTTTQ5Q.....
```

Now, after applying the yaml, we can check the created secret :
```
$ kubectl get secret -n kube-system loxilb-ssl
NAME         TYPE     DATA   AGE
loxilb-ssl   Opaque   2      24s
```
In the subsequent steps, this secret loxilb-ssl will be used throughout.

### Install loxilb-ingress

```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/loxilb-ingress-deploy.yml
```

Check the status of running pods:
```
$ kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-7b98449c4-85qsc                   1/1     Running   0          99m
kube-system   kube-loxilb-575cd8dc7f-xlzw2              1/1     Running   0          3m4s
kube-system   local-path-provisioner-595dcfc56f-b6gt8   1/1     Running   0          99m
kube-system   loxilb-ingress-g7nf5                      1/1     Running   0          36s
kube-system   metrics-server-cdcc87586-pvws9            1/1     Running   0          99m
```

### Create Gateway API resources

#### GatewayClass
The following is the example of GatewayClass with controller name:
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: test-gc
  namespace: kube-system
spec:
  controllerName: "loxilb.io/loxilb"
```

Here, *loxilb.io/loxilb* is the name of the gateway API controller name. Now, kube-loxilb will be able to handle all Gateway APIs that use test-gc as the gatewayClass.

Register the Gateway API controller name:
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/gatewayclass.yaml
```

After creating it, you can check the results using the following command.
```
$ kubectl get gatewayclass
NAME      CONTROLLER         ACCEPTED   AGE
test-gc   loxilb.io/loxilb   True       16s
```
This displays gatewayClass controller and the status. ACCEPTED as True means that kube-loxilb detected the creation of gatewayClass and successfully updated the status. If the provider specified in controllerName does not exist in K8s, ACCEPTED is displayed as Unknown.

#### Gateway
The following is the example of gateway:
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: test-gateway
  namespace: kube-system
spec:
  gatewayClassName: test-gc
  listeners:
  - name: test-listener
    protocol: TCP
    port: 21818
    allowedRoutes:
      kinds:
      - kind: TCPRoute
```
The description of the spec is as follows:

|Name|Description|
|----|----|
|gatewayClassName|The name of the GatewayClass. Specifies which GatewayClass the Gateway will use.|
|listeners| You can specify ports and protocols accessible from the outside. When creating a Gateway, you must have at least one [listener](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2Fv1.Listener)|
|    - name| listener's name|
|    - protocol| Protocol to open via listener. You can use “HTTP”, “HTTPS”, “TCP”, “TLS”, “UDP”, and currently kube-loxilb only supports “TCP”, “UDP”, HTTP, HTTPS.|
|    - port| Port number to open via listener|
|    - allowedRoutes|You can specify Routes resources (TCPRoute, UDPRoute etc) that can be connected to the listener.Multiple Routes resources can be specified, but only one is connected.|

Create the gateway resource:
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/gateway.yaml
```

In the example, *test-gc* was specified as gatewayClassName. TCP port 21818 was set to open through the listener, and the listener was set to connect only to the TCPRoute resource.

Check the created gateway:
```
$ sudo kubectl get gateway -A
NAMESPACE     NAME           CLASS     ADDRESS         PROGRAMMED   AGE
kube-system   test-gateway   test-gc   192.168.80.90   True         59m
```

We can see PROGRAMMED is displayed as True, it means that kube-loxilb was able to detect the creation of the corresponding gateway. If detection fails or an error occurs, PROGRAMMED is displayed as Unknown.

And, 192.168.80.90 IP was assigned to ADDRESS. This is the IP assigned from kube-loxilb's IPAM. If you want to assign a static IP to the Gateway, you can specify it in spec.addresses.

Verify the Gateway Ingress service
```
$ sudo kubectl get svc -A -o wide
NAMESPACE     NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)                      AGE     SELECTOR
default       kubernetes                     ClusterIP      10.43.0.1       <none>              443/TCP                      4h22m   <none>
kube-system   kube-dns                       ClusterIP      10.43.0.10      <none>              53/UDP,53/TCP,9153/TCP       4h22m   k8s-app=kube-dns
kube-system   metrics-server                 ClusterIP      10.43.200.202   <none>              443/TCP                      4h22m   k8s-app=metrics-server
kube-system   test-gateway-ingress-service   LoadBalancer   10.43.210.140   llb-192.168.80.90   80:32413/TCP,443:31345/TCP   64m     app=loxilb-ingress-app
```

#### TCPRoute
The following is the example of TCPRoute resource:
```
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: test-tcproute
  namespace: kube-system
  labels:
    selectorkey: run
    selectorvalue: my-nginx
  annotations:
    ### https://loxilb-io.github.io/loxilbdocs/kube-loxilb/
    #loxilb.io/liveness: "yes"
    #loxilb.io/lbmode: "fullnat"
spec:
  # find gateway and gateway's listener
  parentRefs:
  - name: test-gateway         # name of gateway
    sectionName: test-listener # name of listener
  rules:
  - backendRefs:
    - name: tcproute-lb-service
      port: 80
```
Kindly note that the apiVersion is v1alpha2, different from GatewayClass or Gateway.

The details description of the spec arguments is [here](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1alpha2.TCPRoute).

* Create the TCPRoute rule:
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/tcpRoute.yaml
```

* Check the result:
```
$ kubectl get tcproute -A
NAMESPACE     NAME            AGE
kube-system   test-tcproute   19m
```

All the information cannot be confirmed using TCPRoute alone. However, if you check the service using the following command, you can confirm that the service was created with the name specified in rules.backendRefs.
```
kubectl get svc -A -o wide
NAMESPACE     NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)                      AGE     SELECTOR
default       kubernetes                     ClusterIP      10.43.0.1       <none>              443/TCP                      4h25m   <none>
kube-system   kube-dns                       ClusterIP      10.43.0.10      <none>              53/UDP,53/TCP,9153/TCP       4h25m   k8s-app=kube-dns
kube-system   metrics-server                 ClusterIP      10.43.200.202   <none>              443/TCP                      4h25m   k8s-app=metrics-server
kube-system   tcproute-lb-service            LoadBalancer   10.43.157.245   llb-192.168.80.90   21818:30388/TCP              20m     app=tcproute-pod
kube-system   test-gateway-ingress-service   LoadBalancer   10.43.210.140   llb-192.168.80.90   80:32413/TCP,443:31345/TCP   67m     app=loxilb-ingress-app
```

We can see that tcproute-lb-service was created as a LoadBalancer service, and the address assigned to the gateway is used as the external IP. The port also uses 21818 as specified in the gateway's listener.

* Test the service
```
$ curl http://192.168.80.90:21818
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
#### UDPRoute

Similarily, we can create UDPRoute rule as well.

* Create the UDPRoute rule:
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/udpRoute.yaml
```

* Check the result:
```
$ kubectl get udproute -A
NAMESPACE     NAME            AGE
kube-system   test-udproute   24m
```

* Check the service created:
```
$ kubectl get svc -A -o wide
NAMESPACE     NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)                      AGE     SELECTOR
default       kubernetes                     ClusterIP      10.43.0.1       <none>              443/TCP                      4h26m   <none>
kube-system   kube-dns                       ClusterIP      10.43.0.10      <none>              53/UDP,53/TCP,9153/TCP       4h26m   k8s-app=kube-dns
kube-system   metrics-server                 ClusterIP      10.43.200.202   <none>              443/TCP                      4h26m   k8s-app=metrics-server
kube-system   tcproute-lb-service            LoadBalancer   10.43.157.245   llb-192.168.80.90   21818:30388/TCP              21m     app=tcproute-pod
kube-system   test-gateway-ingress-service   LoadBalancer   10.43.210.140   llb-192.168.80.90   80:32413/TCP,443:31345/TCP   68m     app=loxilb-ingress-app
kube-system   udproute-lb-service            LoadBalancer   10.43.254.49    llb-192.168.80.90   21819:32369/UDP              24m     app=udproute-pod
```

* Test the service
```
$ ./udp_client 192.168.80.90 21819
Client address: 10.42.0.1:11607
Data sent by client:
Hello

```
#### HTTPRoute
Now, we will see how we can use HTTPRoute to create HTTP gateway api rules.

* Create the HTTPRoute
```
$ kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/httpRoute.yaml
```

* Check the result
```
$ kubectl get httproute -A
NAMESPACE     NAME               HOSTNAMES                       AGE
kube-system   test-http-route    ["test.loxilb.gateway.http"]    29m

$ kubectl get ingress -A
NAMESPACE     NAME               CLASS    HOSTS                       ADDRESS             PORTS     AGE
kube-system   test-http-route    loxilb   test.loxilb.gateway.http    llb-192.168.80.90   80        30m
```

* Test the service
```
curl -s --connect-timeout 30 -H "Application/json" -H "Content-type: application/json" -H "HOST: test.loxilb.gateway.http" http://192.168.80.90:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
#### HTTPRoute for HTTPs
Lastly, we will see how we can use HTTPRoute to create HTTPs gateway api rules.

* Create the HTTPRoute for HTTPs
```
$ kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/gateway-api/httpsRoute.yaml
```

* Check the result
```
$ kubectl get httproute -A
NAMESPACE     NAME               HOSTNAMES                       AGE
kube-system   test-http-route    ["test.loxilb.gateway.http"]    33m
kube-system   test-https-route   ["test.loxilb.gateway.https"]   48m

$ kubectl get ingress -A
NAMESPACE     NAME               CLASS    HOSTS                       ADDRESS             PORTS     AGE
kube-system   test-http-route    loxilb   test.loxilb.gateway.http    llb-192.168.80.90   80        34m
kube-system   test-https-route   loxilb   test.loxilb.gateway.https   llb-192.168.80.90   80, 443   49m
```

* Test the service
```
curl -s --connect-timeout 30 -H "Application/json" -H "Content-type: application/json" -H "HOST: test.loxilb.gateway.https" --insecure https://192.168.80.90:443
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

