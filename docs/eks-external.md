## Create an EKS cluster with ingress access enabled by loxilb (external-mode)

This document details the steps to create an EKS cluster and allow external ingress access using loxilb running in external mode. loxilb will run as EC2 instances in EKS cluster's VPC while loxilb's operator, kube-loxilb, will run as a replica-set inside EKS cluster.

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

## Deploy loxilb as EC2 instances in EKS's VPC

- Create a file ```launch-loxilb.sh``` with the following contents (in bastion node)
```
#!/bin/bash
sudo apt-get update && apt-get install -y snapd
sudo snap install docker
sleep 30
sudo docker run -u root --cap-add SYS_ADMIN --net=host --restart unless-stopped --privileged -dit -v /dev/log:/dev/log --name loxilb ghcr.io/loxilb-io/loxilb:latest
```
- Deploy loxilb ec2 instance(s) using the above init-script
```
$ aws ec2 run-instances --image-id ami-01ed8ade75d4eee2f --count 1 --instance-type t3.medium --key-name aws-netlox --security-group-ids sg-0e2638db05b256476 --subnet-id subnet-0109b973f5f674f99 --associate-public-ip-address --user-data file://launch-loxilb.sh
```
#### Note : subnet-id should be any subnet with public access enabled from the EKS cluster. Rest of the args can be changed as applicable

- Double confirm loxilb EC2 instances are running properly in amazon aws console or using aws cli.
- Disable source/dest check of the loxilb EC2 instances

```
aws ec2 modify-network-interface-attribute --network-interface-id eni-02e1cbfa022eb0901 --no-source-dest-check
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
        - --loxiURL=http://192.168.31.175:11111
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
####  Note1: --externalCIDR args can be set to any Public IP address via which any of the worker nodes can be accessed. It can be also set to simply 0.0.0.0/32 which means LB will be performed on any of the nodes where loxilb runs. The decision of which loxilb node/instance will be chosen as ingress in this case can be done by Route53/DNS.
####  Note2: --loxiURL args should be set to privateIP address(es) of the loxilb ec2 instances accessible from the EKS cluster. Currently, kube-loxilb can't autodetect the EC2 instances running loxilb in external mode.

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
NAMESPACE     NAME                           READY   STATUS    RESTARTS   AGE
kube-system   aws-node-2fpm4                 2/2     Running   0          14m
kube-system   aws-node-6vhlr                 2/2     Running   0          14m
kube-system   aws-node-9kzb2                 2/2     Running   0          14m
kube-system   aws-node-vvkq5                 2/2     Running   0          14m
kube-system   coredns-5ff5b8d45c-gj9kj       1/1     Running   0          21m
kube-system   coredns-5ff5b8d45c-p64fd       1/1     Running   0          21m
kube-system   kube-proxy-5j9gf               1/1     Running   0          14m
kube-system   kube-proxy-5tm8w               1/1     Running   0          14m
kube-system   kube-proxy-894k9               1/1     Running   0          14m
kube-system   kube-proxy-xgfb8               1/1     Running   0          14m
kube-system   kube-loxilb-6477d6897f-vz74f   1/1     Running   0           5m
```

## Install a test service 

- Create a file nginx.yml with the following contents:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb1
  annotations:
    loxilb.io/usepodnetwork : "yes"
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
- Please not the usage of annotation ```loxilb.io/usepodnetwork : "yes"```. This would imply loxilb will directly use PodIP and TargetPort to reach out as its end-points. This feature is only available with EKS currently and should provide additional performance boost.
- Deploy test nginx service to EKS
```
$ kubectl apply -f nginx.yml
service/nginx-lb1 created
```
- Check the state of the EKS cluster
```
$ kubectl get pods -A 
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
default       nginx-test                     1/1     Running   0          50s
kube-system   aws-node-2fpm4                 2/2     Running   0          14m
kube-system   aws-node-6vhlr                 2/2     Running   0          14m
kube-system   aws-node-9kzb2                 2/2     Running   0          14m
kube-system   aws-node-vvkq5                 2/2     Running   0          14m
kube-system   coredns-5ff5b8d45c-gj9kj       1/1     Running   0          21m
kube-system   coredns-5ff5b8d45c-p64fd       1/1     Running   0          21m
kube-system   kube-proxy-5j9gf               1/1     Running   0          14m
kube-system   kube-proxy-5tm8w               1/1     Running   0          14m
kube-system   kube-proxy-894k9               1/1     Running   0          14m
kube-system   kube-proxy-xgfb8               1/1     Running   0          14m
kube-system   kube-loxilb-6477d6897f-vz74f   1/1     Running   0           5m
```

- Check the external service for service ingress (via loxilb)
```
$  kubectl get svc
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
kubernetes   ClusterIP      10.100.0.1       <none>        443/TCP           10h
nginx-lb1    LoadBalancer   10.100.244.105   llbanyextip   55005:30055/TCP   24s
```

## Test the service 

- Try to access the service from outside (internet). We can use any public IP associated with any of the loxilb ec2 instances

```
$ curl http://3.37.191.xx:55005  
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
