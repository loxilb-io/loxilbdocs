
## Kubernetes service sharding with loxilb

This guide elaborates on how to shard Kubernetes services for efficient resource utilization and to maximize cluster performance. In the flat L2 networking (active-backup) [HA](https://docs.loxilb.io/latest/ha-deploy/) mode of loxilb, usually, one pod in a deployment is selected as the active pod. Thereafter, whenever services are created, the respective external VIP allocated is always associated with this master pod. When this master pod, or the node hosting it, fails, another master pod is selected, and all VIPs are then migrated to this new master pod.

This scheme works perfectly fine; however, if multiple loxilb pods are active, the backup pods usually don’t perform any functions, resulting in resource under-utilization. With service sharding, kube-loxilb (loxilb's operator) can effectively shard the services in an automated way across the available loxilb pods to maximize utilization.

![image](https://github.com/user-attachments/assets/2e61af4c-2a92-472b-9bbf-d60942a67695)

If any loxilb pod fails, the services associated with that particular pod are rebalanced to other available active pods without any traffic disruption.

![image](https://github.com/user-attachments/assets/0272e923-3cde-44b0-be16-1c1c7ac7c92c)

## How to deploy service sharding ?

In this example, we will have a Kubernetes cluster with 3 master nodes and 2 worker nodes. 

```
$ kubectl get nodes -A
NAME      STATUS   ROLES                       AGE     VERSION
master1   Ready    control-plane,etcd,master   6h29m   v1.30.5+k3s1
master2   Ready    control-plane,etcd,master   6h23m   v1.30.5+k3s1
master3   Ready    control-plane,etcd,master   6h21m   v1.30.5+k3s1
worker1   Ready    <none>                      6h13m   v1.30.5+k3s1
worker2   Ready    <none>                      6h12m   v1.30.5+k3s1

```

### Deploy loxilb in-cluster daemonset 
```
kubectl apply -f https://raw.githubusercontent.com/loxilb-io/kube-loxilb/refs/heads/main/manifest/in-cluster/loxilb.yaml
```

### Deploy kube-loxilb with appropriate args

Get the kube-loxilb manifest file :
```
wget https://github.com/loxilb-io/kube-loxilb/raw/main/manifest/in-cluster/kube-loxilb-nobgp.yaml
```
Change the args in this file : 
```
        args:               
        - --cidrPools=defaultPool=192.168.80.250/24                   
        - --setRoles=0.0.0.0
        - --setUniqueIP
        - --numZoneInstances=3
```
The argument ```--setUniqueIP``` is necessary to enable service sharding. This arg makes sure that each service gets an unique externalIP from IPAM. This external IP will be then associated across the different active loxilb in-cluster pods.

The argument ```--numZoneInstances``` should be set to the max. number of instances of loxilb deployment. In this example, we schedule loxilb pods in all the three master nodes so this argument is set to 3. Please note that we can schedule loxilb Daemonset in any nodes as necessary. Please check loxilb yaml manifest in the previous step.

Now, we can deploy kube-loxilb :

```
$ kubectl apply -f kube-loxilb-nobgp.yaml 
serviceaccount/kube-loxilb created
clusterrole.rbac.authorization.k8s.io/kube-loxilb created
clusterrolebinding.rbac.authorization.k8s.io/kube-loxilb created
deployment.apps/kube-loxilb created
```

Let's recap all the resources created so far:

```
kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-7b98449c4-n895r                   1/1     Running   0          7h6m
kube-system   kube-loxilb-95c8758b-gblq7                1/1     Running   0          51s
kube-system   local-path-provisioner-6795b5f9d8-zftg8   1/1     Running   0          7h6m
kube-system   loxilb-lb-ktq55                           1/1     Running   0          24m
kube-system   loxilb-lb-n5zfk                           1/1     Running   0          24m
kube-system   loxilb-lb-vp25b                           1/1     Running   0          24m
kube-system   metrics-server-cdcc87586-kk2st            1/1     Running   0          7h6m
```

We can quick check the HA sharding instances created by kube-loxilb :

```
$ kubectl exec -it -n kube-system loxilb-lb-ktq55 -- loxicmd get ha
| INSTANCE  | HASTATE |
|-----------|---------|
| default   | BACKUP  |
| llb-inst1 | BACKUP  |
| llb-inst2 | MASTER  |
$ kubectl exec -it -n kube-system loxilb-lb-vp25b -- loxicmd get ha
| INSTANCE  | HASTATE |
|-----------|---------|
| llb-inst1 | MASTER  |
| llb-inst2 | BACKUP  |
| default   | BACKUP  |
$ kubectl exec -it -n kube-system loxilb-lb-n5zfk -- loxicmd get ha
| INSTANCE  | HASTATE |
|-----------|---------|
| llb-inst2 | BACKUP  |
| llb-inst1 | BACKUP  |
| default   | MASTER  |
```
Hence, it creates three instances "default", "llb-inst1" and "llb-inst2"  and assigns a separate master instance for each.

### Creating services

We can create LoadBalancer services as usual. Here, kube-loxilb will choose a sharding instance by itself for the service. If an user needs to place the service in a specific service instance, they can do so with help of loxilb annotation as explained afterwards.

#### Create service without specifying any sharding instance 

```
cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: tcp-lb
  annotations:
    loxilb.io/lbmode: "onearm"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: tcp-test
  ports:
    - port: 57002
      targetPort: 80
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: tcp-test
  labels:
    what: tcp-test
spec:
  containers:
    - name: tcp-test
      image: ghcr.io/loxilb-io/nginx:stable
      ports:
        - containerPort: 80
EOF
```
Check the created service :

```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP          PORT(S)           AGE
kubernetes   ClusterIP      10.43.0.1       <none>               443/TCP           7h17m
tcp-lb       LoadBalancer   10.43.184.236   llb-192.168.80.250   57002:31819/TCP   27s
```
To figure out the exact instance selected, we need to check in each loxilb instance (no easy way out right now). 

```
sudo kubectl exec -it -n kube-system loxilb-lb-n5zfk -- loxicmd get ip | grep 192.168.80.250
|             | 192.168.80.250/32 |
```
In this case, kube-loxilb chooses the "default" shard and assigns the respective VIP to the current master loxilb pod for "default" shard. 

#### Create service by specifying sharding instance 

```
cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: tcp-lb2
  annotations:
    loxilb.io/lbmode: "onearm"
    loxilb.io/zoneinstance: "llb-inst2"
spec:
  externalTrafficPolicy: Local
  loadBalancerClass: loxilb.io/loxilb
  selector:
    what: tcp-test2
  ports:
    - port: 57002
      targetPort: 80
  type: LoadBalancer
---
apiVersion: v1
kind: Pod
metadata:
  name: tcp-test2
  labels:
    what: tcp-test2
spec:
  containers:
    - name: tcp-test2
      image: ghcr.io/loxilb-io/nginx:stable
      ports:
        - containerPort: 80
EOF
```

Here we are explicitly specifying ```loxilb.io/zoneinstance: "llb-inst2"``` to choose an instance. The created services as follows :
```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP          PORT(S)           AGE
kubernetes   ClusterIP      10.43.0.1       <none>               443/TCP           7h25m
tcp-lb       LoadBalancer   10.43.184.236   llb-192.168.80.250   57002:31819/TCP   8m7s
tcp-lb2      LoadBalancer   10.43.240.78    llb-192.168.80.251   57002:30253/TCP   10s
```
As per our earlier observation, the pod ```loxilb-lb-ktq55``` is the master of the instance ```llb-inst2```. So, we can check whether VIP has been associated with it or not :

```
sudo kubectl exec -it -n kube-system loxilb-lb-ktq55 -- loxicmd get ip | grep 192.168.80.251
|             | 192.168.80.251/32 |
```

We can also double confirm that these VIPS are at a time associated with only one loxilb pod. 

### Test

Let's do a quick test to check both these services are accessible from outside the cluster :

#### Service 1
```
curl http://192.168.80.250:57002
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

#### Service 2
```
curl http://192.168.80.251:57002
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
Curious readers can further analyze where each service traffic is heading with tools like tcpdump and left as an exercise !
