## Lab1: Create an EKS cluster through eksctl (version 1.31 @ 2025.01.03)
#### !!!Tips, all commands here are tested on AMZN Linux 2023!!!


## 1.Install needed tools on workstation: awscli v2, kubectl, eksctl, aws-iam-authenticator, kubectx and kubens
#### 1) Inastall and configure awscli v2 (aws-cli/2.22.26)
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip

[new install]
sudo ./aws/install

[update install] If found preexisting AWS CLI installation, please execute following command to install
sudo ./aws/install --update


complete -C '/usr/local/bin/aws_completer' aws; echo "complete -C '/usr/local/bin/aws_completer' aws" >> ~/.bashrc
source ~/.bash_profile
```
Check awscli version, 
```
aws --version
```
If the version of awscli is older(v1), you can follow the official link below to update awscli v2:  
[https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-upgrade](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-upgrade)  

configure credential file:
```
[ec2-user@ip-172-31-1-111 ~]$ aws configure
AWS Access Key ID [None]:  Input your AK 
AWS Secret Access Key [None]:  Input your SK
Default region name [cn-northwest-1]: By default 宁夏 region
Default output format: json   
```
Official link for configure aws cli:  
[https://docs.amazonaws.cn/en_us/cli/latest/userguide/cli-configure-files.html](https://docs.amazonaws.cn/en_us/cli/latest/userguide/cli-configure-files.html)

#### 2)Install kubectl(Kubernetes 1.31)
```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin

source <(kubectl completion bash); echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bash_profile  
```

Check kubectl version, 
```
kubectl version
```
Official link for install kubectl(aws):  
[https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-kubectl.html](https://docs.amazonaws.cn/en_us/eks/latest/userguide/install-kubectl.html)  
Official link for install kubectl(Kubernetes):  
[https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

#### 3)Install eksctl
```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -LO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
#curl -OL https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar xvf eksctl_*_amd64.tar.gz
sudo mv ./eksctl /usr/local/bin

source <(eksctl completion bash); echo "source <(eksctl completion bash)" >> ~/.bashrc
source ~/.bash_profile
```
Check eksctl version, 
```
eksctl version
```
Official link for install eksctl:  
[https://docs.amazonaws.cn/en_us/eks/latest/userguide/eksctl.html](https://docs.amazonaws.cn/en_us/eks/latest/userguide/eksctl.html)  
[https://eksctl.io/installation/](https://eksctl.io/installation/)


####  4)Install Helm(Kubernetes 1.31)
``` 
curl -OL https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz  
tar xvf helm-*-linux-amd64.tar.gz

sudo mv ./linux-amd64/helm /usr/local/bin



curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy_cn.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_cn.json

eksctl create iamserviceaccount \
  --cluster=eksgo202 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole202 \
  --attach-policy-arn=arn:aws-cn:iam::659702723394:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve



helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eksgo202 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

