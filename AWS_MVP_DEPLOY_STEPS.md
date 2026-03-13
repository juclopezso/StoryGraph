# AWS Full Architecture — Step-by-Step Setup Guide

> **Prerequisites:** AWS account, AWS CLI configured (`aws configure`), Docker installed, Node.js 18+, and a basic React + Vite app ready.

---

## Step 1 — Create the VPC and Networking

Everything lives inside a single VPC. Do this first so all other services can reference it.

**1.1 Create the VPC**

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=app-vpc}]'
```

Save the returned `VpcId` (e.g. `vpc-0abc123`).

**1.2 Create subnets**

Create 2 public subnets (for ALB) and 2 private subnets (for ECS, DBs) across two AZs for resilience.

```bash
# Public subnet - AZ a
aws ec2 create-subnet --vpc-id vpc-0abc123 \
  --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-a}]'

# Public subnet - AZ b
aws ec2 create-subnet --vpc-id vpc-0abc123 \
  --cidr-block 10.0.2.0/24 --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-b}]'

# Private subnet - AZ a
aws ec2 create-subnet --vpc-id vpc-0abc123 \
  --cidr-block 10.0.3.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-a}]'

# Private subnet - AZ b
aws ec2 create-subnet --vpc-id vpc-0abc123 \
  --cidr-block 10.0.4.0/24 --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet-b}]'
```

**1.3 Internet Gateway and routing (for public subnets)**

```bash
aws ec2 create-internet-gateway
# Returns igw-0abc123 — attach it:
aws ec2 attach-internet-gateway --vpc-id vpc-0abc123 --internet-gateway-id igw-0abc123

# Create a public route table and add a default route
aws ec2 create-route-table --vpc-id vpc-0abc123
# Returns rtb-0abc123 (public)
aws ec2 create-route --route-table-id rtb-0abc123 \
  --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0abc123

# Associate both public subnets with the public route table
aws ec2 associate-route-table --subnet-id subnet-PUBLIC-A --route-table-id rtb-0abc123
aws ec2 associate-route-table --subnet-id subnet-PUBLIC-B --route-table-id rtb-0abc123
```

**1.4 NAT Gateway (so private subnets can reach the internet for ECR pulls)**

```bash
# Allocate an Elastic IP
aws ec2 allocate-address --domain vpc
# Returns AllocationId: eipalloc-0abc123

# Create NAT Gateway in one public subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-PUBLIC-A \
  --allocation-id eipalloc-0abc123

# Add a route in the private route table pointing to NAT GW
aws ec2 create-route --route-table-id rtb-PRIVATE \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-0abc123
```

---

## Step 2 — S3 Bucket for the Frontend

**2.1 Create the bucket**

```bash
aws s3api create-bucket \
  --bucket my-app-frontend \
  --region us-east-1
```

**2.2 Block all public access (CloudFront will serve it)**

```bash
aws s3api put-public-access-block \
  --bucket my-app-frontend \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**2.3 Build and upload the React + Vite app**

```bash
# In your React project:
npm run build

# Sync the dist folder to S3
aws s3 sync ./dist s3://my-app-frontend --delete
```

---

## Step 3 — CloudFront Distribution

**3.1 Create an Origin Access Control (OAC)**

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config \
    "Name=my-app-oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3"
# Returns OACId — save it
```

**3.2 Create the distribution**

Do this from the AWS Console (easiest for a prototype) or via CLI. In the Console:

1. Go to **CloudFront → Create distribution**.
2. Origin domain: select your S3 bucket.
3. Origin access: choose **Origin Access Control → your OAC**.
4. Viewer protocol policy: **Redirect HTTP to HTTPS**.
5. Default root object: `index.html`.
6. Error pages: add a custom error for 403 → response `/index.html`, status 200 (needed for React Router).
7. Click **Create distribution**.

**3.3 Update the S3 bucket policy to allow CloudFront**

After creating the distribution, CloudFront will show you the bucket policy to copy. Paste it in S3 → Permissions → Bucket Policy.

> Your frontend is now live at `https://dxxxxx.cloudfront.net`.

