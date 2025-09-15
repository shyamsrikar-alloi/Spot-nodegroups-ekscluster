# Spot-nodegroups-ekscluster

```
nano create-spot-nodegroup.sh
```

```
#!/bin/bash

# === CONFIGURE THESE ===
CLUSTER_NAME="opshealth-dev-eks"
NODEGROUP_NAME="spot-node-group-eks"
REGION="us-west-2"
INSTANCE_TYPES="t3.medium"  # single instance type
MIN_SIZE=2
MAX_SIZE=8
DESIRED_SIZE=4
DISK_SIZE=20

# Optional labels and taints (correct JSON format for EKS)
LABELS="spot=true,workload=non-critical"
TAINTS='[{"key":"spot","value":"true","effect":"NO_SCHEDULE"}]'

echo ":satellite_antenna: Fetching subnet IDs for cluster '$CLUSTER_NAME'..."
SUBNET_IDS=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION \
  --query "cluster.resourcesVpcConfig.subnetIds" --output text)

echo ":closed_lock_with_key: Fetching IAM role for EKS node group..."
NODE_ROLE_ARN=$(aws iam list-roles --query "Roles[?contains(RoleName, 'AmazonEKSNodeRole')].Arn" --output text | head -n 1)

if [ -z "$NODE_ROLE_ARN" ]; then
  echo ":x: No IAM role found with name containing 'AmazonEKSNodeRole'. Please create one and retry."
  exit 1
fi

echo ":rocket: Creating Spot node group with instance type '$INSTANCE_TYPES'..."
aws eks create-nodegroup \
    --cluster-name $CLUSTER_NAME \
    --nodegroup-name $NODEGROUP_NAME \
    --scaling-config minSize=$MIN_SIZE,maxSize=$MAX_SIZE,desiredSize=$DESIRED_SIZE \
    --disk-size $DISK_SIZE \
    --subnets $SUBNET_IDS \
    --instance-types $INSTANCE_TYPES \
    --node-role $NODE_ROLE_ARN \
    --capacity-type SPOT \
    --labels $LABELS \
    --taints "$TAINTS" \
    --tags k8s.io/cluster-autoscaler/enabled=true,k8s.io/cluster-autoscaler/$CLUSTER_NAME=true \
    --region $REGION

echo ":hourglass: Waiting for node group '$NODEGROUP_NAME' to become ACTIVE..."
aws eks wait nodegroup-active --cluster-name $CLUSTER_NAME --nodegroup-name $NODEGROUP_NAME --region $REGION

echo ":white_check_mark: Spot node group '$NODEGROUP_NAME' created successfully!"

```

### modified the subnets by enabling the auto assign public ip for the below subnets
<img width="1858" height="955" alt="image" src="https://github.com/user-attachments/assets/d4a7a64e-3f03-4bc7-96ee-d45900fead06" />


### Apply

```
./create-spot-nodegroup.sh
```

<img width="1858" height="1011" alt="image" src="https://github.com/user-attachments/assets/234eef76-114a-4c58-bd3a-553f025fc418" />

<img width="1840" height="874" alt="image" src="https://github.com/user-attachments/assets/649a5054-122d-4c63-bbb6-924a60f1bfdb" />




## If we need shell access to the node then use this

```
#!/bin/bash

# === CONFIGURE THESE ===
CLUSTER_NAME="opshealth-dev-eks"
NODEGROUP_NAME="spot-node-group-eks"
REGION="us-west-2"
INSTANCE_TYPE="t3.medium"
KEY_PAIR_NAME="ng-spot-keypair"

echo ":satellite_antenna: Fetching subnet IDs for cluster '$CLUSTER_NAME'..."
SUBNET_IDS=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION \
  --query "cluster.resourcesVpcConfig.subnetIds" --output text)

echo ":closed_lock_with_key: Fetching IAM role for EKS node group..."
NODE_ROLE_ARN=$(aws iam list-roles --query "Roles[?contains(RoleName, 'AmazonEKSNodeRole')].Arn" --output text | head -n 1)

if [ -z "$NODE_ROLE_ARN" ]; then
  echo ":x: No IAM role found with name containing 'AmazonEKSNodeRole'. Please create one and retry."
  exit 1
fi

echo ":rocket: Creating Spot node group with SSH access..."
aws eks create-nodegroup \
    --cluster-name $CLUSTER_NAME \
    --nodegroup-name $NODEGROUP_NAME \
    --scaling-config minSize=2,maxSize=8,desiredSize=4 \
    --disk-size 20 \
    --subnets $SUBNET_IDS \
    --instance-types $INSTANCE_TYPE \
    --node-role $NODE_ROLE_ARN \
    --capacity-type SPOT \
    --tags k8s.io/cluster-autoscaler/enabled=true,k8s.io/cluster-autoscaler/$CLUSTER_NAME=true \
    --remote-access ec2SshKey=$KEY_PAIR_NAME \
    --region $REGION

echo ":hourglass: Waiting for node group '$NODEGROUP_NAME' to become ACTIVE..."
aws eks wait nodegroup-active --cluster-name $CLUSTER_NAME --nodegroup-name $NODEGROUP_NAME --region $REGION
```