``` 
Official link for install Helm(Helm):   
[https://github.com/helm/helm/releases/](https://github.com/helm/helm/releases/)


####  5)[optional] Install jq
``` 
sudo yum install -y jq
``` 

####  6)Create an SSH key pair,replace your-key-name to what you like, such as: jerrykey 
``` 
aws ec2 create-key-pair --key-name your-key-name --query 'KeyMaterial' --output text > your-key-name.pem  
chmod 400 your-key-name.pem
``` 
Example:
``` 
[ec2-user@ip-172-31-1-73 ~]$ aws ec2 create-key-pair --key-name jerrykey --query 'KeyMaterial' --output text > jerrykey.pem  
[ec2-user@ip-172-31-1-73 ~]$ ls -la | grep jerrykey
-rw-rw-r--  1 ec2-user ec2-user      1679 Nov 25 14:20 jerrykey.pem
[ec2-user@ip-172-31-1-73 ~]$ chmod 400 jerrykey.pem 
[ec2-user@ip-172-31-1-73 ~]$ ls -la | grep jerrykey
-r--------  1 ec2-user ec2-user      1679 Nov 25 14:20 jerrykey.pem
```  

##  2. Create EKS cluster on AWS China region through eksctl, we choose Ningxia region here. You have three ways to create the EKS cluster, one is single command line, the others are with yaml file 
#### 1) one is single command line
```
# 环境变量
# AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
# CLUSTER_NAME 集群名称
# 替换ekswork-yourname为适合值，比如eksworkjerry
export CLUSTER_NAME=ekswork-yourname
# 替换ng-ekswork-yourname为适合值，比如ng-eksworkjerry
export NODEGROUP_NAME=ng-ekswork-yourname
export INSTANCE_TYPE=t3.medium
# 替换your-key为你的key的名称
export AWS_KEY=your-key-name
# 替换1.19为你需要的版本
export VERSION=1.31
```
Create eks cluster through follow single command line
```
eksctl create cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --nodegroup-name ${NODEGROUP_NAME} --node-type ${INSTANCE_TYPE} --nodes 3 --nodes-min 1 --nodes-max 4 --ssh-access --ssh-public-key ${AWS_KEY} --managed --version ${VERSION}
```

#### 2) Prepare EKS cluster yaml file - eksworkjerry.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eksworkjerry
  region: cn-northwest-1
  version: "1.31"

vpc:
  cidr: 10.192.0.0/16
  clusterEndpoints:
    privateAccess: true
    publicAccess: true

addons:
  - name: eks-pod-identity-agent

iam:
  withOIDC: true
  podIdentityAssociations:
    - serviceAccountName: my-app-sa
      namespace: my-app-namespace
      roleARN: arn:aws-cn:iam::xxxxxxxxxxxx:role/your-role-name
      createServiceAccount: true

managedNodeGroups:
  - name: default
    desiredCapacity: 3
    minSize: 1
    maxSize: 6
    instanceType: m6i.large
    volumeSize: 80
    volumeType: gp3
    ssh:
      publicKeyName: jerrykey
      allow: true
    iam:
      withAddonPolicies:
        ebs: true
        fsx: true
        efs: true
```

Create eks cluster through follow single command line
```
eksctl create cluster -f eksworkjerry.yaml
```

#### 3) Prepare EKS cluster yaml file for existing VPC with private network only - eksworkjerryvpc.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkjerryvpc
  region: cn-northwest-1
  version: "1.31"

vpc:
#  This vpc is with 3 subnets, nat gateway is within 1 public subnet 
#  Those private subnets are with route table which contains route entry to nat gateway
#  id: "vpc-0c445fc9cb3c357c8"  # (optional, must match VPC ID used for each subnet below)
#  cidr: "10.62.0.0/16"       # (optional, must match CIDR used by the given VPC)
  subnets:
    # must provide 'private' and/or 'public' subnets by availibility zone as shown
    private:
      cn-northwest-1a:
        id: "subnet-0dbdffd43d569a3a1"
#        cidr: "10.62.2.0/24" # (optional, must match CIDR used by the given subnet)

      cn-northwest-1b:
        id: "subnet-0e2355b831c9b187e"
#        cidr: "10.62.3.0/24"  # (optional, must match CIDR used by the given subnet)

managedNodeGroups:
  - name: ng-eksworkjerryvpc-01
    instanceType: t3.medium
    instanceName: ng-eksworkjerryvpc
    desiredCapacity: 3
    minSize: 1
    maxSize: 4 
    volumeSize: 100
    privateNetworking: true # if only 'Private' subnets are given, this must be enabled
    ssh:
      publicKeyName: jerrykey
      allow: true
      enableSsm: true 
```

Create eks cluster through follow single command line
```
eksctl create cluster -f eksworkjerryvpc.yaml
```


#### 4) There are several yaml examples for creating an EKS cluster under different scenarios
[https://github.com/eksctl-io/eksctl/tree/main/examples](https://github.com/eksctl-io/eksctl/tree/main/examples)