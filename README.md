# graviton-multiarch-workshop
Graviton multi-arch workshop
## Module-1

In this module of the workshop, you will deploy a mixed architecture Auto Scaling group with x86 and Graviton instances
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
```
  

Create Auto Scaling group
```bash
aws autoscaling create-auto-scaling-group --cli-input-json file://asg-config-multiarch.json
```