---

## Step 4 — Amazon Cognito

**4.1 Create a User Pool**

```bash
aws cognito-idp create-user-pool \
  --pool-name my-app-users \
  --policies '{"PasswordPolicy":{"MinimumLength":8}}' \
  --auto-verified-attributes email
# Returns UserPoolId — save it
```

**4.2 Create an App Client**

```bash
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_XXXXXX \
  --client-name my-app-client \
  --generate-secret \
  --allowed-o-auth-flows code \
  --allowed-o-auth-scopes email openid profile \
  --allowed-o-auth-flows-user-pool-client \
  --callback-urls '["https://dxxxxx.cloudfront.net/callback"]' \
  --logout-urls '["https://dxxxxx.cloudfront.net"]' \
  --supported-identity-providers COGNITO
# Returns ClientId — save it
```

**4.3 Configure a domain for the Hosted UI**

```bash
aws cognito-idp create-user-pool-domain \
  --domain my-app-login \
  --user-pool-id us-east-1_XXXXXX
```

Your login page is now at:
```
https://my-app-login.auth.us-east-1.amazoncognito.com/login
```

**4.4 Configure the React app to use Cognito**

Install the AWS Amplify library:
```bash
npm install aws-amplify
```

In `src/main.jsx`:
```jsx
import { Amplify } from 'aws-amplify';

Amplify.configure({
  Auth: {
    region: 'us-east-1',
    userPoolId: 'us-east-1_XXXXXX',
    userPoolWebClientId: 'YOUR_CLIENT_ID',
    oauth: {
      domain: 'my-app-login.auth.us-east-1.amazoncognito.com',
      scope: ['email', 'openid', 'profile'],
      redirectSignIn: 'https://dxxxxx.cloudfront.net/callback',
      redirectSignOut: 'https://dxxxxx.cloudfront.net',
      responseType: 'code',
    },
  },
});
```

To trigger the login redirect:
```jsx
import { Auth } from 'aws-amplify';
Auth.federatedSignIn(); // redirects to Cognito Hosted UI
```

---

## Step 5 — Lambda + SQS for Post-Signup Event

**5.1 Create the SQS queue**

```bash
aws sqs create-queue --queue-name user-signup-queue
# Returns QueueUrl — save it
```

**5.2 Create the Lambda function**

Create a file `lambda/post-signup/index.js`:
```js
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');
const sqs = new SQSClient({ region: 'us-east-1' });

exports.handler = async (event) => {
  const user = {
    email: event.request.userAttributes.email,
    sub: event.userName,
    triggeredAt: new Date().toISOString(),
  };

  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.QUEUE_URL,
    MessageBody: JSON.stringify(user),
  }));

  // Must return the event back to Cognito
  return event;
};
```

Zip and deploy:
```bash
cd lambda/post-signup
zip -r function.zip index.js node_modules
aws lambda create-function \
  --function-name post-signup-handler \
  --runtime nodejs18.x \
  --role arn:aws:iam::ACCOUNT_ID:role/lambda-execution-role \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --environment Variables={QUEUE_URL=https://sqs.us-east-1.amazonaws.com/ACCOUNT_ID/user-signup-queue}
```

> You'll need an IAM role for Lambda with `AWSLambdaBasicExecutionRole` + `sqs:SendMessage` permission.

**5.3 Attach the Lambda trigger to Cognito**

```bash
aws cognito-idp update-user-pool \
  --user-pool-id us-east-1_XXXXXX \
  --lambda-config PostConfirmation=arn:aws:lambda:us-east-1:ACCOUNT_ID:function:post-signup-handler
```

---

## Step 6 — ECR — Container Registry

Create one repository per microservice.

```bash
aws ecr create-repository --repository-name user-service
aws ecr create-repository --repository-name business-service
aws ecr create-repository --repository-name processing-service
```

**Authenticate Docker and push an image:**

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