# Step 1: Create an SQS queue
Queue-Processor mode relies on AWS SQS for receiving Spot termination/rebalance events.
```
# Create the queue
aws sqs create-queue --queue-name node-termination-handler-queue

# Get the queue URL
QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name node-termination-handler-queue \
  --query 'QueueUrl' \
  --output text)

echo $QUEUE_URL
```
# Step 2: Set up EventBridge rules
AWS publishes Spot interruption and rebalance events. We need to forward them to the SQS queue.
```
# Spot Interruption events
aws events put-rule --name spot-interruption \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Spot Instance Interruption Warning"]}'

aws events put-targets --rule spot-interruption \
  --targets "Id"="1","Arn"="$(aws sqs get-queue-attributes \
      --queue-url $QUEUE_URL \
      --attribute-name QueueArn \
      --query 'Attributes.QueueArn' \
      --output text)"
```

(Optional) Repeat for rebalance recommendations:
```
aws events put-rule --name spot-rebalance \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Spot Instance Rebalance Recommendation"]}'

aws events put-targets --rule spot-rebalance \
  --targets "Id"="1","Arn"="$(aws sqs get-queue-attributes \
      --queue-url $QUEUE_URL \
      --attribute-name QueueArn \
      --query 'Attributes.QueueArn' \
      --output text)"
```
# Step 3: Create IAM role and ServiceAccount
- NTH needs AWS permissions to:
- Read SQS queue messages
- Drain EC2 nodes / modify ASG

### policy name: NodeTerminationHandlerQueuePolicy
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-west-2:533267292058:node-termination-handler-queue"
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:UpdateAutoScalingGroup",
        "autoscaling:DescribeAutoScalingInstances",
        "ec2:DescribeInstances",
        "ec2:DescribeTags",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```
# creating service account
```
eksctl create iamserviceaccount \
  --cluster opshealth-dev-eks \
  --namespace node-termination-handler \
  --name aws-node-termination-handler \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess \
  --attach-policy-arn arn:aws:iam::aws:policy/AutoScalingFullAccess \
  --attach-policy-arn arn:aws:iam::533267292058:policy/NodeTerminationHandlerQueuePolicy \
  --approve
```
<img width="1790" height="440" alt="image" src="https://github.com/user-attachments/assets/008fc03f-08a8-4f33-b406-b2f9f702e874" />

# Step 4: Helm install NTH as a Deployment
- Now we install NTH as a Deployment, targeting stable nodes (not the Spot nodes themselves).
```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
```
helm upgrade --install node-termination-handler eks/aws-node-termination-handler \
  --namespace node-termination-handler \
  --create-namespace \
  --set enableSqsTerminationDraining=true \
  --set queueURL=$QUEUE_URL \
  --set serviceAccount.name=aws-node-termination-handler \
  --set serviceAccount.create=false
```
<img width="1859" height="504" alt="image" src="https://github.com/user-attachments/assets/ec7cba08-b3c2-4bdd-9d7e-a0253e83e04c" />






# Karpenter
- KarpenterControllerPolicy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances",
                "ec2:DescribeInstances",
                "ec2:DescribeLaunchTemplates",
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:TerminateInstances",
                "iam:PassRole",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeAvailabilityZones"
            ],
            "Resource": "*"
        }
    ]
}
```
## Attach it
```
eksctl create iamserviceaccount \
  --cluster opshealth-dev-eks \
  --namespace node-termination-handler \
  --name karpenter \
  --attach-policy-arn arn:aws:iam::533267292058:policy/KarpenterControllerPolicy \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess \
  --approve
```
<img width="1857" height="498" alt="image" src="https://github.com/user-attachments/assets/c3bff3ac-101d-408f-9ef8-2650ec412867" />

## created a role and instance profile and attached to karpenter while installing
<img width="1551" height="709" alt="image" src="https://github.com/user-attachments/assets/ed58d70a-dda5-4d39-82a8-c071974a2b40" />

```
helm repo add karpenter https://charts.karpenter.sh
```
```
helm repo update
```
```
helm upgrade --install karpenter karpenter/karpenter \
  --namespace node-termination-handler \
  --set serviceAccount.create=false \
  --set serviceAccount.name=karpenter \
  --set clusterName=opshealth-dev-eks \
  --set clusterEndpoint=$(aws eks describe-cluster --name opshealth-dev-eks --query "cluster.endpoint" --output text) \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-opshealth-dev-eks  
```

<img width="1753" height="308" alt="image" src="https://github.com/user-attachments/assets/162e0457-7519-4dd4-9d22-69e124a8e2c5" />


- tag the subnets and security groups

<img width="1851" height="441" alt="image" src="https://github.com/user-attachments/assets/d620ad1e-39c6-4b92-802e-5c772f02742b" />

```
 nano aws-node-template.yaml
```

```
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: spot-template
spec:
  subnetSelector:
    karpenter.sh/discovery: opshealth-dev-eks
  securityGroupSelector:
    karpenter.sh/discovery: opshealth-dev-eks
```

```
kubectl apply -f aws-node-template.yaml
```

```
nano provisioner.yaml
```
```
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot-provisioner
spec:
  providerRef:
    name: spot-template          # must match AWSNodeTemplate name
  requirements:
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64"]
    - key: karpenter.k8s.aws/instance-family
      operator: In
      values: ["t3"]
    - key: karpenter.k8s.aws/instance-size
      operator: In
      values: ["medium"]
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  taints:
    - key: spot
      value: "true"
      effect: NoSchedule
  ttlSecondsAfterEmpty: 300
```
```
kubectl apply -f provisioner.yaml
```




# 📚 References and Documentation

https://karpenter.sh/docs/getting-started/migrating-from-cas/
