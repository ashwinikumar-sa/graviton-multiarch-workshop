# Multi-architecture workshop with Graviton and x86 architectures
Graviton multi-arch workshop consists of two modules.
* [Module 1-Mixed-architecture Auto Scaling group for running a multi-arch application](#module-1-deploy-and-run-a-multi-arch-application-on-a-mixed-arch-auto-scaling-group-with-x86-and-graviton-instances)
* [Module 2-Mixed-architecture Amazon EKS cluster for running multi-arch container](#module-2-build-deploy-and-run-multi-arch-containers-on-a-multi-arch-amazon-eks-cluster-with-x86-and-graviton-instances)

#### For smooth operation of workshop, it is recommended to follow both modules 1 and 2 in the same sequence as some prerequisites have been completed in module 1 for module 2.

# How to access the workshop in an AWS event
* Please click on this event URL (Right click and open in new tab) - https://dashboard.eventengine.run/login
* Enter the event hash (12 or 16 digit event hash to be provided by us). All the participants to use the same Hash code and click Accept Terms and Login. 
```diff
- Please do not share the Hash code outside the event as we have limited AWS accounts created for this event
```

![image](https://user-images.githubusercontent.com/75417152/165553329-55c2e644-41dc-4159-8d78-01b0103faba0.png)

* Now on the next page, select – “Email One-Time Password (OTP)”

![image](https://user-images.githubusercontent.com/75417152/165553745-dfaed528-9d80-41ed-90bd-8271778f3ef9.png)

* Enter your official or personal email id (email account you can easily access) and click Send passcode. 
```diff
- Note: Please use same email id to get OTP to access your workshop account if logged out and need to access again 
- (Email account gets mapped to your AWS account)
```

![image](https://user-images.githubusercontent.com/75417152/165554098-5f6ab730-fd00-407f-9171-d89d47dc882d.png)

* Grab your On-time passcode from your email (please also check spam folder for it if not found in inbox) and use this to sign-in. You will be taken to an Event Dashboard as below. Please click on “Set Team Name” and enter your name (this is recommended to enter your name to map it with your account so that we can help you troubleshoot in case of any challenges during workshop). Click on "AWS Console" below:

![image](https://user-images.githubusercontent.com/75417152/165557086-9abfcfb3-3666-4fca-ba16-3b162359af45.png)

* You will be taken to the following screen. Please click on “Open AWS Console”:

![image](https://user-images.githubusercontent.com/75417152/165557558-deecccae-2122-4617-b01f-ac0e10d0437c.png)

* You will be taken to the AWS Console

![image](https://user-images.githubusercontent.com/75417152/165558078-b33344e5-789e-468d-911a-28af3d11afa0.png)


# Contents
* [Know your pre-deployed workshop environment](#know-your-pre-deployed-workshop-environment-prerequisites)
* [Go to Cloud9 IDE](#go-to-cloud9-ide)
* [Update IAM settings of your workspace](#update-iam-settings-for-your-workspace)
* [Validate IAM Role](#validate-iam-role)
* [Resize Cloud9 instance](#resize-cloud9-instance-root-volume-to-100-gib)
* [Clone workshop repo on Cloud9](#lets-now-clone-workshop-repo-to-cloud9)
* [Module 1-Mixed-architecture Auto Scaling group for running a multi-arch application](#module-1-deploy-and-run-a-multi-arch-application-on-a-mixed-arch-auto-scaling-group-with-x86-and-graviton-instances)


### Know your pre-deployed workshop environment (Prerequisites)
The workshop account is pre-deployed with following components:

* Region: us-west-2

* Cloud9 environment pre-installed with utility tools like AWS CLI, kubectl, eksctl etc.

* A VPC with 6 subnets; 3 public and 3 private subnets

* Application Load Balancer (ALB) with its own security group

* Target Group and an ALB listener

* 2 Launch Templates with Graviton and x86 compatible Amazon Machine Images (AMIs)

* An EKS cluster with 2 management nodes (for installing monitoring tools etc)

In CloudFormation console, you should be able to see four (4) CloudFormation stacks created in your account as shown below.

#### Note: If any of the below stacks is missing, please ask AWS support team for help in troubleshooting or giving you access to another account.

![image](https://user-images.githubusercontent.com/75417152/167646245-85caa265-a171-4814-9690-c802304f4249.png)


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

### Resize Cloud9 Instance Root Volume to 100 GiB
The default 10GB may not be enough to build your application docker images. Thus, let us resize the EBS volume used by the Cloud9 instance using below link:

https://ec2spotworkshops.com/ecs-spot-capacity-providers/workshopsetup/resize_ebs.html

### Let's now clone workshop repo to Cloud9

```bash
git clone https://github.com/ashwinikumar-sa/graviton-multiarch-workshop.git
cd graviton-multiarch-workshop 
```
### Let's store CloudFormation stack outputs to some environment variables for using throughout workshop:

```bash
export AWS_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)
export stack_name=$(aws cloudformation list-stacks | jq -r '.StackSummaries[] | select(.StackName|test("^mod.")) | .StackName')

# load outputs to env vars
for output in $(aws cloudformation describe-stacks --stack-name $stack_name --query 'Stacks[].Outputs[].OutputKey' --output text)
do
    export $output=$(aws cloudformation describe-stacks --stack-name $stack_name --query 'Stacks[].Outputs[?OutputKey==`'$output'`].OutputValue' --output text)
    eval "echo $output : \"\$$output\""
done
export tg_arn=$(aws elbv2 describe-target-groups --names $stack_name --query TargetGroups[].TargetGroupArn --output text)
```

## Module-1: Deploy and run a multi-arch application on a mixed-arch Auto Scaling group (with x86 and Graviton instances) 
In this module of the workshop, you will deploy a mixed architecture Auto Scaling group with x86 and Graviton instances. You will be deploying a sample node.js application with node.js dependencies with user data script by modifying launch templates.
![image](https://user-images.githubusercontent.com/75417152/164983418-05f26d68-46d7-4450-a318-6c88cb390aac.png)


### Step 1: Create Auto Scaling group
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

### Step 2: Let's explore a sample node.js app
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

### Step 3: Let's modify Launch Templates with user data to install our app
Once you click on modify template, scroll to the bottom of the page and expand the Advanced Tab to find the user-data section and populate with below user-data:

### Step 3.1: Modify x86 Launch Template (create new version)

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
### Step 3.2: Set latest version as Default version
![image](https://user-images.githubusercontent.com/75417152/163228623-26dddbbd-9a65-47e1-8650-bd613ee9fe64.png)
![image](https://user-images.githubusercontent.com/75417152/163228839-0f667d2e-8cba-427d-ac79-3e6a3bd31316.png)
![image](https://user-images.githubusercontent.com/75417152/163230448-21c9e714-deaa-49a3-aaa8-478579b72b4e.png)


### Step 3.3: Modify Graviton Launch Template (create new version)

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
### Step 3.4: Set latest version to default version for this Launch Template as well (same as Step 4.2)
![image](https://user-images.githubusercontent.com/75417152/163230229-97165501-ac7d-4cd6-88ad-cf9a9bd32acb.png)



### Step 4: Refresh the instances in ASG to use modified version of the launch template and install node.js app
```bash
aws autoscaling start-instance-refresh \
--auto-scaling-group-name asg-mixed-arch \
--preferences '{"InstanceWarmup": 0, "MinHealthyPercentage": 0}'
```

### Step 5: check that instances are healthy in Target group

![image](https://user-images.githubusercontent.com/75417152/163221701-2fa7210e-16b9-422c-b714-84cd7fadcef0.png)

![image](https://user-images.githubusercontent.com/75417152/163221811-76952a67-447e-4697-8aff-749e48e89a60.png)


### Step 6: Let's see how app is running on mixed instances (x86 and Graviton) behind load balancer
![image](https://user-images.githubusercontent.com/75417152/163224071-fd578fcd-830c-4447-90fa-146f73b597bc.png)

![image](https://user-images.githubusercontent.com/75417152/163224196-6ef45418-1ced-4cc3-95da-11ea154878ae.png)

![image](https://user-images.githubusercontent.com/75417152/163224609-98e6a229-4165-43b7-b317-75fecde21fbb.png)

![image](https://user-images.githubusercontent.com/75417152/163224694-4bb80f08-a024-4978-96cc-61c4e70bd026.png)

=======================================================================================

## Module-2: Build, deploy and run multi-arch containers on a multi-arch Amazon EKS cluster (with x86 and Graviton instances)
In this module of the workshop, you will be creating EKS managed node groups with Graviton and x86 instances in an Amazon EKS cluster. Then, you will build a multi-arch container image of a sample node.js application for x86_64 and arm64 target architectures and store it on Amazon ECR with a single manifest list. Finally,  you will be deploying pods with multi-arch container image on x86 and Graviton based nodes on EKS cluster

![image](https://user-images.githubusercontent.com/75417152/164983321-19ba8862-0cc7-4a58-a902-c1160be001e0.png)


### Step 1: Check the Amazon EKS cluster
```bash
eksctl get cluster
aws eks update-kubeconfig --name eksworkshop-eksctl
kubectl get nodes
```
You should see two nodes running in the EKS cluster.
![image](https://user-images.githubusercontent.com/75417152/167797240-1e30e807-b2a6-48dc-83e7-7e84f4546d1e.png)


Let's now install some Kubernetes tools in the EKS cluster

### Step 2: Install Helm CLI
Helm is a package manager for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called Chart.

Helm is a tool that streamlines installing and managing Kubernetes applications. Think of it like apt/yum/homebrew for Kubernetes. We will use Helm during the workshop to install other components out from the list of available charts. To install, please follow instructions below:

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
Verify the version
```bash
helm version --short
```
Let’s configure our first Chart repository. Chart repositories are similar to APT or yum repositories that you might be familiar with on Linux, or Taps for Homebrew on macOS.

Download the stable repository so we have something to start with:
```bash
helm repo add stable https://charts.helm.sh/stable/
helm repo update
```
List the helm charts you can install:
```bash
helm search repo stable
```
Finally, let’s configure Bash completion for the helm command:
```bash
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```

### Step 3: Install KUBE-OPS-VIEW
Now that we have helm installed, we are ready to use the stable helm catalog and install tools that will help with understanding our cluster setup in a visual way. The first of those tools that we are going to install is Kube-ops-view from Henning Jacobs.

The following line updates the stable helm repository and then installs kube-ops-view using a LoadBalancer Service type and creating a RBAC (Resource Base Access Control) entry for the read-only service account to read nodes and pods information from the cluster.
```bash
helm install kube-ops-view \
stable/kube-ops-view \
--set service.type=LoadBalancer \
--set nodeSelector.intent=control-apps \
--version 1.2.4 \
--set rbac.create=True
```
The execution above installs kube-ops-view exposing it through a Service using the LoadBalancer type. A successful execution of the command will display the set of resources created and will prompt some advice asking you to use kubectl proxy and a local URL for the service. Given we are using the type LoadBalancer for our service, we can disregard this; Instead we will point our browser to the external load balancer.

Let's check that chart was installed successfully:
```bash
helm list
```
Now, we can explore kube-ops-view output by checking the details about the newly created service.
```bash
kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'
```
This will display a line similar to Kube-ops-view URL = http://<URL_PREFIX_ELB>.amazonaws.com Opening the URL in your browser will provide the current state of our cluster. Check out different components
![image](https://user-images.githubusercontent.com/75417152/165549692-32d52652-46dd-4ea8-81e4-2928a6965b47.png)

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
### Let's now check nodes available in EKS cluster along with their CPU architectures.
```bash
kubectl get nodes --label-columns=kubernetes.io/arch
```
You can visualize same with kube-ops-view:
```bash
kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'
```
This will display a line similar to Kube-ops-view URL = http://<URL_PREFIX_ELB>.amazonaws.com Opening the URL in your browser will provide the current state of our cluster. You can see two new nodes created with m5.xlarge (x86_64) and m6g.xlarge (ARM64) instance types.

### Step 5: Install Docker Buildx and configure for building images for multiple target architectures
We will now use the Docker Buildx CLI plug-in that extends the Docker command to transparently build multi-arch images, link them together with a manifest file, and push them all to Amazon ECR repository using a single command. Let's install Buildx first.
```bash
wget https://github.com/docker/buildx/releases/download/v0.8.2/buildx-v0.8.2.linux-amd64
mkdir -p ~/.docker/cli-plugins
mv buildx-v0.8.2.linux-amd64 buildx
mv buildx ~/.docker/cli-plugins/docker-buildx
chmod a+x ~/.docker/cli-plugins/docker-buildx
```
Enter the following command to configure Buildx binary for different architectures. The following command installs emulators so that you can run and build containers for x86 and Arm64.
```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```

Check to see a list of build environments. If this is first time, you should only see the default builder.
```bash
docker buildx ls
```
We recommend using new builder. Enter the following command to create a new builder named mybuild and switch to it to use it as default. The bootstrap flag ensures that the driver is running.
```bash
docker buildx create --name mybuild --use
docker buildx inspect --bootstrap
docker buildx ls
```

### Step 6: Create multi-arch images for x86 and Arm64 platforms and push them to your Amazon ECR repository
Interpreted and bytecode-compiled languages such as Java and Node.js tend to work without any code modifications, unless they are pulling in any native binary extensions. In order to run a Node.js docker image on both x86 and Arm64, you must build images for those two architectures. Using Docker Buildx, you can build images for both x86 and Arm64 then push those container images to Amazon ECR at the same time.

Check your ECR repository named "myrepo" available in pre-deployed workshop environment:

```bash
aws ecr describe-repositories
```

Now, authenticate your Docker client to your Amazon ECR registry so that you can use the docker push commands to push images to the repositories. Enter the following command to retrieve an authentication token and authenticate your Docker client to your Amazon ECR registry. 

```bash
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

Explore your application files to build container images:
* app.js
* Dockerfile
* package.json

Now run following command to create a package-lock.json file:
```bash
npm install
```

Now, create your multi-arch images with the docker buildx. This single command instructs Buildx to create images for x86 and Arm64 architecture, generate a multi-arch manifest and push all images to your myrepo Amazon ECR registry.

```bash
docker buildx build --platform linux/amd64,linux/arm64 --tag ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/myrepo:latest --push .
```
Inspect the manifest and images created:

```bash
docker buildx imagetools inspect ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/myrepo:latest
```
### Step 7: Let's now deploy and run multi-arch container images on our mixed-arch Amazon EKS cluster
Explore your Kubernetes service and pod deployment config.

### Check "multiarch-app-deployment.yaml" file and update container image URI with your {ACCOUNT_ID}

```bash
---
apiVersion: v1
kind: Service 
metadata: 
  name: multiarch-app 
spec: 
  type: LoadBalancer 
  ports: 
    - port: 80
      targetPort: 80 
  selector: 
    app: multiarch-app 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multiarch-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multiarch-app
  template:
    metadata:
      labels:
        app: multiarch-app
    spec:
      nodeSelector:
        intent: mng-multiarch-apps
      containers:
        - name: multiarch-app
          image: {Put your Account ID}.dkr.ecr.us-west-2.amazonaws.com/myrepo:latest
          resources:
            requests:
              cpu: 2
              memory: 4Gi
          ports: 
            - containerPort: 80
```

### Deploy Kubernetes service with a load balancer and deploy multi-arch container app with 2 replicas on EKS cluster
```bash
kubectl apply -f multiarch-app-deployment.yaml
kubectl get svc multiarch-app | tail -n 1 | awk '{ print "multiarch-app URL = http://"$4 }'
```
This will display a line similar to multiarch-app URL = http://<URL_PREFIX_ELB>.amazonaws.com Open your browser and paste this Load Balancer DNS name url. Refresh to see the output from your multi-arch container app switching between x86 and Graviton2 instances.
![image](https://user-images.githubusercontent.com/75417152/164982645-ecb24afb-3dcc-4ce1-8e7e-c64c9828a640.png)

![image](https://user-images.githubusercontent.com/75417152/164982676-20c2660b-7c9b-418f-ad6f-e87f2f38befa.png)

You can also visualize your pods with multi-arch container image running on x86 and Graviton instances in EKS cluster with KUBE-OPS-VIEW:
```bash
kubectl get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'
```

![image](https://user-images.githubusercontent.com/75417152/164983135-d32fde7b-abea-41db-a3b4-4f09eca65c73.png)


# Key Takeaways
* You learnt how to configure your Auto Scaling group to use both Graviton2 and x86 based instances in the same Auto Scaling group with different AMIs using multiple launch templates
* You also deployed node.js application using launch template user data with two different launch templates (installing x86 and arm64 compatible binaries)
* You learnt how to build multi-architecture docker containers for your node.js application using Buildx, push and store them on Amazon ECR and deploy them on a mixed-arch EKS cluster running Graviton and x86 instances

# Additional Resources:
Blog: https://aws.amazon.com/blogs/containers/deploy-a-spring-boot-application-on-a-multi-architecture-amazon-eks-cluster/

Blog: https://aws.amazon.com/blogs/devops/creating-multi-architecture-docker-images-to-support-graviton2-using-aws-codebuild-and-aws-codepipeline/

Blog: https://www.docker.com/blog/multi-platform-docker-builds/

Containers on Graviton: https://github.com/aws/aws-graviton-getting-started/blob/main/containers.md

Graviton Workshop: https://graviton2-workshop.workshop.aws/en/amazoncontainers.html