docker build -t user-service ./services/user-service
docker tag user-service:latest ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
```

Repeat for `business-service` and `processing-service`.

---

## Step 7 — ECS Fargate Cluster + Services

**7.1 Create the cluster**

```bash
aws ecs create-cluster --cluster-name app-cluster
```

**7.2 Create an IAM task execution role**

In the Console: IAM → Roles → Create role → select **ECS Task** as the use case, attach `AmazonECSTaskExecutionRolePolicy`. Name it `ecsTaskExecutionRole`.

**7.3 Register task definitions (one per service)**

Save this as `user-service-task.json`:
```json
{
  "family": "user-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "user-service",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest",
      "portMappings": [{ "containerPort": 3000 }],
      "environment": [
        { "name": "DB_URL", "value": "your-rds-endpoint" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/user-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
aws ecs register-task-definition --cli-input-json file://user-service-task.json
```

Repeat for `business-service` and `processing-service`.

**7.4 Create an Application Load Balancer**

In the Console: EC2 → Load Balancers → Create:
- Type: **Application Load Balancer**
- Scheme: **Internet-facing**
- Subnets: select both **public** subnets
- Security group: allow port 80 and 443 inbound

Create 3 target groups (one per service), type **IP**, port 3000.

Add listener rules:
- `/users/*` → user-service target group
- `/business/*` → business-service target group
- `/processing/*` → processing-service target group

**7.5 Create the ECS services**

```bash
aws ecs create-service \
  --cluster app-cluster \
  --service-name user-service \
  --task-definition user-service \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-PRIVATE-A,subnet-PRIVATE-B],securityGroups=[sg-0abc123],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=user-service,containerPort=3000"
```

Repeat for the other two services.

---

## Step 8 — API Gateway

**8.1 Create an HTTP API**

```bash
aws apigatewayv2 create-api \
  --name my-app-api \
  --protocol-type HTTP \
  --cors-configuration \
    AllowOrigins='["https://dxxxxx.cloudfront.net"]',AllowMethods='["GET","POST","PUT","DELETE"]',AllowHeaders='["Authorization","Content-Type"]'
```

**8.2 Add a Cognito JWT authorizer**

```bash
aws apigatewayv2 create-authorizer \
  --api-id YOUR_API_ID \
  --name cognito-authorizer \
  --authorizer-type JWT \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration \
    Audience=YOUR_COGNITO_CLIENT_ID,Issuer=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXX
```

**8.3 Create a VPC Link to reach the ALB**

```bash
aws apigatewayv2 create-vpc-link \
  --name my-vpc-link \
  --subnet-ids subnet-PRIVATE-A subnet-PRIVATE-B \
  --security-group-ids sg-0abc123
```

**8.4 Create integrations and routes**

For each service, create an integration pointing to the ALB via the VPC Link, then attach routes with the JWT authorizer:

```bash
# Integration for user-service
aws apigatewayv2 create-integration \
  --api-id YOUR_API_ID \
  --integration-type HTTP_PROXY \
  --integration-method ANY \
  --integration-uri arn:aws:elasticloadbalancing:...:listener/... \
  --connection-type VPC_LINK \
  --connection-id YOUR_VPC_LINK_ID \
  --payload-format-version 1.0

# Route
aws apigatewayv2 create-route \
  --api-id YOUR_API_ID \
  --route-key 'ANY /users/{proxy+}' \
  --authorization-type JWT \
  --authorizer-id YOUR_AUTHORIZER_ID \
  --target integrations/YOUR_INTEGRATION_ID
```

**8.5 Deploy the API**

```bash
aws apigatewayv2 create-stage \
  --api-id YOUR_API_ID \
  --stage-name prod \
  --auto-deploy

# Your API URL will be:
# https://XXXXX.execute-api.us-east-1.amazonaws.com/prod
```

---

## Step 9 — Persistence Layer

### 9.1 PostgreSQL — Amazon RDS

```bash
aws rds create-db-instance \
  --db-instance-identifier app-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username appuser \
  --master-user-password YourPassword123! \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-0abc123 \
  --db-subnet-group-name app-db-subnet-group \
  --no-publicly-accessible
```

> Create the DB Subnet Group first: RDS → Subnet groups → Create (select private subnets).

### 9.2 MongoDB — Amazon DocumentDB

```bash
aws docdb create-db-cluster \
  --db-cluster-identifier app-docdb \
  --engine docdb \
  --master-username appuser \
  --master-user-password YourPassword123! \
  --vpc-security-group-ids sg-0abc123 \
  --db-subnet-group-name app-db-subnet-group

aws docdb create-db-instance \
  --db-instance-identifier app-docdb-instance \
  --db-instance-class db.t3.medium \
  --engine docdb \
  --db-cluster-identifier app-docdb
```

### 9.3 Redis — Amazon ElastiCache

```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id app-redis \
  --engine redis \
  --cache-node-type cache.t3.micro \
  --num-cache-nodes 1 \
  --security-group-ids sg-0abc123 \
  --cache-subnet-group-name app-cache-subnet-group
```

> Create the Cache Subnet Group first: ElastiCache → Subnet groups → Create (select private subnets).

---

## Step 10 — Security Groups Summary

Create separate security groups for each tier. Keep it tight:

| Security Group      | Inbound rule                              |
|---------------------|-------------------------------------------|
| `sg-alb`            | 80, 443 from `0.0.0.0/0`                 |
| `sg-ecs`            | 3000 from `sg-alb` only                  |
| `sg-rds`            | 5432 from `sg-ecs` only                  |
| `sg-docdb`          | 27017 from `sg-ecs` only                 |
| `sg-redis`          | 6379 from `sg-ecs` only                  |

---

## Step 11 — Wire the User Service to the SQS Queue

Your User Service needs to poll the SQS queue for new signup events. Add an SQS consumer in the service (example in Node.js):

```js
const { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } = require('@aws-sdk/client-sqs');

const sqs = new SQSClient({ region: 'us-east-1' });
const QUEUE_URL = process.env.QUEUE_URL;

async function poll() {
  while (true) {
    const res = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: QUEUE_URL,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20, // long polling
    }));

    for (const msg of res.Messages || []) {
      const user = JSON.parse(msg.Body);
      await createUserInDatabase(user); // your DB logic
      await sqs.send(new DeleteMessageCommand({
        QueueUrl: QUEUE_URL,
        ReceiptHandle: msg.ReceiptHandle,
      }));
    }
  }
}

poll();
```

Give the ECS task role `sqs:ReceiveMessage`, `sqs:DeleteMessage`, and `sqs:GetQueueAttributes` permissions on the queue ARN.

---

## Step 12 — Final Checklist

```
[ ] VPC with public + private subnets
[ ] NAT Gateway attached to private route table
[ ] S3 bucket created, vite build uploaded
[ ] CloudFront distribution pointing to S3 via OAC
[ ] Cognito User Pool + App Client configured
[ ] Cognito Hosted UI domain created
[ ] React app configured with Amplify
[ ] Post-Signup Lambda + SQS queue deployed
[ ] Lambda trigger attached to Cognito
[ ] ECR repos created, images pushed
[ ] ECS cluster + 3 task definitions registered
[ ] ALB created with 3 target groups and path rules
[ ] ECS services running in private subnets
[ ] API Gateway HTTP API with JWT authorizer
[ ] API Gateway VPC Link → ALB integrations
[ ] RDS PostgreSQL in private subnet
[ ] DocumentDB cluster in private subnet
[ ] ElastiCache Redis in private subnet
[ ] Security groups locked down per tier
[ ] User Service polling SQS for signup events
```

---

## Cost tip for prototypes

To minimize cost while testing, use the smallest sizes: `db.t3.micro` for RDS, `cache.t3.micro` for Redis, `256 CPU / 512 MB` for Fargate tasks, and `db.t3.medium` for DocumentDB (its minimum). Stop/delete resources when not in use — RDS and DocumentDB can be stopped for up to 7 days at a time.