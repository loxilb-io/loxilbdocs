### Deploy LoxiLB with multi-AZ HA support in AWS

LoxiLB supports stateful HA configuration in various cloud environments such as AWS. Especially for AWS, one can configure HA using the Floating IP [pattern](https://docs.aws.amazon.com/whitepapers/latest/real-time-communication-on-aws/floating-ip-pattern-for-ha-between-activestandby-stateful-servers.html), together with [LoxiLB](https://github.com/loxilb-io/loxilb).

The HA configuration described in the above [document](https://docs.aws.amazon.com/whitepapers/latest/real-time-communication-on-aws/floating-ip-pattern-for-ha-between-activestandby-stateful-servers.html) has certain limitations. It could only be configured within a single Availability-Zone(AZ). The HA instances need to share the VIP of the same subnet in order to provide a single access point to users, but this configuration was so far not possible in a multi-AZ environment. This blog explains how to deploy LoxiLB in a multi-AZ environment and configure HA.

### Overall Scenario

Two LoxiLB instances - loxilb1 and loxilb2 will be deployed in different AZs. These two loxilbs form a HA pair and operate in active-backup roles.

The active loxilb1 instance is additionally assigned a secondary network interface called loxi-eni. The loxi-eni network interface has a private IP (192.168.248.254 in this setup) which is used as a secondary IP.

loxilb1 associates this *192.168.248.254* secondary IP with an user-specified public ElasticIP address. When a user accesses the EKS service externally using an ElasticIP address, this traffic is NATed to the 192.168.248.254 IP and delivered to the active loxilb instance. The active loxilb instance can then load balance the traffic to the appropriate endpoint in EKS.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/40ce1e2a-06a0-4e80-9ce7-22491cd3b547)

If loxilb1 goes down due to any reason, the status of loxilb2, which was backup previously, changes to active.

During this transition, loxilb2 instance is assigned a new loxil-eni secondary network interface, and the 192.168.248.254 IP used by the the original master "loxilb1" is set to the secondary network interface of loxilb2.

The ElasticIP used by the user is also (re)associated to the 192.168.248.254 private IP address of the "new" active instance. This makes it possible to maintain active sessions even during failover or situations where there is a need to upgrade orginal master instance etc.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/c6b5e454-1830-4c67-b2f5-ccb7895fa93d)

To summarize, when a failover occurs the public ElasticIP address is always associated to the active LoxiLB instance, so users who were previously accessing EKS using the same ElasticIP address can continue to do so without being affected by any node failure or other issues.

### An example scenario

We will use eksctl to create an EKS cluster. To use eksctl, we need to register authentication information through AWS CLI. Instructions for installing aws CLI & eksctl etc are omitted in this document and can be found in AWS.

Using eksctl, let's create an EKS cluster with the following command. For this test, we are using  AWS's Osaka region, so using ```ap-northeast-3``` in the --region option.

```
eksctl create cluster \
  --version 1.24 \
  --name multi-az-eks \
  --vpc-nat-mode Single \
  --region ap-northeast-3 \
  --node-type t3.medium \
  --nodes 2 \
  --with-oidc \
  --managed
```
After running the above, we will have an EKS clsuter with two nodes named "multi-az-eks".

### Configuring LoxiLB EC2 Instances
####  Create LoxiLB subnet

After configuring EKS, it's time to configure the LoxiLB instance. Let's create the subnet that each of the LoxiLB instances will use.

LoxiLB instances will be created each located in a different AZ. Therefore, the subnets to be used by the instances will also be created in different AZs: AZ-a and AZ-b.

First, create a subnet loxilb-subnet-a in ap-northeast-3a with the subnet 192.168.218.0/24.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/33e24772-df26-4344-a970-6215cebf26d6)

Similarly, create a subnet loxilb-subnet-b in ap-northeast-3b with the subnet 192.168.228.0/24.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/997b7754-0c98-42f2-a877-ab89b40f6522)

After creating it, we can double check the "Enable auto-assign public IPv4 address" setting so that interfaces connected to each subnet are automatically assigned a public IP.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/da20de2a-8cc0-4d30-ae3d-e0ff6ca4dd0e)

#### AWS Route table

Newly created subnets automatically use the default route table. We will connect the default route table to the internet gateway so that users can access the LoxiLB instance from outside.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/6ad5fa46-63e6-4ea0-a974-81606a62a842)

#### LoxiLB IAM Settings

LoxiLB instances require permission to access the AWS EC2 API to associate ElasticIPs and create secondary interfaces and subnets.

