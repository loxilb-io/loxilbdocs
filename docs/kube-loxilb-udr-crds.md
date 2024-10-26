## kube-loxilb dynamic configuration with Kubernetes CRD

By default, kube-loxilb's operation is controller by its initial manifest file. But it might be also necessary to configure some of its options without stopping kube-loxilb. This is supported using config CRDs exposed by kube-loxilb. Currently the following configuration can be added/updated/deleted using such CRDS:

* Create/Update/Delete loxilb URL when running in external mode
* Create/Update/Delete IPAM Pool definitions to use with services

### Prerequisite

As a first step, we need to add the necessary CRD definitions in Kubernetes:

```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/crds/loxilb-url-crd.yaml
```

### Create/Update/Delete loxilb URL when running in external mode

To add a loxilb instance to the kube-loxilb, we can use the following yaml file (save as loxiurl1.yaml) :

```
apiVersion: "loxiurl.loxilb.io/v1"
kind: LoxiURL
metadata:
  name: llb-10.10.10.1
spec:
  loxiURL: http://10.10.10.1:11111
  zone: llb
  type: default
```
Apply to add :
```
kubectl apply -f loxiurl1.yaml
```
The above will add loxilb instance running in external mode reachable at http://10.10.10.1:11111 to this kube-loxilb. We can use both http or https.

To check, we can use :
```  
kubectl get loxiurl
NAME             AGE
llb-10.10.10.1   5h21m
```
After this, kube-loxilb will pair with respective loxilb instance as mentioned.

### Create/Update/Delete IPAM Pool definitions to use with services

To add a new IPAM CIDR pool to the kube-loxilb, we can use the following yaml file (save as loxipool1.yaml) :

```
apiVersion: "loxiurl.loxilb.io/v1"
kind: LoxiURL
metadata:
  name: pool5
spec:
  loxiURL: 12.12.12.0/24
  zone: llb
  type: cidrpool
```

Apply to add :
```
kubectl apply -f loxipool1.yaml
```
When creating a Kubernetes service, we can then select this IPAM pool for external address allocation to this service :

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb1
  annotations:
    loxilb.io/poolSelect: pool5
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: nginx-test
  ports:
    - port: 80
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
  nodeSelector:
    node: wlznode02
  containers:
    - name: nginx-test
      image: nginx
      imagePullPolicy: Always
      ports:
        - containerPort: 80
```

