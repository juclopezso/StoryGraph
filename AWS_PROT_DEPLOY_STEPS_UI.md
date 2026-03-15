# AWS Console — Prototype Setup Guide

> **Do these in order.** Keep a notepad open to save IDs and endpoints as you go.
> All resources go in the **same region** — use `us-east-1` throughout.

---

## 1. VPC & Networking

**Go to: VPC → Create VPC**

1. Select **"VPC and more"**
2. Name: `app-vpc`
3. IPv4 CIDR: `10.0.0.0/16`
4. Availability Zones: **1**
5. Public subnets: **1**
6. Private subnets: **1**
7. NAT Gateways: **In 1 AZ** *(required — ECS needs this to pull images)*

NOTE: 

NAT Gateways → select Regional - new
This is the new multi-AZ option AWS just introduced. One NAT Gateway covers all your subnets — simpler and slightly cheaper than the old Zonal option. This is what you want.
VPC Endpoints → select S3 Gateway
This is free and means your ECS containers talk to S3 (and ECR, which uses S3 under the hood) directly through the VPC instead of going through the NAT Gateway. It reduces your NAT Gateway costs and is just a better default. Always pick this.
DNS options → leave both checkboxes enabled (default)
These are required for RDS, DocumentDB, and ElastiCache to be reachable by hostname inside your VPC. Don't touch them.
8. Click **Create VPC**

📋 Save: VPC ID, public subnet ID, private subnet ID.

---

**Create Security Groups — VPC → Security groups → Create security group**


### How to create each Security Group

**Go to: VPC → Security groups → Create security group**

---

#### Group 1 — `alb-sg`

| Field | Value |
|-------|-------|
| Security group name | `alb-sg` |
| Description | `Load balancer inbound` |
| VPC | select `app-vpc` |

Under **Inbound rules** → click **Add rule**:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| HTTP | TCP | 80 | `0.0.0.0/0` |

Leave **Outbound rules** as default → click **Create security group**.

📋 Save the security group ID (format: `sg-0abc1234...`).

---

#### Group 2 — `ecs-sg`

| Field | Value |
|-------|-------|
| Security group name | `ecs-sg` |
| Description | `ECS Fargate tasks` |
| VPC | select `app-vpc` |

Under **Inbound rules** → click **Add rule**:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | 3000 | start typing `alb-sg` and select it from the dropdown |

> You're referencing the security group itself as the source, not an IP. This means only traffic coming from the ALB is allowed in.

Leave **Outbound rules** as default → click **Create security group**.

📋 Save the security group ID.

---

#### Group 3 — `databases-sg`

| Field | Value |
|-------|-------|
| Security group name | `databases-sg` |
| Description | `RDS, DocumentDB, Redis` |
| VPC | select `app-vpc` |

Under **Inbound rules** → click **Add rule** 3 times, one row per database:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | 5432 | select `ecs-sg` from dropdown |
| Custom TCP | TCP | 27017 | select `ecs-sg` from dropdown |
| Custom TCP | TCP | 6379 | select `ecs-sg` from dropdown |

Leave **Outbound rules** as default → click **Create security group**.

📋 Save the security group ID.

---

**What each port is for:**

| Port | Database |
|------|----------|
| 5432 | PostgreSQL (RDS) |
| 27017 | MongoDB (DocumentDB) |
| 6379 | Redis (ElastiCache) |

Once all three are created you should see them listed in **VPC → Security groups** filtered to your `app-vpc`.

---

## 2. S3 + CloudFront

**Go to: S3 → Create bucket**

1. Name: `my-app-frontend-yourname` *(must be globally unique)*
2. Region: `us-east-1`
3. Block all public access: ✅ keep checked
4. Click **Create bucket**
5. Open the bucket → **Upload** → upload all files from your `dist/` folder *(run `npm run build` first)*

---

**Go to: CloudFront → Create distribution**

### Step 1 — Open CloudFront

1. In the AWS Console top search bar, type **CloudFront**
2. Click **CloudFront** in the results
3. Click the orange **Create distribution** button

---

### Step 2 — Origin Settings

You'll see a section called **Origin** at the top.