We will create a role with the following IAM policy for LoxiLB EC2 instances.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

#### LoxiLB EC2 instance creation

We will create two LoxiLB instances for this example and connect the instances with subnets A and B created above.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/732c6b66-33b3-4927-8864-abf03377042b)

And specify to use the IAM role created above in the IAM instance profile of the Advanced details settings.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/bdbfa17d-69ea-4eba-9fec-ae6481f6c971)

After the instance is created, go to the Action → networking → Change Source /destination check menu in the instance menu and disable this check.  Since LoxiLB is a load balancer, this configration must be disabled for LoxiLB to operate properly.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/bd958cea-684e-431a-bf7e-d0d8c6b5ee14)

#### Create Elastic IP

Next we will create an Elastic IP to use to access the service from outside.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/43c43753-d30c-46a1-97d0-bfd92a76a293)

For this example, the IP ```13.208.x.x``` was assigned. The Elastic IP is used when deploying kube-loxilb, and is automatically associated to the LoxiLB master instance when configuring LoxiLB HA without any user intervention.

#### kube-loxilb deployment

kube-loxilb is a K8s operator for LoxiLB. Download the manifest file required for your deployment in EKS.

```
wget https://raw.githubusercontent.com/loxilb-io/kube-loxilb/main/manifest/ext-cluster/kube-loxilb.yaml
```

Change the args inside this yaml (as applicable)

```
spec:
  containers:
  - name: kube-loxilb
    image: ghcr.io/loxilb-io/kube-loxilb:latest
    imagePullPolicy: Always
    command:
    - /bin/kube-loxilb
    args:
    - --loxiURL=http://192.168.228.108:11111,http://192.168.218.60:11111
    - --externalCIDR=13.208.X.X/32
    - --privateCIDR=192.168.248.254/32
    - --setRoles=0.0.0.0
    - --setLBMode=2 
```

* Modify loxiURL with the IPs of the LoxiLB EC2instances created above.
* For externalCIDR, specify the Elastic IP created above.
* PrivateCIDR specifies the VIP that will be associated with the Elastic IP. As described in the scenario above, we will use 192.168.248.254 as the VIP in this article. The IP must be set within the range of the VPC CIDR and not currently part of any another subnet.

#### Run LoxiLB Pods
##### Install docker on LoxiLB instance(s)

LoxiLB is deployed as a container on each instance. After creating two EC2 instances with appropriate EMI (preferably Ubuntu 20.04 or 22.04). To use loxilb pods, docker must first be installed on the instance. Docker installation guide can be found [here](https://docs.docker.com/engine/install/ubuntu/)

##### Setup security groups to allow necessary traffic to loxilb instance

LoxiLB will by default use TCP ports 11111, 22222 for communication with each other and with kube-loxilb. So, at minimum these should be allowed as part of the inbound security rules associated with each instance. For simplicity, in this example we have allowed all ports from any source as follows:

![Screenshot 2024-09-28 at 9 32 36 AM](https://github.com/user-attachments/assets/63a27d2f-042d-4057-a240-3aa130f58622)

#### Running LoxiLB container

The following command is for a LoxiLB instance (loxilb1) using subnet-a. Similar command can be followed for loxilb2 instance.

```
sudo docker run -u root --cap-add SYS_ADMIN \
  --restart unless-stopped \
  --net=host \
  --privileged \
  -dit \
  -v /dev/log:/dev/log -e AWS_REGION=ap-northeast-3 \
  --name loxilb \
  ghcr.io/loxilb-io/loxilb:aws-support \
  --cloud=aws --cloudcidrblock=192.168.248.0/24 --cluster=192.168.228.108 --self=0
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
root@ip-192-168-228-108:/#
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
We can now access the service from a host client :

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/8ea03fe2-5c28-4f04-8600-9c407278debf)

#### Testing HA functionality

Once LoxiLB HA is configured, we can check in the AWS console that a secondary interface has been added to the master. To test HA operation, simply stop the LoxiLB pod in master state.
```
ubuntu@ip-192-168-228-108:~$ sudo docker stop loxilb
loxilb
ubuntu@ip-192-168-228-108:~$
```

Even after stopping the masterLB, the service can be accessed without interruption :

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/bcc28acd-8dc7-4a00-a884-95c572714b50)

During failover, a secondary interface is created on the *new* master instance, and you can see that the ElasticIP is also associated to the new interface.

![image](https://github.com/loxilb-io/loxilbdocs/assets/111065900/7a5c10ba-6456-49d5-8756-c6e630b3e639)

