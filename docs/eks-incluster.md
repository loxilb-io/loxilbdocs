## Create an EKS cluster with ingress access enabled by loxilb (incluster-mode)

This document details the steps to create an EKS cluster and allow external ingress access using loxilb running in incluster mode. loxilb will run as a daemon-set in all the worker nodes while loxilb's operator, kube-loxilb, will run as a replica-set.

Although loxilb has built-in support for associating (floating) AWS EIPs to private subnet addresses of EC2 instances, this is not considered in this particular scenario. But if it is needed, the functionality can be enabled by changing a few parameters in yaml config files.

### Create EKS cluster with 4 worker nodes from a bastion node inside your VPC

- It is assumed that aws-cli, kubectl and eksctl are installed in a bastion node

```
$ eksctl create cluster --version 1.24 --name loxilb-demo --vpc-nat-mode Single --region ap-northeast-2 --node-type t3.small --nodes 4 --with-oidc --managed
```

- Create kube config for kubectl access 
```
$ aws eks update-kubeconfig --region ap-northeast-2 --name loxilb-demo
```
- Double confirm the cluster created 
```
$ kubectl get pods -A
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-2fpm4             2/2     Running   0          14m
kube-system   aws-node-6vhlr             2/2     Running   0          14m
kube-system   aws-node-9kzb2             2/2     Running   0          14m
kube-system   aws-node-vvkq5             2/2     Running   0          14m
kube-system   coredns-5ff5b8d45c-gj9kj   1/1     Running   0          21m
kube-system   coredns-5ff5b8d45c-p64fd   1/1     Running   0          21m
kube-system   kube-proxy-5j9gf           1/1     Running   0          14m
kube-system   kube-proxy-5tm8w           1/1     Running   0          14m
kube-system   kube-proxy-894k9           1/1     Running   0          14m
kube-system   kube-proxy-xgfb8           1/1     Running   0          14m
```

## Deploy loxilb as a daemon-set 

- Create a file loxilb.yml with the following contents
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: loxilb-lb
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: loxilb-app
  template:
    metadata:
      name: loxilb-lb
      labels:
        app: loxilb-app
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: loxilb-app
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        command: [ "/root/loxilb-io/loxilb/loxilb", "--egr-hooks", "--blacklist=cni[0-9a-z]|veth.|flannel.|eni." ]
        ports:
        - containerPort: 11111
        - containerPort: 179
        securityContext:
          privileged: true
          capabilities:
            add:
              - SYS_ADMIN
---
apiVersion: v1
kind: Service
metadata:
  name: loxilb-lb-service
  namespace: kube-system
spec:
  clusterIP: None
  selector:
    app: loxilb-app
  ports:
  - name: loxilb-app
    port: 11111
    targetPort: 11111
    protocol: TCP
  - name: loxilb-app-bgp
    port: 179
    targetPort: 179
    protocol: TCP
```
- Deploy loxilb
```
$ kubectl apply -f  loxilb.yml
daemonset.apps/loxilb-lb created
service/loxilb-lb-service created

```
- Double confirm loxilb pods are running properly as a daemonset

```
$ kubectl get pods -A
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-2fpm4             2/2     Running   0          19m
kube-system   aws-node-6vhlr             2/2     Running   0          19m
kube-system   aws-node-9kzb2             2/2     Running   0          19m
kube-system   aws-node-vvkq5             2/2     Running   0          19m
kube-system   coredns-5ff5b8d45c-gj9kj   1/1     Running   0          26m
kube-system   coredns-5ff5b8d45c-p64fd   1/1     Running   0          26m
kube-system   kube-proxy-5j9gf           1/1     Running   0          19m
kube-system   kube-proxy-5tm8w           1/1     Running   0          19m
kube-system   kube-proxy-894k9           1/1     Running   0          19m
kube-system   kube-proxy-xgfb8           1/1     Running   0          19m
kube-system   loxilb-lb-7s45t            1/1     Running   0          18s
kube-system   loxilb-lb-fp6nv            1/1     Running   0          18s
kube-system   loxilb-lb-pbzql            1/1     Running   0          18s
kube-system   loxilb-lb-zzth8            1/1     Running   0          18s
```

## Deploy loxilb's operator (kube-loxilb)

- Create a file kube-loxilb.yml with the following contents
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-loxilb
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-loxilb
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
      - watch
      - list
      - patch
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - watch
      - list
      - patch
  - apiGroups:
      - ""
    resources:
      - endpoints
      - namespaces
      - services
      - services/status
    verbs:
      - get
      - watch
      - list
      - patch
      - update
  - apiGroups:
      - gateway.networking.k8s.io
    resources:
      - gatewayclasses
      - gatewayclasses/status
      - gateways
      - gateways/status
      - tcproutes
      - udproutes
    verbs: ["get", "watch", "list", "patch", "update"]
  - apiGroups:
      - discovery.k8s.io
    resources:
      - endpointslices
    verbs:
      - get
      - watch
      - list
  - apiGroups:
      - authentication.k8s.io
    resources:
      - tokenreviews
    verbs:
      - create
  - apiGroups:
      - authorization.k8s.io
    resources:
      - subjectaccessreviews
    verbs:
      - create
  - apiGroups:
      - bgppeer.loxilb.io
    resources:
      - bgppeerservices
    verbs:
      - get
      - watch
      - list
      - create
      - update
      - delete

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-loxilb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-loxilb
subjects:
  - kind: ServiceAccount
    name: kube-loxilb
    namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-loxilb
  namespace: kube-system
  labels:
    app: kube-loxilb-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-loxilb-app
  template:
    metadata:
      labels:
        app: kube-loxilb-app
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
      priorityClassName: system-node-critical
      serviceAccountName: kube-loxilb
      terminationGracePeriodSeconds: 0
      containers:
      - name: kube-loxilb
        image: ghcr.io/loxilb-io/kube-loxilb:latest
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        - --externalCIDR=0.0.0.0/32
        - --setLBMode=5
        #- --setRoles:0.0.0.0
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
```
#### Note: --externalCIDR args can be set to any Public IP address via which any of the worker nodes can be accessed. It can be also set to simply 0.0.0.0/32 which means LB will be performed on any of the nodes where loxilb runs. The decision of which loxilb node/instance will be chosen as ingress in this case can be done by Route53/DNS.