1. Click the **Origin domain** field — a dropdown appears with your AWS resources
2. Find and select your S3 bucket (it will look like `my-app-frontend-yourname.s3.us-east-1.amazonaws.com`)
3. **Name** field auto-fills — leave it as is
4. Under **Origin access** you'll see 3 radio options — select **Origin access control settings (recommended)**
5. A new field appears called **Origin access control** — click **Create new OAC**
6. A small modal appears with a pre-filled name — don't change anything → click **Create**
7. The OAC now appears selected in the dropdown

> ⚠️ Do **not** select "Public" — that would expose your S3 bucket to the internet directly.

---

### Step 3 — Default Cache Behavior Settings

Scroll down to the **Default cache behavior** section.

1. **Viewer protocol policy** → select **Redirect HTTP to HTTPS**
2. **Allowed HTTP methods** → leave as **GET, HEAD**
3. Everything else in this section → leave as default

---

### Step 4 — Web Application Firewall

Scroll down to the **Web Application Firewall (WAF)** section.

- Select **Do not enable security protections** *(for prototype — WAF adds cost)*

---

### Step 5 — Settings

Scroll down to the **Settings** section at the bottom.

1. **Default root object** — type `index.html` in this field
2. Everything else → leave as default

---

### Step 6 — Create the Distribution

1. Click the orange **Create distribution** button at the bottom right
2. You land on your distribution's detail page — it will show **Status: Deploying** (takes 3–5 minutes)

---

### Step 7 — Copy and Apply the Bucket Policy

Right after creating, you'll see a **blue banner** at the top of the page that says:

> *"S3 bucket policy needs to be updated"*

1. Click the **Copy policy** button inside that banner
2. Open a new browser tab → go to **S3**
3. Click your bucket `my-app-frontend-yourname`
4. Click the **Permissions** tab
5. Scroll to **Bucket policy** → click **Edit**
6. The editor will be empty — **paste** the policy you copied
7. Click **Save changes**

> This policy tells S3 to only accept requests coming from your CloudFront distribution — nothing else can access the bucket directly.

📋 Go back to CloudFront → your distribution → copy the **Distribution domain name** (looks like `https://dxxxxx.cloudfront.net`). Save this — it's your app's URL.

---

### Step 8 — Custom Error Response (for React Router)

Without this, if a user refreshes the page on any route other than `/`, they'll get a 403 error from S3 instead of your app.

1. On your distribution page, click the **Error pages** tab
2. Click **Create custom error response**
3. Fill in exactly:

| Field | Value |
|-------|-------|
| HTTP error code | `403: Forbidden` |
| Customize error response | **Yes** |
| Response page path | `/index.html` |
| HTTP response code | `200: OK` |

4. Click **Create custom error response**
5. Repeat the same steps for error code **`404: Not Found`** with the same response values

> You need both 403 and 404 because S3 sometimes returns 403 and other times 404 for missing paths — covering both ensures React Router always loads.

---

### How to verify it works

Once the distribution status changes from **Deploying** to **Enabled** (3–5 min):

1. Open `https://dxxxxx.cloudfront.net` in your browser — your React app should load
2. Navigate to any route in your app, then hit refresh — it should still load correctly
---

## 3. Cognito

**Go to: Cognito → Create user pool**

1. Sign-in option: ✅ **Email**
2. Click **Next**
3. Password policy: **Cognito defaults** (keep as is)
4. MFA: **No MFA** *(prototype)*
5. Click **Next** through the remaining steps until **Integrate your app**
6. User pool name: `app-users`
7. Hosted UI: ✅ **Use the Cognito Hosted UI**
8. Cognito domain: type a unique name e.g. `my-app-login-yourname`
9. App client name: `app-client`
10. Callback URL: `https://dxxxxx.cloudfront.net/callback` *(your CloudFront URL)*
11. Sign-out URL: `https://dxxxxx.cloudfront.net`
12. Click **Create user pool**

📋 Save: User Pool ID, App Client ID, Hosted UI domain.

---

## 4. Lambda + SQS (Post-Signup Event)

**Go to: SQS → Create queue**

1. Type: **Standard**
2. Name: `user-signup-queue`
3. Click **Create queue**

📋 Save: Queue URL and Queue ARN.

---

**Go to: IAM → Roles → Create role**

