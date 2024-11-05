
# HTTPS guide for loxilb API

By default loxilb uses plain http for its API operation. Please refere to the arch [guide](https://docs.loxilb.io/latest/kube-loxilb/#overall-topology) for more info. This guide will detail the steps needed to enable https in both loxilb (server-mode) and kube-loxilb (client-mode). For enabling https, we need to have proper certificate and keys in place. We will use popular tool [mkcert](https://github.com/FiloSottile/mkcert) to configure locally-trusted development certificates. One could also use tools like [letsencrypt](https://letsencrypt.org) for production grade certificates. Nonetheless, overall process is the same.

## Generate the certificates 

```
mkdir cert
cd cert
wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64
chmod +x mkcert-v1.4.3-linux-amd64
mv mkcert-v1.4.3-linux-amd64 mkcert
mkdir loxilb.io
export CAROOT=`pwd`/loxilb
./mkcert -install
./mkcert 192.168.80.9
cp loxilb/rootCA.pem ./rootCA.crt
mv 192.168.80.9.pem ./server.crt
mv 192.168.80.9-key.pem ./server.key
cd - 
```

The above creates SSL certificate with IP in the SAN(Subject Alternative Name). In this example, we assume loxilb  will run in a host with private IP address ```192.168.80.9```.

## Run loxilb with the certificates 

To run loxilb, we can simply mount the cert directory created earlier into appropriate mount point of the loxilb pod/docker :

```
docker run -u root --cap-add SYS_ADMIN   --restart unless-stopped --privileged -dit -v /dev/log:/dev/log -v `pwd`/cert:/opt/loxilb/cert/ --net=host --name loxilb ghcr.io/loxilb-io/loxilb:latest --tls
```
The http only api channel is still available at this point. We can restrict its availability only inside the pod by adding the argument ```--host=127.0.0.1```. If loxilb is running in-cluster, we can use volume mounts to the loxilb pod. The volume mount option is similar to what will be used for kube-loxilb as explained below. 

## Run kube-loxilb with updated rootCA 

Any https client needs to have the rootCA certificate to validate the authenticity of the certificates presented by a server. Since, we are using local certificates we need to add the local rootCA to the system store of the kube-loxilb pod.

As a first step, we need to copy the ```rootCA.pem``` from the previous step to the host managing the kubernetes cluster. Then we create a configmap as follows :
```
kubectl -n kube-system create configmap loxilb-cacert --from-file=`pwd`/loxilbCA.pem
```

To make kube-loxilb, use this root CA, we need to append the following to kube-loxilb.yaml before applying it :

```
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
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
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
        - --loxiURL=https://192.168.80.9:8091
        - --cidrPools=defaultPool=192.168.80.9/32
        volumeMounts:
        - mountPath: /etc/ssl/certs/loxilbCA.pem
          name: loxilb-cacert
          subPath: loxilbCA.pem
        securityContext:
          privileged: true
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
      volumes:
        - name: loxilb-cacert
          configMap:
            defaultMode: 420
            name: loxilb-cacert
```
 
Please note that here the loxiURL has changed to https and loxilb rootCA will be added to the pod system store of CA certs. If more than one root CA need to be added, we can concat them into a single file loxilbCA.pem. Additionally, we can mount them separately as loxilbCAx.pem, loxilbCAy.pem etc.
