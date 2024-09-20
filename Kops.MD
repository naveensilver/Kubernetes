#Production Grade kubernetes Cluster deployment with kops 

Management/Bastian Server = 1 (We do everything in Management server)
Master nodes = 1
Slaves nodes = 3 


# DNS NAME (naveenops.live)

>> Once After Purchasing the domain name, Go to Route 53 create hosted zone 
>> Go to newly created hosted zone (naveenops.live) you will see nameServers. start with ns-**** like this.
>> Configure nameServers with your domain in GoDADDy.

# Create s3 bucket to store Kops state
>> naveenops.live

# Create IAM User and Role

>> User - set permissions as FullAdminAccess (Not recommended)
>> Role - Aws service type as Ec2 - AdministratorAccess (Not recormended)

Recommended: Give individual access 


# Create one Bastian Server (Management server) EC2 instance t2.medium 

>> Assign IAM role to EC2 Instance (Bastian Server)
>> Select instance > Actions > security > Modify iam role > Choose iam role 

# Now Connect to Ec2 instance and Generate ssh keys (These keys used by kops, so kops will apply these keys to all the nodes)

    sudo -i 
    ssh-keygen 
    cd .ssh/
    ls -al
    cd 

# Now Download kops and Kubectl to usr/local/bin and change permissions

>> cd /usr/local/bin/

# Install Kops  (Kops github)

    wget https://github.com/kubernetes/kops/releases/download/v1.29.2/kops-linux-amd64 
    mv kops-linux-amd64 kops
    chmod 777 kops 


# Install Kubectl and varify 

    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod 777 kubectl 
    kubectl version 


# edit .bashrc and add all the env variables 

export NAME=naveenops.live
export KOPS_STATE_STORE=s3://naveenops.live
export AWS_REGION=us-east-1
export CLUSTER_NAME=naveenops.live
export EDITOR='/usr/bin/nano'
alias k=kubectl
#export K8S_VERSION=1.6.4

# After copying the above files to .bashrc run “ source .bashrc ”

# Create a Cluster using Kops and generate a cluster file and save it carefully and do neccessary changes

kops create cluster --name=naveenops.live \
--state=s3://naveenops.live --zones=us-east-1a,us-east-1b \
--node-count=2 --control-plane-count=1 --node-size=t3.medium --control-plane-size=t3.medium \
--control-plane-zones=us-east-1a --control-plane-volume-size 10 --node-volume-size 10 \
--ssh-public-key ~/.ssh/id_rsa.pub \
--dns-zone=naveenops.live --dry-run --output yaml  


# 6 Node cluster 
```
# kops create cluster --name=naveenops.live \
# --state=s3://naveenops.live --zones=us-east-1a,us-east-1b,us-east-1c \
# --node-count=3 --master-count=3 --node-size=t3.medium --master-size=t3.medium \
# --master-zones=us-east-1a,us-east-1b,us-east-1c --master-volume-size 10 --node-volume-size 10 \
# --ssh-public-key ~/.ssh/id_rsa.pub \
# --dns-zone=naveenops.live --dry-run --output yaml
```
# After dry run, you will see YAML file.. thats is our k8s cluster yaml file. Now copy entire yaml Script 

# Now create $ vi cluster.yaml and paste the k8s cluster yaml script, you can change the subnets if you want !!

# Once done run below commands to create the cluster 

kops create -f cluster.yml
kops update cluster --name naveenops.live --yes --admin
kops validate cluster --wait 10m
kops delete -f cluster.yml  --yes

# When you see cluster is ready.  Try below commands 

kubectl get pods 

kubectl get nodes 

kubectl get nodes -o wide

kubectl cluster-info 

kubectl get ns # 4 default namespaces

kubectl get pods -n kube-system 

kubectl get pods -n kube-system -o wide | grep -i api 
# api server is running and see IP addr and each and every component in the control plane running on api ip addr

kubectl get pods -n kube-system -o wide | grep -i etcd 
kubectl get pods -n kube-system -o wide | grep -i control
kubectl get pods -n kube-system -o wide | grep -i scheduler 


These all components IP addr same as API ip.


# If you want to deploy any resource these are two ways to deploy 

1. Imperative 
- using cli commands 

$ kubectl run testpod1 --image nginx:latest --dry-run -o yaml 
$ kubectl run testpod1 --image nginx:latest

2. Declarative 
- using yml or json $ vi testpod2.yml 

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: testpod2
  name: testpod2
spec:
  containers:
  - image: nginx:latest
    name: testpod2
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
$ kubectl apply -f testpod2.yml 

$ kubectl get pods -o wide

# Another way 

$ echo '
> apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: testpod3
  name: testpod3
spec:
  containers:
  - image: nginx:latest
    name: testpod3
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}'  | kubectl apply -f - 

$ kubectl get pods -o wide

____________
