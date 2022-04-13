# graviton-multiarch-workshop
Graviton multi-arch workshop
## Module-1
In this module of the workshop, you will deploy a mixed architecture Auto Scaling group with x86 and Graviton instances. You will be deploying a sample node.js application with node.js dependencies with user data script by modifying launch templates.

### Step 1: Go to Cloud9 IDE

![image](https://user-images.githubusercontent.com/75417152/163193042-41ca1705-a8d8-48ac-995e-4ae8fe43339b.png)


```bash
git clone https://github.com/ashwinikumar-sa/graviton-multiarch-workshop.git
cd graviton-multiarch-workshop 
```

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
### Modify x86 Launch Template

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

### Modify Graviton Launch Template

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



