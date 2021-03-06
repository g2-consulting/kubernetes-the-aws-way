# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across multiple AZs (availability zones) [azs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).

> Ensure that the AWS Cli is correctly configured [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone).

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Amazon Virtual Private Cloud

In this section a dedicated [Amaxon Virtual Private Cloud](https://aws.amazon.com/documentation/vpc/) (VPC) network will be setup to host the Kubernetes cluster.

Create the stack `kubernetes-the-aws-way-vpc` custom VPC network:

```
aws cloudformation create-stack --stack-name kubernetes-the-aws-way-vpc --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates-releases-eu-west-1/stable/vpc/vpc-2azs.yaml
```

The output will be similar to this:
```
{
    "StackId": "arn:aws:cloudformation:us-east-1:853412474297:stack/kubernetes-the-aws-way-vpc/cfcee3a0-6533-11e8-93ce-500c28659c35"
}
``` 

This [cloudformation](https://aws.amazon.com/cloudformation/?nc1=h_ls) stack will create the network architecture as described [here](http://templates.cloudonaut.io/en/stable/vpc/) in details.

Architecture: 
![Details](http://templates.cloudonaut.io/en/stable/img/vpc-2azs.png)

> The `10.0.0.0/16` IP address range can host up to 65534 instances.

### Alerting Stack

As part of the compute architecture used in this tutorial, will be necessary to create [SNS](https://aws.amazon.com/sns/) stack with topics to be used by some alarms associated with the EC2 instaces:

```
aws cloudformation create-stack --stack-name kubernetes-the-aws-way-alerting --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates-releases-eu-west-1/stable/operations/alert.yaml
```
Output similar to:
```
{
    "StackId": "arn:aws:cloudformation:us-east-1:853412474297:stack/kubernetes-the-aws-way-alerting/35fac140-6538-11e8-b369-500c289032fe"
}
```

### EC2 Vault Stack
Before creating the kubernetes componentes, we will setup a EC2 instance with Vault, that will give support to certifications creation e maintenance. These certificates will be used to establish secure communications between kubernetes architectural components.

To setup the Vault stack
```
aws cloudformation create-stack --stack-name kubernetes-the-aws-way-vault --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates-releases-eu-west-1/stable/ec2/ec2-auto-recovery.yaml \
--parameters ParameterKey=ParentVPCStack,ParameterValue=kubernetes-the-aws-way-vpc \
ParameterKey=ParentAlertStack,ParameterValue=kubernetes-the-aws-way-alerting \
ParameterKey=Name,ParameterValue=vault-server \
ParameterKey=IAMUserSSHAccess,ParameterValue=true \
ParameterKey=IngressTcpPort1,ParameterValue=22 \
ParameterKey=IngressTcpPort2,ParameterValue=8200 \
--capabilities CAPABILITY_IAM
```

Output:
```
{
    "StackId": "arn:aws:cloudformation:us-east-1:853412474297:stack/kubernetes-the-aws-way-vault/58ac1ab0-653b-11e8-a038-500c2854e035"
}
```

### Upload a public key to specific IAM user

The following steps are required to a enable some IAM user to login to the Vault EC2 instance using ssh. Replace the my-user used in the command below for some IAM user. 

Generate a PEM private key
```
openssl genrsa -out private.pem 2048
``` 

Extract a public key in PEM format
```
openssl rsa -in private.pem -pubout -outform PEM -out public.pem
```

Finally, upload your key to a specific IAM user
```
export SSH_KEY=`cat public.pem`
aws iam upload-ssh-public-key --user-name k8s --ssh-public-key-body $SSH_KEY
```

The output is similar to this
```
{
    "SSHPublicKey": {
        "UserName": "my-user",
        "SSHPublicKeyId": "APKAJU65L4EGHKK2H6MA",
        "Fingerprint": "81:60:cf:f5:3b:db:58:be:3e:41:c5:a8:b0:0f:29:df",
        "SSHPublicKeyBody": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtOwqtQah8o4JWsBLkt5r\nkhgfKoTzbusu5wpsJuTmSi+V8EO+i9iP81L3NfGcc8aH6+mYC55laIOE1LrE8E1e\notTBAYFXGddDJVSij4swnXtba48YRIPCvpNzvSKeYYzSTGi8UFi//xTPGokLIShf\nwqoBu9nuuLT4sOYjP3/HZzqXFsOYvBaIs06QtI78XjZQFTfU66OJy9w8cFd2hCLl\nakmxig5UMR683gXay4iwtJvdjOGslVBbaKCPIfSkSqtvfZU17n8OZXy6PgtTmg75\nzpFMWvl+80URQDWA73S3b7x6y0LESmpfxmrgLTQnKt4UsYa9EagTbOxenK2teIM4\ncwIDAQAB\n-----END PUBLIC KEY-----",
        "Status": "Active",
        "UploadDate": "2018-06-02T16:18:46.726Z"
    }
}
```
### Connecting to the Vault instance
```
export IP_ADDRESS=$(aws cloudformation --region us-east-1 describe-stacks --stack-name kubernetes-the-aws-way-vault --query 'Stacks[0].Outputs[?OutputKey==`IPAddress`].OutputValue' --output text)

chmod 400 private.pem
ssh -i private.pem ec2-user@$IP_ADDRESS

```
After you update your ssh key to IAM, you should wait for about 10 minutes before trying to login. The vault instance updates its credentials every 10 minutes.

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> output

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as describe in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1006-gcp x86_64)

...

Last login: Sun May 13 14:34:27 2018 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)



