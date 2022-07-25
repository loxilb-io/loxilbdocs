Howto - ccm plugin
======
loxi-ccm is a [cloud-manager][ccmLink] that provides kubernetes with loxilb load balancer.
kubernetes provides the [cloud-provider interface][cloudProviderLink] for the implementation of external cloud provider-specific  logic, and loxi-ccm is an implementation of the cloud-provider interface.

[ccmLink]: https://kubernetes.io/docs/concepts/architecture/cloud-controller/ "k8s Cloud Manager concept"
[cloudProviderLink]: https://github.com/kubernetes/cloud-provider "k8s cloud-provider github page"

Typical loxi-ccm deployment topology
======
As seen in the [loxilb architecture documentation][loxiArchLink], loxi-ccm is logically shown as part of the loxilb cluster. But it's actually running on the k8s master/control-plane node.

loxi-ccm implements the k8s load balancer service function using RESTful API of loxilb. 
When a user creates a k8s load balancer type service, loxi-ccm allocates an IP from the registered External IP subnet Pool. loxi-ccm sets rules in loxilb to allow service access from external with the assigned IP.
In other words, loxi-ccm needs two information.

1. loxilb API server address
2. External IP Subnet

These informations are managed through k8s ConfigMap. loxi-ccm users should modify this informations to suit your environment.

[loxiArchLink]: https://github.com/loxilb-io/loxilbdocs/blob/main/docs/arch.md "loxi architecture document"

Deploy loxi-ccm on kubernetes
===========
The guide below has been tested in environment on Ubuntu 20.04, kubernetes v1.24 (calico CNI)

### 1. Modify k8s ConfigMap
In the manifests/loxi-ccm.yaml manifests file, the ConfigMap is defined as follows
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: loxilb-config
  namespace: kube-system
data:
  apiServerURL: "http://192.168.20.54:11111"
  externalIPcidr: 123.123.123.0/24
---
```
The ConfigMap has two values: apiServerURL and externalIPcidr.

- apiServerURL : API Server address of loxilb.
- externalIPcidr : Subnet band to be allocated by loxilb as External IP of the load balancer.

apiServerURL and externalIPcidr must be modified according to the environment of the user using loxi-ccm.

### 2. Deploy loxi-ccm
Once you have modified ConfigMap, you can deploy loxi-ccm using the loxi-ccm.yaml manifest file.
Run the following command on the kubernetes you want to deploy.
```
kubectl apply -f loxi-ccm.yaml
```
After entering the command, check whether loxi-cloud-controller-manager is created in the daemonset of the kube-system namespace.

Manual build
======
If you want to build loxi-ccm manually, do the following:
### 1. build
```
./build.sh
```
### 2. Build & upload container image
Below is an example. This case use docker to build container images, and images is uploaded to docker hub.
```
TAG="0.1"
DOCKER_ID=YOUR_DOCKER_ID
sudo docker build -t $DOCKER_ID/loxi-ccm:$TAG -f ./Dockerfile .
sudo docker push $DOCKER_ID/loxi-ccm:$TAG
```

### 3. create loxi-ccm daemonset using custom image
In the DaemonSet section of the ./manifests/loxi-ccm.yaml file, 
change the image name to a custom image. (spec.template.spec.containers.image)
```
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: loxi-cloud-controller-manager
  name: loxi-cloud-controller-manager
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: loxi-cloud-controller-manager
  template:
    metadata:
      labels:
        k8s-app: loxi-cloud-controller-manager
    spec:
      serviceAccountName: loxi-cloud-controller-manager
      containers:
        - name: loxi-cloud-controller-manager
          imagePullPolicy: Always
          # for in-tree providers we use k8s.gcr.io/cloud-controller-manager
          # this can be replaced with any other image for out-of-tree providers
          image: {DOCKER_ID}/loxi-ccm:{TAG}
          command:
            - /bin/loxi-cloud-controller-manager
```
