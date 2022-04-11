# graviton-multiarch-workshop
Graviton multi-arch workshop
## Module-1

In this module of the workshop, you will deploy a mixed architecture Auto Scaling group with x86 and Graviton instances
```bash
git clone https://github.com/ashwinikumar-sa/graviton-multiarch-workshop.git
cd abc
```

Create Auto Scaling group
```bash
aws autoscaling create-auto-scaling-group --cli-input-json file://asg.json
```
