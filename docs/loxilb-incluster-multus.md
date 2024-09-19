## HowTo: Run loxilb in-cluster with multus based secondary services

This guide will explain how to run loxilb in in-cluster mode and provide LB services for multus based secondary networks. Some  users have used loxilb in external-mode to provide LB for multus secondary services as explained in this [blog](https://cloudybytes.medium.com/k8s-bringing-load-balancing-to-multus-workloads-with-loxilb-a0746f270abe). Here we will explore how do achieve the same with loxilb in-cluster mode.

### PreRequisites

A functioning K8s cluster with multiple nodes and multus installed. [Multus](https://github.com/k8snetworkplumbingwg/multus-cni) plugin can be installed by following the guide [here](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md). Also, for IPAM to work properly with multus, one might need to use [whereabouts](https://github.com/k8snetworkplumbingwg/whereabouts) plugin as well. In the beginning the state of the cluster should be similar to the following :

```
kube-flannel   kube-flannel-ds-cq9rx            1/1     Running            0                5h30m
kube-flannel   kube-flannel-ds-mmvrf            1/1     Running            0                5h36m
kube-flannel   kube-flannel-ds-tbdzx            1/1     Running            0                5h33m
kube-system    coredns-76f75df574-frjh5         1/1     Running            0                5h36m
kube-system    coredns-76f75df574-sxgjf         1/1     Running            0                5h36m
kube-system    etcd-master                      1/1     Running            0                5h36m
kube-system    kube-apiserver-master            1/1     Running            0                5h36m
kube-system    kube-controller-manager-master   1/1     Running            0                5h36m
kube-system    kube-multus-ds-98zq5             1/1     Running            0                3h57m
kube-system    kube-multus-ds-qrp4n             1/1     Running            0                3h57m
kube-system    kube-multus-ds-s9cnm             1/1     Running            0                3h57m
kube-system    kube-proxy-6k9kb                 1/1     Running            0                5h33m
kube-system    kube-proxy-lqclz                 1/1     Running            0                5h36m
kube-system    kube-proxy-qclfz                 1/1     Running            0                5h30m
kube-system    kube-scheduler-master            1/1     Running            1 (42m ago)      5h36m
kube-system    metrics-server-d4dc9c4f-pjp4t    1/1     Running            0                5h36m
```

### Topology

We will deploy a topology similar to the following :

![image](https://github.com/user-attachments/assets/94524f1e-6cc2-4407-8013-cd722a945816)

### Configuration
#### Create a network attachment definition with the following yaml :

```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: vlan5
spec:
  config: '{
    "name": "vlan5-net",
    "cniVersion": "0.3.1",
    "type": "vlan",
    "master": "eth1",
    "mtu": 1450,
    "vlanId": 5,
    "linkInContainer": false,
    "ipam": {
        "type": "host-local",
        "subnet": "123.123.123.0/24"
    },
    "dns": {
      "nameservers": [ "8.8.8.8" ]
    }
  }'
```

We can check the created attachment as follows :

```
$ kubectl describe network-attachment-definitions.k8s.cni.cncf.io vlan5
Name:         vlan5
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  Creation Timestamp:  2024-09-19T05:48:05Z
  Generation:          1
  Resource Version:    15678
  UID:                 0c0dc69c-bda2-4b49-bd82-1872f76ba2d3
Spec:
  Config:  { "name": "vlan5-net", "cniVersion": "0.3.1", "type": "vlan", "master": "eth1", "mtu": 1450, "vlanId": 5, "linkInContainer": false, "ipam": { "type": "host-local", "subnet": "123.123.123.0/24" }, "dns": { "nameservers": [ "8.8.8.8" ] } }
Events:    <none>
```

#### Install loxilb daemonset

To install loxilb as a daemonset, we can apply the following yaml :

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: loxilb-lb
  #namespace: kube-system
spec:
  selector:
    matchLabels:
      app: loxilb-app
  template:
    metadata:
      name: loxilb-lb
      labels:
        app: loxilb-app
      annotations:
        k8s.v1.cni.cncf.io/networks: vlan5
    spec:
      #hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
        #- key: "node-role.kubernetes.io/master"
        #operator: Exists
      - key: "node-role.kubernetes.io/control-plane"
        operator: Exists
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              #- key: "node-role.kubernetes.io/master"
              #  operator: Exists
              - key: "node-role.kubernetes.io/control-plane"
                operator: Exists
      containers:
      - name: loxilb-app
        image: "ghcr.io/loxilb-io/loxilb:latest"
        imagePullPolicy: Always
        command: [ "/root/loxilb-io/loxilb/loxilb" ]
        ports:
        - containerPort: 11111
        - containerPort: 179
        - containerPort: 50051
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
  #namespace: kube-system
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
  - name: loxilb-app-gobgp
    port: 50051
    targetPort: 50051
    protocol: TCP
```

The above creates a loxilb daemonset which for this example is labeled to run only in the master node. But the same example can be followed to let it run in any node necessary. However, in certain multus scenarios, it might be necessary to run loxilb pods in nodes which are not running the actual multus workloads. This is due to the fact how various vlan, mcvlan etc drivers work with multus. Hence appropirate planning has to be done beforehand in this regard. Also, we need to note that we add the network attachment created in previous step as an annotation "k8s.v1.cni.cncf.io/networks". This is to make sure loxilb has connectivity to the workload pods.

##### Example :

```
$ kubectl apply -f loxilb.yaml 
daemonset.apps/loxilb-lb created
service/loxilb-lb-service created

$ kubectl get pods -A | grep loxilb-lb
default        loxilb-lb-kdvj7                  1/1     Running            0                64s
```

#### Install kube-loxilb

kube-loxilb is the operator of loxilb. To install kube-loxilb, we can apply the following yaml :

```
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
      - namespaces
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
  - apiGroups:
      - bgppolicydefinedsets.loxilb.io
    resources:
      - bgppolicydefinedsetsservices
    verbs:
      - get
      - watch
      - list
      - create
      - update
      - delete
  - apiGroups:
      - bgppolicydefinition.loxilb.io
    resources:
      - bgppolicydefinitionservices
    verbs:
      - get
      - watch
      - list
      - create
      - update
      - delete
  - apiGroups:
      - bgppolicyapply.loxilb.io
    resources:
      - bgppolicyapplyservices
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
        - --cidrPools=defaultPool=123.123.123.123/32
        - --setRoles=0.0.0.0
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
:point_right: <b>It is to be noted that in this example kube-loxilb runs in kube-system namespace while loxilb runs in default namespace. This is due to the fact loxilb needs to run in a same namespace as the endpoint pod for multus connectivity to work properly. It is assumed that  kube-loxilb running in kube-system namespace can access loxilb pods just fine and there are no network policies in play here. </b>

##### Example :

```
$ kubectl apply -f kube-loxilb.yaml 
serviceaccount/kube-loxilb created
clusterrole.rbac.authorization.k8s.io/kube-loxilb created
clusterrolebinding.rbac.authorization.k8s.io/kube-loxilb created
deployment.apps/kube-loxilb created

$ kubectl get pods -A | grep kube-loxilb
kube-system    kube-loxilb-d97c89f4c-8zvhk      1/1     Running            0                74s
```

#### Create a sample Pod with multus attachment

We can use the following yaml for this :

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  labels:
    app: pod-01
  annotations:
    k8s.v1.cni.cncf.io/networks: vlan5
spec:
  containers:
    - name: nginx
      image: ghcr.io/loxilb-io/nginx:stable
      ports:
        - containerPort: 80
```

##### Example :

```
$ kubectl apply -f multus-pod.yaml
pod/pod-01 created

$ kubectl describe pods pod-01
Name:             pod-01
Namespace:        default
Priority:         0
Service Account:  default
Node:             worker2/192.168.80.202
Start Time:       Thu, 19 Sep 2024 09:14:42 +0000
Labels:           app=pod-01
Annotations:      k8s.v1.cni.cncf.io/network-status:
                    [{
                        "name": "cbr0",
                        "interface": "eth0",
                        "ips": [
                            "10.244.2.20"
                        ],
                        "mac": "56:aa:2f:db:95:82",
                        "default": true,
                        "dns": {},
                        "gateway": [
                            "10.244.2.1"
                        ]
                    },{
                        "name": "default/vlan5",
                        "interface": "net1",
                        "ips": [
                            "123.123.123.17"
                        ],
                        "mac": "08:00:27:55:ec:f5",
                        "dns": {
                            "nameservers": [
                                "8.8.8.8"
                            ]
                        }
                    }]
                  k8s.v1.cni.cncf.io/networks: vlan5
Status:           Running
IP:               10.244.2.20
IPs:
  IP:  10.244.2.20
Containers:
  nginx:
    Container ID:   cri-o://46932f3cb26facd041bbeed16e4373e7efa8b5f89b2ca20dfbcffad26d2b88dd
    Image:          ghcr.io/loxilb-io/nginx:stable
    Image ID:       ghcr.io/loxilb-io/nginx@sha256:46f036b0be0dfe50ca81ba2428390570a4c179830e29e67cd5872ecf8ecbfbb3
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 19 Sep 2024 09:14:43 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-s8mjs (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-s8mjs:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason          Age   From               Message
  ----    ------          ----  ----               -------
  Normal  Scheduled       60s   default-scheduler  Successfully assigned default/pod-01 to worker2
  Normal  AddedInterface  60s   multus             Add eth0 [10.244.2.20/24] from cbr0
  Normal  AddedInterface  60s   multus             Add net1 [123.123.123.17/24] from default/vlan5
  Normal  Pulled          60s   kubelet            Container image "ghcr.io/loxilb-io/nginx:stable" already present on machine
  Normal  Created         60s   kubelet            Created container nginx
  Normal  Started         60s   kubelet            Started container nginx

```

#### Create a service to access the pod from outside

We can use the following yaml :

```
apiVersion: v1
kind: Service
metadata:
  name: multus-service
  annotations:
    loxilb.io/multus-nets: vlan5
    loxilb.io/lbmode: "onearm"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    app: pod-01
  ports:
    - port: 55002
      targetPort: 80
  type: LoadBalancer
```

Please make sure that loadBalancerClass is set to ```loxilb.io/loxilb``` so that loxilb can co-exist with any other LB which one might be using for primary services. And finally the annotation ```loxilb.io/multus-nets``` which denotes the network attachement definition name.

##### Example :

```
$ kubectl apply -f  /vagrant/multus/multus-service.yml 
service/multus-service created

$ kubectl get svc 
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP           PORT(S)                       AGE
kubernetes          ClusterIP      10.245.0.1      <none>                443/TCP                       6h19m
loxilb-lb-service   ClusterIP      None            <none>                11111/TCP,179/TCP,50051/TCP   21m
multus-service      LoadBalancer   10.245.18.238   llb-123.123.123.123   55002:32488/TCP               30s
```

#### Test the service

Now we can access the service from any host connected to this secondary network.

```
$ curl http://123.123.123.123:55002
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