1. Trusted entity: **AWS service** → Use case: **Lambda**
2. Attach policies:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonSQSFullAccess` *(simple for prototype)*
3. Name: `lambda-post-signup-role`
4. Click **Create role**

---

**Go to: Lambda → Create function**

1. **Author from scratch**
2. Name: `post-signup-handler`
3. Runtime: **Node.js 20.x**
4. Execution role: **Use an existing role** → `lambda-post-signup-role`
5. Click **Create function**
6. In the **Code** tab, replace the default code with:

```js
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";
const sqs = new SQSClient({});

export const handler = async (event) => {
  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.QUEUE_URL,
    MessageBody: JSON.stringify({
      email: event.request.userAttributes.email,
      sub: event.userName,
    }),
  }));
  return event; // must return event back to Cognito
};
```

7. Click **Deploy**
8. Go to **Configuration → Environment variables → Edit**
   - Key: `QUEUE_URL` | Value: *(your SQS queue URL)*
   - Save

---

**Attach the Lambda to Cognito**

1. Go to **Cognito → your user pool → User pool properties**
2. Scroll to **Lambda triggers → Add Lambda trigger**
3. Trigger type: **Sign-up → Post confirmation**
4. Lambda: select `post-signup-handler`
5. Click **Add Lambda trigger**

---

## 5. ECR — Container Registry

**Go to: ECR → Create repository** — repeat 3 times:

| Repository name |
|----------------|
| `user-service` |
| `business-service` |
| `processing-service` |

Settings for each: **Private**, leave everything else default.

📋 Save: the URI for each repo (format: `ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/service-name`)

**Push your images from your terminal:**
```bash
# Authenticate
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build, tag, push (repeat for each service)
docker build -t user-service ./services/user-service
docker tag user-service:latest ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
```

---

## 6. ECS — Fargate Cluster & Services

### 6.1 Create the Cluster

**Go to: ECS → Clusters → Create cluster**

1. Name: `app-cluster`
2. Infrastructure: ✅ **AWS Fargate**
3. Click **Create**

---

### 6.2 Create IAM Task Execution Role

**Go to: IAM → Roles → Create role**

1. Trusted entity: **AWS service** → Use case: **Elastic Container Service Task**
2. Attach policy: `AmazonECSTaskExecutionRolePolicy`
3. Name: `ecsTaskExecutionRole`
4. Click **Create role**

---

### 6.3 Create Task Definitions

**Go to: ECS → Task definitions → Create new task definition**

Repeat for each of the 3 services with the values below:

| Field | Value |
|-------|-------|
| Family name | `user-service` / `business-service` / `processing-service` |
| Launch type | **AWS Fargate** |
| CPU | **0.25 vCPU** |
| Memory | **0.5 GB** |
| Task execution role | `ecsTaskExecutionRole` |

Under **Container**:
1. Name: `user-service` *(match the task family)*
2. Image URI: *(your ECR image URI)*
3. Container port: `3000`
4. Click **Add**
5. Click **Create**

---

### 6.4 Create the Application Load Balancer

**Go to: EC2 → Load Balancers → Create load balancer → Application Load Balancer**

1. Name: `app-alb`
2. Scheme: **Internet-facing**
3. VPC: `app-vpc`
4. Subnets: select the **public subnet**
5. Security groups: select `sg-alb`
6. Listeners: keep HTTP port 80

**Create 3 Target Groups** (EC2 → Target Groups → Create target group) before finishing the ALB:

| Name | Port | Protocol |
|------|------|----------|
| `tg-user-service` | 3000 | HTTP |
| `tg-business-service` | 3000 | HTTP |
| `tg-processing-service` | 3000 | HTTP |

For each: target type = **IP**, VPC = `app-vpc`, health check path = `/health`.

Back in the ALB, after creating it:
- Go to **Listeners → HTTP:80 → Manage rules → Add rule** for each service:
  - Rule 1: Path `/users/*` → forward to `tg-user-service`
  - Rule 2: Path `/business/*` → forward to `tg-business-service`
  - Rule 3: Path `/processing/*` → forward to `tg-processing-service`

📋 Save: ALB DNS name.

---

### 6.5 Create ECS Services

**Go to: ECS → app-cluster → Services → Create**

Repeat for each of the 3 services:

1. Compute: **Launch type → FARGATE**
2. Task definition: select the matching one
3. Service name: `user-service` / `business-service` / `processing-service`
4. Desired tasks: **1**
5. Networking:
   - VPC: `app-vpc`
   - Subnets: **private subnet**
   - Security group: `sg-ecs`
   - Public IP: **Turned off**
6. Load balancing:
   - Load balancer type: **Application Load Balancer**
   - Select `app-alb`
   - Listener: `80`
   - Target group: select the matching one
7. Click **Create**

---

## 7. API Gateway

**Go to: API Gateway → Create API → HTTP API → Build**

# Step 7 — API Gateway (HTTP API)

> **Values you need before starting:**
> - CloudFront URL: `https://d226maag9rwm5s.cloudfront.net`
> - Cognito User Pool ID: `us-east-1_XXXXXX`
> - Cognito App Client ID: *(from Cognito → User pools → App clients)*
> - ALB DNS: `app-alb-510868804.us-east-1.elb.amazonaws.com`

---

## 7.1 — Create the HTTP API

1. Search **API Gateway** in the top search bar → open it
2. Click **Create API**
3. Under **HTTP API** click **Build**
4. Under **Integrations** — skip for now, click **Next**
5. **API name**: `app-api`
6. Leave everything else default → click **Next**
7. **Configure routes** — skip for now → click **Next**
8. **Stage name**: `$default` *(leave as is)*
9. **Auto-deploy**: make sure it is **Enabled** ✅
10. Click **Next** → click **Create**

📋 Save: your API URL shown on the API detail page — it looks like `https://XXXXX.execute-api.us-east-1.amazonaws.com`

---

## 7.2 — Add the Cognito JWT Authorizer

1. On your API page, click **Authorization** in the left sidebar
2. Click the **Manage authorizers** tab
3. Click **Create**
4. Fill in:
   - **Authorizer type**: JWT
   - **Name**: `cognito-auth`
   - **Identity source**: `$request.header.Authorization`
   - **Issuer URL**: `https://cognito-idp.us-east-1.amazonaws.com/YOUR_USER_POOL_ID`
     *(replace `YOUR_USER_POOL_ID` with your actual pool ID, e.g. `us-east-1_ABC123`)*
   - **Audience**: paste your **App Client ID**
5. Click **Create**

---

## 7.3 — Create the 3 Integrations (one per service)

Go to **Integrations** in the left sidebar → click **Manage integrations** → **Create**

### Integration 1 — User service

| Field | Value |
|-------|-------|
| Integration type | **HTTP URI** |
| Integration URI | `http://app-alb-510868804.us-east-1.elb.amazonaws.com/users/{proxy}` |
| HTTP method | **ANY** |
| Integration subtype | — |

Click **Create**.

### Integration 2 — Story service

| Field | Value |
|-------|-------|
| Integration type | **HTTP URI** |
| Integration URI | `http://app-alb-510868804.us-east-1.elb.amazonaws.com/stories/{proxy}` |
| HTTP method | **ANY** |

Click **Create**.

### Integration 3 — Reader service

| Field | Value |
|-------|-------|
| Integration type | **HTTP URI** |
| Integration URI | `http://app-alb-510868804.us-east-1.elb.amazonaws.com/reading/{proxy}` |
| HTTP method | **ANY** |

Click **Create**.

---

## 7.4 — Create the 3 Routes

Go to **Routes** in the left sidebar → **Create**

### Route 1 — Users

1. **Method**: `ANY`
2. **Path**: `/users/{proxy+}`
3. Click **Create**
4. Click on the route you just created → **Attach integration** → select the user service integration
5. Click **Attach authorizer** → select `cognito-auth`

### Route 2 — Stories

1. **Method**: `ANY`
2. **Path**: `/stories/{proxy+}`
3. Click **Create**
4. Attach the story service integration
5. Attach `cognito-auth` authorizer

### Route 3 — Reading

1. **Method**: `ANY`
2. **Path**: `/reading/{proxy+}`
3. Click **Create**
4. Attach the reader service integration
5. Attach `cognito-auth` authorizer

---

## 7.5 — Configure CORS

Go to **CORS** in the left sidebar → **Configure**

| Field | Value |
|-------|-------|
| Access-Control-Allow-Origin | `https://d226maag9rwm5s.cloudfront.net` |
| Access-Control-Allow-Headers | `Authorization, Content-Type` |
| Access-Control-Allow-Methods | `GET, POST, PUT, DELETE, OPTIONS` |
| Access-Control-Max-Age | `300` |

Click **Save**.

---

## 7.6 — Test it

Your API Gateway URL is on the API detail page. Test your user service:

```
https://XXXXX.execute-api.us-east-1.amazonaws.com/users/health
```

Without a JWT token this should return **401 Unauthorized** — which means it's working correctly. The authorizer is blocking unauthenticated requests as expected.

To test with a token, pass it in the Authorization header:
```
Authorization: Bearer YOUR_JWT_TOKEN
```

---

## Summary — what you now have

```
Frontend (CloudFront)
        |
   API Gateway
   /    |    \
/users /stories /reading
  |       |        |
  ALB  →  ALB  →  ALB
  |       |        |
user   story   reader
service service service
```

All requests from the frontend go through API Gateway which validates the Cognito JWT before forwarding to the ALB → ECS services.

---

## 8. RDS — PostgreSQL

**Go to: RDS → Create database**

1. Method: **Standard create**
2. Engine: **PostgreSQL**
3. Template: **Free tier**
4. DB instance identifier: `app-postgres`
5. Master username: `appuser`
6. Master password: set and save it
7. Instance type: **db.t3.micro**
8. Storage: **20 GB** (default)
9. VPC: `app-vpc`
10. Public access: **No**
11. VPC security group: remove default, add `sg-databases`
12. Initial database name: `appdb`
13. Click **Create database**

📋 Save: endpoint hostname (shown in database details once available, ~5 min).

---

## 9. DocumentDB — MongoDB

**Go to: DocumentDB → Clusters → Create**

1. Engine version: latest
2. Instance class: **db.t3.medium** *(minimum for DocumentDB)*
3. Number of instances: **1**
4. Cluster identifier: `app-docdb`
5. Master username: `appuser`
6. Master password: set and save it
7. VPC: `app-vpc`
8. Subnet group: **Create new** → select private subnet
9. VPC security group: `sg-databases`
10. Click **Create cluster**

📋 Save: cluster endpoint (available in ~10 min).

---

## 10. ElastiCache — Redis

**Go to: ElastiCache → Create cache**

1. Choose: **Design your own cache**
2. Cluster mode: **Disabled** *(simpler for prototype)*
3. Name: `app-redis`
4. Engine: **Redis OSS**
5. Node type: **cache.t3.micro**
6. Number of replicas: **0**
7. Subnet group: **Create new** → name `redis-subnet-group` → VPC `app-vpc` → select private subnet
8. Security group: `sg-databases`
9. Click **Create**

📋 Save: Primary endpoint hostname.

---

## 11. Final Wiring Checklist

Once everything is created, add your DB endpoints as environment variables in your ECS task definitions:

**ECS → Task definitions → select task → Create new revision → edit container → Environment variables:**

```
# user-service
DB_URL         = postgres://appuser:password@YOUR_RDS_ENDPOINT:5432/appdb
REDIS_URL      = redis://YOUR_REDIS_ENDPOINT:6379
QUEUE_URL      = https://sqs.us-east-1.amazonaws.com/ACCOUNT_ID/user-signup-queue

# business-service
MONGO_URL      = mongodb://appuser:password@YOUR_DOCDB_ENDPOINT:27017/appdb?tls=true
REDIS_URL      = redis://YOUR_REDIS_ENDPOINT:6379

# processing-service
REDIS_URL      = redis://YOUR_REDIS_ENDPOINT:6379
```

After saving each new revision, go to the ECS service → **Update service** → select the latest task revision → **Update**.

---

## Prototype Cost Estimate

| Service | Config | ~Monthly cost |
|---------|--------|--------------|
| NAT Gateway | 1 AZ | ~$35 |
| RDS PostgreSQL | db.t3.micro | ~$15 |
| DocumentDB | db.t3.medium | ~$55 |
| ElastiCache Redis | cache.t3.micro | ~$12 |
| ECS Fargate | 3 × 0.25vCPU/0.5GB | ~$10 |
| ALB | 1 instance | ~$16 |
| **Total** | | **~$143/month** |

> 💡 **To save cost when not using it:** Stop the RDS instance and DocumentDB cluster from the console (they can be stopped for up to 7 days). Delete and recreate the NAT Gateway when not needed — it's the priciest item.