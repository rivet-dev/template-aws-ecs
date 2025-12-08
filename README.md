# RivetKit AWS ECS Template

A RivetKit application template configured for deployment on AWS ECS (Fargate).

## Local Development

```bash
npm install
npm run dev
```

---

## Option A: Deploy with Terraform

> **Note:** This only deploys the actors. You need to deploy the frontend yourself.

### Prerequisites

- [AWS CLI](https://aws.amazon.com/cli/) configured with appropriate permissions
- [Docker](https://www.docker.com/) installed and running
- [Terraform](https://www.terraform.io/)

### 1. Create Rivet Project

1. Go to the [Rivet dashboard](https://rivet.dev)
2. Under **Connect Existing Project**, click **AWS ECS**
3. Configure datacenter, then click **Next**
4. Copy the values shown for the next step

### 2. Deploy

Set the environment variables from the Rivet dashboard:

```bash
export TF_VAR_rivet_endpoint="<paste from dashboard>"
export TF_VAR_rivet_namespace="<paste from dashboard>"
export TF_VAR_rivet_token="<paste from dashboard>"
```

Then deploy:

```bash
cd terraform
terraform init
terraform apply
```

This will automatically build and push the Docker image.

To deploy updates, run `terraform apply` again.

### Cleanup

```bash
cd terraform
terraform destroy
```

---

## Option B: Deploy with AWS CLI

> **Note:** This only deploys the actors. You need to deploy the frontend yourself.

### Prerequisites

- [AWS CLI](https://aws.amazon.com/cli/) configured with appropriate permissions
- [Docker](https://www.docker.com/) installed and running

### 1. Create Rivet Project

1. Go to the [Rivet dashboard](https://rivet.dev)
2. Under **Connect Existing Project**, click **AWS ECS**
3. Configure datacenter, then click **Next**
4. Copy and run the environment variables shown:

```bash
export RIVET_ENDPOINT=<paste from dashboard>
export RIVET_NAMESPACE=<paste from dashboard>
export RIVET_TOKEN=<paste from dashboard>
```

### 2. Set AWS Variables

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export APP_NAME=my-backend
```

### 3. Create ECR Repository

```bash
aws ecr create-repository --repository-name ${APP_NAME} --region ${AWS_REGION}
aws ecr get-login-password --region ${AWS_REGION} | \
  docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

### 4. Build and Push Docker Image

```bash
docker build --platform linux/amd64 -t ${APP_NAME} .
docker tag ${APP_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest
```

### 5. Create CloudWatch Log Group

```bash
aws logs create-log-group --log-group-name /ecs/${APP_NAME} --region ${AWS_REGION}
```

### 6. Create ECS Cluster

```bash
aws ecs create-cluster --cluster-name ${APP_NAME}-cluster
```

### 7. Create Task Execution Role (if needed)

If you don't have an `ecsTaskExecutionRole`, create one:

```bash
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### 8. Register Task Definition

```bash
cat > /tmp/task-def.json << EOF
{
  "family": "${APP_NAME}",
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "requiresCompatibilities": ["FARGATE"],
  "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "${APP_NAME}",
    "image": "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest",
    "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
    "environment": [
      {"name": "RIVET_ENDPOINT", "value": "${RIVET_ENDPOINT}"},
      {"name": "RIVET_NAMESPACE", "value": "${RIVET_NAMESPACE}"},
      {"name": "RIVET_TOKEN", "value": "${RIVET_TOKEN}"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/${APP_NAME}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "essential": true
  }]
}
EOF
aws ecs register-task-definition --cli-input-json file:///tmp/task-def.json
```

### 9. Configure VPC

**Option A: Use existing VPC**

```bash
export VPC_ID=vpc-xxxxxxxxx
export SUBNET_ID=subnet-xxxxxxxxx
export SG_ID=sg-xxxxxxxxx
```

**Option B: Create new VPC**

```bash
export VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
export SUBNET_ID=$(aws ec2 create-subnet --vpc-id ${VPC_ID} --cidr-block 10.0.0.0/24 --query 'Subnet.SubnetId' --output text)
export SG_ID=$(aws ec2 create-security-group --group-name ${APP_NAME}-sg --description "ECS security group" --vpc-id ${VPC_ID} --query 'GroupId' --output text)
export IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id ${IGW_ID} --vpc-id ${VPC_ID}
export RTB_ID=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=${VPC_ID} --query 'RouteTables[0].RouteTableId' --output text)
aws ec2 create-route --route-table-id ${RTB_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${IGW_ID}
```

### 10. Create ECS Service

```bash
aws ecs create-service \
  --cluster ${APP_NAME}-cluster \
  --service-name ${APP_NAME} \
  --task-definition ${APP_NAME} \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${SUBNET_ID}],securityGroups=[${SG_ID}],assignPublicIp=ENABLED}"
```

### Deploy Updates

```bash
docker build --platform linux/amd64 -t ${APP_NAME} .
docker tag ${APP_NAME}:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${APP_NAME}:latest
aws ecs update-service --cluster ${APP_NAME}-cluster --service ${APP_NAME} --force-new-deployment
```

### Update Environment Variables

Update `/tmp/task-def.json` with new values, then re-register and deploy:

```bash
aws ecs register-task-definition --cli-input-json file:///tmp/task-def.json
aws ecs update-service --cluster ${APP_NAME}-cluster --service ${APP_NAME} --task-definition ${APP_NAME} --force-new-deployment
```

### Cleanup

```bash
aws ecs update-service --cluster ${APP_NAME}-cluster --service ${APP_NAME} --desired-count 0
aws ecs delete-service --cluster ${APP_NAME}-cluster --service ${APP_NAME}
aws ecs delete-cluster --cluster ${APP_NAME}-cluster
aws ecr delete-repository --repository-name ${APP_NAME} --force
aws logs delete-log-group --log-group-name /ecs/${APP_NAME}
```

---

## After Deployment

Your runner should appear as connected on the Rivet dashboard once the service is healthy.

- View your cluster: https://us-east-1.console.aws.amazon.com/ecs/v2/clusters?region=us-east-1
- View logs: https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups

