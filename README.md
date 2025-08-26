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
INSTANCE_TYPE="t3.medium"

echo ":satellite_antenna: Fetching subnet IDs for cluster '$CLUSTER_NAME'..."
SUBNET_IDS=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION \
  --query "cluster.resourcesVpcConfig.subnetIds" --output text)

echo ":closed_lock_with_key: Fetching IAM role for EKS node group..."
NODE_ROLE_ARN=$(aws iam list-roles --query "Roles[?contains(RoleName, 'AmazonEKSNodeRole')].Arn" --output text | head -n 1)

if [ -z "$NODE_ROLE_ARN" ]; then
  echo ":x: No IAM role found with name containing 'AmazonEKSNodeRole'. Please create one and retry."
  exit 1
fi

echo ":rocket: Creating Spot node group..."
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
    --region $REGION

echo ":hourglass: Waiting for node group '$NODEGROUP_NAME' to become ACTIVE..."
aws eks wait nodegroup-active --cluster-name $CLUSTER_NAME --nodegroup-name $NODEGROUP_NAME --region $REGION
```
### modified the subnets by enabling the auto assign public ip
<img width="1858" height="955" alt="image" src="https://github.com/user-attachments/assets/d4a7a64e-3f03-4bc7-96ee-d45900fead06" />

```
                    "message": "One or more Amazon EC2 Subnets of [subnet-0e99880d9a05bdf17, subnet-048abb54f2205da64] for node group spot-node-group-eks does not automatically assign public IP addresses to instances launched into it. If you want your instances to be assigned a public IP address, then you need to enable auto-assign public IP address for the subnet. See IP addressing in VPC guide: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-ip-addressing.html#subnet-public-ip",
                    "resourceIds": [
                        "subnet-0e99880d9a05bdf17",
                        "subnet-048abb54f2205da64"
```

### Apply

```
./create-spot-nodegroup.sh
```

<img width="1855" height="940" alt="image" src="https://github.com/user-attachments/assets/b2740d25-67d0-4c47-9f8c-7df2f1e168ac" />

<img width="1851" height="934" alt="image" src="https://github.com/user-attachments/assets/4eded2ca-2c0e-41a4-b6b1-68043473197b" />

