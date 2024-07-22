### Deploy LoxiLB with multi-cloud HA support

LoxiLB supports stateful HA configuration in various cloud environments such as AWS. Especially for AWS, one can configure HA using the Floating IP [pattern](https://docs.aws.amazon.com/whitepapers/latest/real-time-communication-on-aws/floating-ip-pattern-for-ha-between-activestandby-stateful-servers.html), together with [LoxiLB](https://github.com/loxilb-io/loxilb).

### Overall Scenario

Overall scenario will look like this:    

![image](photos/loxilb-k8s-arch-multi-cloud-ha.svg)

Setup configuration for Multi-Cloud/Multi-region will be similar to [Multi-AZ-HA configuration](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/aws-multi-az.md)

#### Important considerations
* The steps mentioned in the above documentation are for a single AWS region. For cross-region, similar configuration needs to be done in other AWS regions.
* Two LoxiLB instances - loxilb1 and loxilb2 will be deployed in different AZs per region. These two loxilbs form a HA pair and operate in active-backup roles.
* One instance of kube-loxilb will be deployed per region. 
* Every region’s private CIDR will be different and one region’s privateCIDR should be reachable to others through VPC peering.
* As elastic IP is bound to a particular region, it is impossible to provide connection synchronization for cross-region HA. Only warm stand-by cross-region HA is supported.
* Full Support for elasticIP in GCP is not available yet. For testing HA with GCP, run single loxilb and kube-loxilb with standard configuration. There will not be any privateCIDR in kube-loxilb.yaml. Mention loxilb IP as externalCIDR.

To summarize, when a failover occurs within the region, the public ElasticIP address is always associated to the active LoxiLB instance, so users who were previously accessing EKS using the same ElasticIP address can continue to do so without being affected by any node failure or other issues. When a region-wise failover occurs, DNS will redirect the requests to the different region.

### An example configuration
Please follow the steps to create cluster and prepare VM instances mentioned [here](https://github.com/loxilb-io/loxilbdocs/blob/main/docs/aws-multi-az.md).

### Configuring LoxiLB EC2 Instances

#### kube-loxilb deployment

kube-loxilb is a K8s operator for LoxiLB. Download the manifest file required for your deployment in EKS.
Create the ServiceAccount and other necessary settings for the cluster before start deploying kube-loxilb per cluster.
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
```


#### Change the args inside the yaml below(as applicable) and install it for every region.

kube-loxilb-osaka-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-loxilb-osaka
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
        image: ghcr.io/loxilb-io/kube-loxilb:aws-support
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        - --loxiURL=http://192.168.218.60:11111,192.168.218.61:11111
        - --externalCIDR=13.208.X.X/32
        - --privateCIDR=192.168.248.254/32
        - --setLBMode=2
        - --zone=osaka
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

kube-loxilb-seoul-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-loxilb-seoul
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
        image: ghcr.io/loxilb-io/kube-loxilb:aws-support
        imagePullPolicy: Always
        command:
        - /bin/kube-loxilb
        args:
        - --loxiURL=http://192.168.119.11:11111,192.168.119.12:11111
        - --externalCIDR=14.112.X.X/32
        - --privateCIDR=192.168.150.254/32
        - --setLBMode=2
        - --zone=seoul
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
For every region, Edit kube-loxilb-<i>region</i>.yaml
* Modify loxiURL with the IPs of the LoxiLB EC2 instances created in the region above.
* For externalCIDR, specify the Elastic IP created above.
* PrivateCIDR specifies the VIP that will be associated with the Elastic IP.

#### Run LoxiLB Pods
##### Install docker on LoxiLB instance(s)

LoxiLB is deployed as a container on each instance. To use containers, docker must first be installed on the instance. Docker installation guide can be found [here](https://docs.docker.com/engine/install/ubuntu/)

#### Running LoxiLB container

The following command is for a LoxiLB instance (loxilb1) using subnet-a.

```
sudo docker run -u root --cap-add SYS_ADMIN \
  --restart unless-stopped \
  --net=host \
  --privileged \
  -dit \
  -v /dev/log:/dev/log -e AWS_REGION=ap-northeast-3 \
  --name loxilb \
  ghcr.io/loxilb-io/loxilb:aws-support \
  --cloud=aws --cloudcidrblock=192.168.248.0/24 --cluster=192.168.218.61 --self=0
```

* In the cloudcidrblock option, specify the IP band that includes the VIP set in kube-loxilb's privateCIDR. master LoxiLB uses the value set here to create a new subnet in the AZ where it is located and uses it for HA operation.
* The cluster option specifies the IP of the partner instance (LoxiLB instance using subnet-b) for which HA is configured.
* The self option is set to 0. It is just a identier used internally to identify each instance

Similarily we can run loxilb2 instance in the second EC2 instance using subnet-b:   

```
sudo docker run -u root --cap-add SYS_ADMIN \
  --restart unless-stopped \
  --net=host \
  --privileged \
  -dit \
  -v /dev/log:/dev/log -e AWS_REGION=ap-northeast-3 \
  --name loxilb \
  ghcr.io/loxilb-io/loxilb:aws-support \
  --cloud=aws --cloudcidrblock=192.168.248.0/24 --cluster=192.168.218.60 --self=1
```

For each instance, HA status can be checked as follows:

When the container runs, you can check the HA status as follows:

```
ubuntu@ip-192-168-218-60:~$ sudo docker exec -ti loxilb bash
root@ip-192-168-228-108:/# loxicmd get ha
| INSTANCE | HASTATE |
|----------|---------|
| default  | MASTER  |
root@ip-192-168-218-60:/#
```

#### Creating a  service

Let's create a test service to test HA functionality. Below are the manifest files for the nginx pod and service that we will use for testing.

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
After creating an nginx service with the above,  weu can see that the ElasticIP has been designated as the externalIP of the service.

```
LEIS6N3:~/workspace/aws-demo$ kubectl apply -f nginx.yaml
service/nginx-lb1 created
pod/nginx-test created
LEIS6N3:~/workspace/aws-demo$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP         PORT(S)           AGE
kubernetes   ClusterIP      10.100.0.1     <none>              443/TCP           22h
nginx-lb1    LoadBalancer   10.100.178.3   llb-13.208.X.X      55002:32403/TCP   15s
```
