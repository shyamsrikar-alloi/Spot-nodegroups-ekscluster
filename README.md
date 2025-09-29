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
        "ec2:DescribeTags"
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

Folllowed this official document to install the version-1.7
-Link https://karpenter.sh/docs/getting-started/migrating-from-cas/

## Create default NodePool
- for provisioning spot nodes

```
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["t3.medium"]
        - key: spot
          operator: In
          values: ["true"]
        - key: workload
          operator: In
          values: ["non-critical"]
      taints:
        - key: spot
          value: "true"
          effect: NoSchedule
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h # 30 days
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  role: "KarpenterNodeRole-opshealth-dev-eks"
  amiFamily: AL2023
  amiSelectorTerms:
    - id: "ami-0c5e7e8af3493a528"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "opshealth-dev-eks"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "opshealth-dev-eks"
```
- Apply the file

```
kubectl apply -f nodepool.yaml
```
<img width="1855" height="119" alt="image" src="https://github.com/user-attachments/assets/075360aa-d3dc-47a8-8681-bd33b0a51973" />


## Verified

<img width="1488" height="807" alt="image" src="https://github.com/user-attachments/assets/da655902-1b57-4ae2-8306-1402253728c4" />

<img width="1857" height="943" alt="image" src="https://github.com/user-attachments/assets/74eab831-41d1-4145-aa1b-85a04c35f801" />

<img width="1849" height="983" alt="image" src="https://github.com/user-attachments/assets/e70d197a-4379-4613-893a-a0fca91af7cd" />

<img width="1855" height="980" alt="image" src="https://github.com/user-attachments/assets/3ff2eec7-1457-463b-b83e-6484db2741f2" />
<img width="1850" height="1004" alt="image" src="https://github.com/user-attachments/assets/ace6679e-dcac-462c-9288-722606147bd4" />


## Terminating
- if there is no pods to schedule on provisioned node karpenter will terminate the nodes before 5 min

<img width="1857" height="954" alt="image" src="https://github.com/user-attachments/assets/6377fd3c-7362-45ed-9775-7be8d33a85c9" />