- Deploy kube-loxilb to EKS cluster
```
$ kubectl apply -f  kube-loxilb.yml
serviceaccount/kube-loxilb created
clusterrole.rbac.authorization.k8s.io/kube-loxilb created
deployment.apps/kube-loxilb created
```
- Check the state of the EKS cluster
```
$ kubectl get pods -A 
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
kube-system   aws-node-2fpm4                2/2     Running   0          35m
kube-system   aws-node-6vhlr                2/2     Running   0          35m
kube-system   aws-node-9kzb2                2/2     Running   0          35m
kube-system   aws-node-vvkq5                2/2     Running   0          35m
kube-system   coredns-5ff5b8d45c-gj9kj      1/1     Running   0          42m
kube-system   coredns-5ff5b8d45c-p64fd      1/1     Running   0          42m
kube-system   kube-loxilb-c7cd4fccd-hjg8w   1/1     Running   0          116s
kube-system   kube-proxy-5j9gf              1/1     Running   0          35m
kube-system   kube-proxy-5tm8w              1/1     Running   0          35m
kube-system   kube-proxy-894k9              1/1     Running   0          35m
kube-system   kube-proxy-xgfb8              1/1     Running   0          35m
kube-system   loxilb-lb-7s45t               1/1     Running   0          16m
kube-system   loxilb-lb-fp6nv               1/1     Running   0          16m
kube-system   loxilb-lb-pbzql               1/1     Running   0          16m
kube-system   loxilb-lb-zzth8               1/1     Running   0          16m
```

## Install a test service 

- Create a file nginx.yml with the following contents:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb1
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: nginx-test
  ports:
    - port: 55002
      targetPort: 80
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  labels:
    what: nginx-test
spec:
  containers:
    - name: nginx-test
      image: nginx:stable
      ports:
        - containerPort: 80
```
- Deploy test nginx service to EKS
```
$ kubectl apply -f nginx.yml
service/nginx-lb1 created
```
- Check the state of the EKS cluster
```
$ kubectl get pods -A 
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
default       nginx-test                    1/1     Running   0          50s
kube-system   aws-node-2fpm4                2/2     Running   0          39m
kube-system   aws-node-6vhlr                2/2     Running   0          39m
kube-system   aws-node-9kzb2                2/2     Running   0          39m
kube-system   aws-node-vvkq5                2/2     Running   0          39m
kube-system   coredns-5ff5b8d45c-gj9kj      1/1     Running   0          46m
kube-system   coredns-5ff5b8d45c-p64fd      1/1     Running   0          46m
kube-system   kube-loxilb-c7cd4fccd-hjg8w   1/1     Running   0          6m13s
kube-system   kube-proxy-5j9gf              1/1     Running   0          39m
kube-system   kube-proxy-5tm8w              1/1     Running   0          39m
kube-system   kube-proxy-894k9              1/1     Running   0          39m
kube-system   kube-proxy-xgfb8              1/1     Running   0          39m
kube-system   loxilb-lb-7s45t               1/1     Running   0          20m
kube-system   loxilb-lb-fp6nv               1/1     Running   0          20m
kube-system   loxilb-lb-pbzql               1/1     Running   0          20m
kube-system   loxilb-lb-zzth8               1/1     Running   0          20m
```

- Check the external service for service ingress (via loxilb)
```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
kubernetes   ClusterIP      10.100.0.1      <none>        443/TCP           6h19m
nginx-lb1    LoadBalancer   10.100.63.175   llbanyextip   55002:32704/TCP   4hs
```

## Test the service 

- Try to access the service from outside (internet). We can use any public IP associated with the cluster (worker) nodes

```
$ curl http://43.201.76.xx:55002 
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

#### Note - We would need to make sure AWS security groups are setup properly to allow access for ingress traffic.

## Restricting loxilb service for a local-zone node-group

For limiting loxilb services to a specific node group of a local-zone, we can use kubenetes node-labels to limit the endpoints of that service to that node-group only. For example, if all the nodes in a local-zone node-groups have a label ```node.kubernetes.io/local-zone2=true```, then we can create a loxilb service with a following annotation :

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb1
  annotations:
    loxilb.io/nodelabel: "node.kubernetes.io/local-zone2"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: nginx-test
  ports:
    - port: 55002
      targetPort: 80
  type: LoadBalancer
```
This will make sure that loxilb will pick only the endpoint nodes which belong to that node-group only.
