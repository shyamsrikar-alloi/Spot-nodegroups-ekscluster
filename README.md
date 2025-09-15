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
