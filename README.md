# Multi-architecture workshop with Graviton and x86 architectures
Graviton multi-arch workshop consists of two modules below. We recommend completing the workshop with both modules in same sequence as some steps like cloning the workshop directory are executed in Module 1 and not repeated in Module 2 instructions to avoid duplication of steps.

### Know your pre-deployed workshop environment (Prerequisites)
The workshop account is pre-deployed with following:

•	Region: us-west-2

•	Cloud9 environment

•	A VPC with 6 subnets; 3 public and 3 private subnets

•	Application Load Balancer (ALB) with its own security group

•	Target Group and an ALB listener

•	2 Launch Templates with Graviton and x86 compatible Amazon Machine Images (AMIs)

### Go to Cloud9 IDE

![image](https://user-images.githubusercontent.com/75417152/163193042-41ca1705-a8d8-48ac-995e-4ae8fe43339b.png)

### Update IAM Settings for your workspace
Note: Cloud9 normally manages IAM credentials dynamically. This isn’t currently compatible with the EKS IAM authentication, so we will disable it and rely on the IAM role instead.

### Return to your workspace and click the sprocket, or launch a new tab to open the Preferences tab -> Select AWS SETTINGS -> Turn off AWS managed temporary credentials -> Close the Preferences tab

<img width="1434" alt="Cloud9preferences" src="https://user-images.githubusercontent.com/75417152/164973198-049fd685-5e70-40f5-810a-df1a9560aa4d.png">

### Validate IAM role
Use the GetCallerIdentity CLI command to validate that the Cloud9 IDE is using the correct IAM role.
```bash
aws sts get-caller-identity
```
The output assumed-role ARN should contain TeamRole and Instance ID like below:
```bash
"Arn": "arn:aws:sts::{ACCOUNT_ID}:assumed-role/TeamRole/{Instance_ID}"
```


### Let's now clone workshop repo to Cloud9

```bash
git clone https://github.com/ashwinikumar-sa/graviton-multiarch-workshop.git
cd graviton-multiarch-workshop 
```
## Module-1: Deploy and run a multi-arch application on a mixed-arch Auto Scaling group (with x86 and Graviton instances) 
In this module of the workshop, you will deploy a mixed architecture Auto Scaling group with x86 and Graviton instances. You will be deploying a sample node.js application with node.js dependencies with user data script by modifying launch templates.

```bash
export AWS_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)
export stack_name=$(aws cloudformation describe-stacks --query 'Stacks[3].StackName' --output text)

# load outputs to env vars
for output in $(aws cloudformation describe-stacks --stack-name $stack_name --query 'Stacks[].Outputs[].OutputKey' --output text)
do
    export $output=$(aws cloudformation describe-stacks --stack-name $stack_name --query 'Stacks[].Outputs[?OutputKey==`'$output'`].OutputValue' --output text)
    eval "echo $output : \"\$$output\""
done
export tg_arn=$(aws elbv2 describe-target-groups --names $stack_name --query TargetGroups[].TargetGroupArn --output text)
```


### Step 2: Create Auto Scaling group
```bash
sed -i.bak -e "s#%TargetGroupARN%#$tg_arn#g" -e "s/%publicSubnet1%/$publicSubnet1/g" -e "s/%publicSubnet2%/$publicSubnet2/g" -e "s/%publicSubnet3%/$publicSubnet3/g" asg-config-multiarch.json
```

```bash
aws autoscaling create-auto-scaling-group --cli-input-json file://asg-config-multiarch.json
```

Go to AWS console and see that ASG "asg-mixed-arch" is created. There should be x86 (3 instances) and Graviton (3 instances) launched by ASG with different launch templates:
![image](https://user-images.githubusercontent.com/75417152/163199185-574f656f-3171-4c09-8810-134015836bf2.png)

Please feel free to explore Launch Templates in Console:
![image](https://user-images.githubusercontent.com/75417152/163200439-04615c20-e5d2-4cba-8795-361e54fd895e.png)

### Step 3: Let's explore a sample node.js app
```bash
cat app.js
```
Output:
```bash
const http = require('http');

const port = 80;

const server = http.createServer((req, res) => {
          res.statusCode = 200;
          res.setHeader('Content-Type', 'text/plain');
          res.end(`Hello World. This processor architecture is ${process.arch}`);
});

server.listen(port, () => {
          console.log(`Server running on processor architecture ${process.arch}`);
});
```

### Step 4: Let's modify Launch Templates with user data to install our app
### Step 4.1: Modify x86 Launch Template (create new version)

![image](https://user-images.githubusercontent.com/75417152/163204637-e7d24ab0-44a1-450b-95e3-ed81a7f4e88c.png)

### User data
```bash
#!/bin/bash

yum update -y
yum install git -y
VERSION=v14.15.3
DISTRO=linux-x64
wget https://nodejs.org/dist/$VERSION/node-$VERSION-$DISTRO.tar.xz
mkdir -p /usr/local/lib/nodejs
sudo tar -xJvf node-$VERSION-$DISTRO.tar.xz -C /usr/local/lib/nodejs 
export PATH=/usr/local/lib/nodejs/node-$VERSION-$DISTRO/bin:$PATH
git clone https://github.com/ashwinikumar-sa/graviton-multiarch-workshop
cd graviton-multiarch-workshop
node app.js
```
### Step 4.2: Set latest version as Default version
![image](https://user-images.githubusercontent.com/75417152/163228623-26dddbbd-9a65-47e1-8650-bd613ee9fe64.png)
![image](https://user-images.githubusercontent.com/75417152/163228839-0f667d2e-8cba-427d-ac79-3e6a3bd31316.png)
![image](https://user-images.githubusercontent.com/75417152/163230448-21c9e714-deaa-49a3-aaa8-478579b72b4e.png)


### Step 4.3: Modify Graviton Launch Template (create new version)

![image](https://user-images.githubusercontent.com/75417152/163206210-be0dbd6c-50aa-496c-941f-b0f6e385f72a.png)

### User data
```bash
#!/bin/bash

yum update -y
yum install git -y
VERSION=v14.15.3
DISTRO=linux-arm64
wget https://nodejs.org/dist/$VERSION/node-$VERSION-$DISTRO.tar.xz
mkdir -p /usr/local/lib/nodejs
sudo tar -xJvf node-$VERSION-$DISTRO.tar.xz -C /usr/local/lib/nodejs 
export PATH=/usr/local/lib/nodejs/node-$VERSION-$DISTRO/bin:$PATH
git clone https://github.com/ashwinikumar-sa/graviton-multiarch-workshop
cd graviton-multiarch-workshop
node app.js
```
### Step 4.4: Set latest version to default version for this Launch Template as well (same as Step 4.2)
![image](https://user-images.githubusercontent.com/75417152/163230229-97165501-ac7d-4cd6-88ad-cf9a9bd32acb.png)



### Step 5: Refresh the instances in ASG to use modified version of the launch template and install node.js app
```bash
aws autoscaling start-instance-refresh \
--auto-scaling-group-name asg-mixed-arch \
--preferences '{"InstanceWarmup": 0, "MinHealthyPercentage": 0}'
```

### Step 6: check that instances are healthy in Target group

![image](https://user-images.githubusercontent.com/75417152/163221701-2fa7210e-16b9-422c-b714-84cd7fadcef0.png)

![image](https://user-images.githubusercontent.com/75417152/163221811-76952a67-447e-4697-8aff-749e48e89a60.png)


### Step 7: Let's see how app is running on mixed instances (x86 and Graviton) behind load balancer
![image](https://user-images.githubusercontent.com/75417152/163224071-fd578fcd-830c-4447-90fa-146f73b597bc.png)

![image](https://user-images.githubusercontent.com/75417152/163224196-6ef45418-1ced-4cc3-95da-11ea154878ae.png)

![image](https://user-images.githubusercontent.com/75417152/163224609-98e6a229-4165-43b7-b317-75fecde21fbb.png)

![image](https://user-images.githubusercontent.com/75417152/163224694-4bb80f08-a024-4978-96cc-61c4e70bd026.png)

=======================================================================================

## Module-2: Build, deploy and run multi-arch containers on a multi-arch Amazon EKS cluster (with x86 and Graviton instances)
In this module of the workshop, you will be creating EKS managed node groups with Graviton and x86 instances in an Amazon EKS cluster. Then, you will build a multi-arch container image of a sample node.js application for x86_64 and arm64 target architectures and store it on Amazon ECR with a single manifest list. Finally,  you will be deploying pods with multi-arch container image on x86 and Graviton based nodes on EKS cluster

### Step 1: Check the Amazon EKS cluster
```bash
eksctl get cluster
kubectl get nodes
```
You should see two nodes running in the EKS cluster. Let's now install some Kubernetes tools in the EKS cluster

### Step 2: Install Helm CLI
Please follow instructions below:

https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks/030_k8s_tools/helm_deploy.html

### Step 3: Install KUBE-OPS-VIEW
https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks/030_k8s_tools/install_kube_ops_view.html

### Step 4: Create two Managed Node groups with Graviton and x86 based instances in Amazon EKS cluster
To create a Graviton based nodegroup, kube-proxy, coredns and aws-node addons should be up to date. Please use below eksctl commands to update them.

```bash
eksctl utils update-coredns --cluster eksworkshop-eksctl --approve 

eksctl utils update-kube-proxy --cluster eksworkshop-eksctl --approve 

eksctl utils update-aws-node --cluster eksworkshop-eksctl --approve
```

Now, explore node group configuration files:
### add-mng-gv2.yaml

```bash
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

managedNodeGroups:
- name: arm64-mng-od-4vcpu-16gb
  desiredCapacity: 1
  minSize: 0
  maxSize: 2
  instanceType: m6g.xlarge
  iam:
    withAddonPolicies:
      autoScaler: true
      cloudWatch: true
      albIngress: true
  privateNetworking: true
  labels:
    alpha.eksctl.io/cluster-name: eksworkshop-eksctl
    alpha.eksctl.io/nodegroup-name: arm64-mng-od-4vcpu-16gb
    intent: mng-multiarch-apps
  tags:
    alpha.eksctl.io/nodegroup-name: arm64-mng-od-4vcpu-16gb
    alpha.eksctl.io/nodegroup-type: managed
    k8s.io/cluster-autoscaler/node-template/label/intent: mng-multiarch-apps

metadata:
  name: eksworkshop-eksctl
  region: us-west-2
  version: "1.21"
```
### add-mng-x86.yaml

```bash
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

managedNodeGroups:
- name: x86-mng-od-4vcpu-16gb
  desiredCapacity: 1
  minSize: 0
  maxSize: 2
  instanceType: m5.xlarge
  iam:
    withAddonPolicies:
      autoScaler: true
      cloudWatch: true
      albIngress: true
  privateNetworking: true
  labels:
    alpha.eksctl.io/cluster-name: eksworkshop-eksctl
    alpha.eksctl.io/nodegroup-name: x86-mng-od-4vcpu-16gb
    intent: mng-multiarch-apps
  tags:
    alpha.eksctl.io/nodegroup-name: x86-mng-od-4vcpu-16gb
    alpha.eksctl.io/nodegroup-type: managed
    k8s.io/cluster-autoscaler/node-template/label/intent: mng-multiarch-apps

metadata:
  name: eksworkshop-eksctl
  region: us-west-2
  version: "1.21"
```
### Create Graviton instances based EKS managed node group
```bash
eksctl create nodegroup --config-file=add-mng-gv2.yaml
```
### Create x86 instances based EKS managed node group
```bash
eksctl create nodegroup --config-file=add-mng-x86.yaml
```
