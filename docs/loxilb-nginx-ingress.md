## How to run loxilb with ingress-nginx

In Kubernetes, there is usually a lot of overlap between network load-balancer and an Ingress functionality. This creates a lot of confusion. Overall, the differences between an Ingress and a load-balancer service can be categorized as follows:

| Feature       | Ingress       | Load-balancer |
| ------------- | ------------- |-------------- |
| Protocol      | HTTP(s) level - Layer7  | Network Layer4   |
| Additional Features | Ingress Rules, Resource-Backends | Based on L4 Session Params |
| Yaml Manifest| apiVersion: networking.k8s.io/v2 | type: LoadBalancer|

[image]: (https://github.com/loxilb-io/loxilbdocs/assets/111065900/ed3fcbcb-2e8b-40d6-90dc-6b5f3539e963)

With Kubernetes ingress, we can expose multiple paths with the same service IP. This might be helpful if one is using public cloud, where one has to pay for managed LB services. Hence, creating a single service and exposing mulitple URL paths might be optimal in such use-cases.

For this example, we will use ingress-nginx which is a kubernetes community driven ingress. loxilb has its own [ingress implementation](docs/loxilb-ingress.md), which is optimized (with eBPF helpers) for cases which require long-lived connections, https termination etc. However, if someone needs to use it any other ingress implementation, they can follow this guide. This guide uses ingress-nginx as the ingress implementation.

### Considerations

This example is not specific to any particular managed kubernetes implementation like EKS, GKE etc but should work well with any. We will simply use K3s as a based kubernetes platform.

### Install K3s
```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik,servicelb" K3S_KUBECONFIG_MODE="644" sh -
```

### Install loxilb

Follow any of the getting started [guides](https://github.com/loxilb-io/loxilb?tab=readme-ov-file#getting-started) as per requirement. Check all the pods are up and running as expected :

```
$ kubectl get pods -A
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   coredns-6799fbcd5-4n4kl                  1/1     Running   0          56m
kube-system   kube-loxilb-b466c99bb-fpgll              1/1     Running   0          56m
kube-system   local-path-provisioner-6f5d79df6-f52sw   1/1     Running   0          56m
kube-system   loxilb-lb-gbkw7                          1/1     Running   0          30s
kube-system   metrics-server-54fd9b65b-dchv2           1/1     Running   0          56m
```

### Install ingress-nginx

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/baremetal/deploy.yaml
```

Double confirm :
```
$ kubectl get pods -A
NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx   ingress-nginx-admission-create-9vq66        0/1     Completed   0          113s
ingress-nginx   ingress-nginx-admission-patch-k4d74         0/1     Completed   1          113s
ingress-nginx   ingress-nginx-controller-845698f4f6-xq6hm   1/1     Running     0          113s
kube-system     coredns-6799fbcd5-4n4kl                     1/1     Running     0          59m
kube-system     kube-loxilb-b466c99bb-fpgll                 1/1     Running     0          59m
kube-system     local-path-provisioner-6f5d79df6-f52sw      1/1     Running     0          59m
kube-system     loxilb-lb-gbkw7                             1/1     Running     0          3m33s
kube-system     metrics-server-54fd9b65b-dchv2              1/1     Running     0          59m
```

### Install service, backend app and ingress rules

* Create a LB service for exposing ingress ports

```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller-loadbalancer
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
    - name: https
      port: 443
      protocol: TCP
      targetPort: 443
  type: LoadBalancer
```

Check the services created :

```
$ kubectl get svc -A
NAMESPACE       NAME                                    TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)                       AGE
default         kubernetes                              ClusterIP      10.43.0.1       <none>              443/TCP                       61m
ingress-nginx   ingress-nginx-controller                NodePort       10.43.114.138   <none>              80:30958/TCP,443:31794/TCP    3m22s
ingress-nginx   ingress-nginx-controller-admission      ClusterIP      10.43.107.66    <none>              443/TCP                       3m22s
ingress-nginx   ingress-nginx-controller-loadbalancer   LoadBalancer   10.43.27.248    llb-192.168.80.10   80:32218/TCP,443:32617/TCP    9s
kube-system     kube-dns                                ClusterIP      10.43.0.10      <none>              53/UDP,53/TCP,9153/TCP        61m
kube-system     loxilb-lb-service                       ClusterIP      None            <none>              11111/TCP,179/TCP,50051/TCP   5m2s
kube-system     metrics-server                          ClusterIP      10.43.20.55     <none>              443/TCP                       61m
```

At this point of time, all services exposed via ingress can be accessed via "192.168.80.10". This IP could be different as per use-case and scenario. This IP can then be associated with DNS for name based access.


* Create backend apps for ```domain1.loxilb.io``` and configure ingress rules with the following yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: site
spec:
  replicas: 1
  selector:
    matchLabels:
      name: site-nginx-frontend
  template:
    metadata:
      labels:
        name: site-nginx-frontend
    spec:
      containers:
        - name: blog
          image: ghcr.io/loxilb-io/nginx:stable
          imagePullPolicy: Always
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: site-nginx-service
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    name: site-nginx-frontend
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site-nginx-ingress
  annotations:
    #app.kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: domain1.loxilb.io
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: site-nginx-service
              port:
                number: 80
```

Double check status of pods, services and ingress:

```
$ kubectl get pods -A
NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
default         site-69d64fcd49-j4qhj                       1/1     Running     0          46s
ingress-nginx   ingress-nginx-admission-create-9vq66        0/1     Completed   0          8m21s
ingress-nginx   ingress-nginx-admission-patch-k4d74         0/1     Completed   1          8m21s
ingress-nginx   ingress-nginx-controller-845698f4f6-xq6hm   1/1     Running     0          8m21s
kube-system     coredns-6799fbcd5-4n4kl                     1/1     Running     0          66m
kube-system     kube-loxilb-b466c99bb-fpgll                 1/1     Running     0          66m
kube-system     local-path-provisioner-6f5d79df6-f52sw      1/1     Running     0          66m
kube-system     loxilb-lb-gbkw7                             1/1     Running     0          10m
kube-system     metrics-server-54fd9b65b-dchv2              1/1     Running     0          66m

$ kubectl get svc -A
NAMESPACE       NAME                                    TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)                       AGE
default         kubernetes                              ClusterIP      10.43.0.1       <none>              443/TCP                       67m
default         site-nginx-service                      ClusterIP      10.43.16.35     <none>              80/TCP                        108s
ingress-nginx   ingress-nginx-controller                NodePort       10.43.114.138   <none>              80:30958/TCP,443:31794/TCP    9m23s
ingress-nginx   ingress-nginx-controller-admission      ClusterIP      10.43.107.66    <none>              443/TCP                       9m23s
ingress-nginx   ingress-nginx-controller-loadbalancer   LoadBalancer   10.43.27.248    llb-192.168.80.10   80:32218/TCP,443:32617/TCP    6m10s
kube-system     kube-dns                                ClusterIP      10.43.0.10      <none>              53/UDP,53/TCP,9153/TCP        67m
kube-system     loxilb-lb-service                       ClusterIP      None            <none>              11111/TCP,179/TCP,50051/TCP   11m
kube-system     metrics-server                          ClusterIP      10.43.20.55     <none>              443/TCP                       67m


$ kubectl get ingress -A
NAMESPACE   NAME                 CLASS   HOSTS               ADDRESS     PORTS   AGE
default     site-nginx-ingress   nginx   domain1.loxilb.io   10.0.2.15   80      2m10s

```


* Now, lets create backend apps for ```domain2.loxilb.io``` and configure ingress rules with the following yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: site2
spec:
  replicas: 1
  selector:
    matchLabels:
      name: site-nginx-frontend2
  template:
    metadata:
      labels:
        name: site-nginx-frontend2
    spec:
      containers:
        - name: blog
          image: ghcr.io/loxilb-io/nginx:stable
          imagePullPolicy: Always
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: site-nginx-service2
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    name: site-nginx-frontend2
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site-nginx-ingress2
  annotations:
    #app.kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: domain2.loxilb.io
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: site-nginx-service2
              port:
                number: 80
```

Again, we can check the status of pods, service and ingress:
```
$ kubectl get pods -A
NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
default         site-69d64fcd49-j4qhj                       1/1     Running     0          9m12s
default         site2-7fff6cfbbf-8d6rp                      1/1     Running     0          2m34s
ingress-nginx   ingress-nginx-admission-create-9vq66        0/1     Completed   0          16m
ingress-nginx   ingress-nginx-admission-patch-k4d74         0/1     Completed   1          16m
ingress-nginx   ingress-nginx-controller-845698f4f6-xq6hm   1/1     Running     0          16m
kube-system     coredns-6799fbcd5-4n4kl                     1/1     Running     0          74m
kube-system     kube-loxilb-b466c99bb-fpgll                 1/1     Running     0          74m
kube-system     local-path-provisioner-6f5d79df6-f52sw      1/1     Running     0          74m
kube-system     loxilb-lb-gbkw7                             1/1     Running     0          18m
kube-system     metrics-server-54fd9b65b-dchv2              1/1     Running     0          74m

$ kubectl get svc -A
NAMESPACE       NAME                                    TYPE           CLUSTER-IP      EXTERNAL-IP         PORT(S)                       AGE
default         kubernetes                              ClusterIP      10.43.0.1       <none>              443/TCP                       75m
default         site-nginx-service                      ClusterIP      10.43.16.35     <none>              80/TCP                        9m32s
default         site-nginx-service2                     ClusterIP      10.43.107.99    <none>              80/TCP                        2m54s
ingress-nginx   ingress-nginx-controller                NodePort       10.43.114.138   <none>              80:30958/TCP,443:31794/TCP    17m
ingress-nginx   ingress-nginx-controller-admission      ClusterIP      10.43.107.66    <none>              443/TCP                       17m
ingress-nginx   ingress-nginx-controller-loadbalancer   LoadBalancer   10.43.27.248    llb-192.168.80.10   80:32218/TCP,443:32617/TCP    13m
kube-system     kube-dns                                ClusterIP      10.43.0.10      <none>              53/UDP,53/TCP,9153/TCP        75m
kube-system     loxilb-lb-service                       ClusterIP      None            <none>              11111/TCP,179/TCP,50051/TCP   18m
kube-system     metrics-server                          ClusterIP      10.43.20.55     <none>              443/TCP                       75m

$ kubectl get ingress -A
NAMESPACE   NAME                  CLASS   HOSTS               ADDRESS     PORTS   AGE
default     site-nginx-ingress    nginx   domain1.loxilb.io   10.0.2.15   80      9m49s
default     site-nginx-ingress2   nginx   domain2.loxilb.io   10.0.2.15   80      3m11s
```

## Test 

If you are testing locally you can simply add the following for dns resolution in your bastion/host :

```
$ tail -n 2  /etc/hosts
192.168.80.10   domain1.loxilb.io
192.168.80.10   domain2.loxilb.io

```
The above step is similar to adding A records in a DNS like route53.

* Finally, try to access the service "domain1.loxilb.io" :
```
$ curl -H "HOST: domain1.loxilb.io" domain1.loxilb.io
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

* And then try to access services domain2.loxilb.io:
```
$ curl -H "HOST: domain2.loxilb.io" domain2.loxilb.io
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
