# 1. Frontend Layer

The frontend is a **static React + Vite application**.

Components:

* **Amazon S3**
  Stores the built frontend (`vite build`).

* **Amazon CloudFront**
  Serves the frontend globally and provides HTTPS.

Users access the application through the **CloudFront URL**, for example:

```
https://dxxxxx.cloudfront.net
```

Flow:

```
User Browser
     |
CloudFront
     |
S3 Bucket (React + Vite)
```

---

# 2. Authentication Layer

Authentication is handled by **Amazon Cognito** using the **Hosted UI**.

### Login Flow

1. User opens the frontend.
2. React detects the user is not authenticated.
3. React redirects to the Cognito Hosted UI login page.
4. User logs in.
5. Cognito redirects back to the frontend with **JWT tokens**.
6. Frontend uses the token to call the backend APIs.

Flow:

```
User
 |
Frontend (React)
 |
Redirect
 |
Cognito Hosted Login
 |
JWT Tokens
 |
Frontend
 |
API Requests
```

---

# 3. User Signup Event Flow

When a new user signs up, Cognito triggers a Lambda function.

Components:

* **AWS Lambda**
* **Amazon Simple Queue Service**

Purpose: create the user in your system asynchronously.

Flow:

```
User Signup
     |
Cognito
     |
PostSignup Trigger
     |
Lambda Function
     |
SQS Queue
     |
User Microservice
     |
User stored in database
```

This is a **good microservice event-driven pattern**.

---

# 4. Backend Layer (Microservices)

Your backend contains **3 microservices running in containers**.

Components:

* **Amazon API Gateway**
* **Amazon Elastic Container Service**
* **AWS Fargate**
* **Elastic Load Balancing**

### Responsibilities

**API Gateway**

* Entry point for the frontend
* Validates Cognito JWT tokens
* Routes requests to services

**ECS Fargate**

Runs the microservices:

```
Microservice 1 → User Service
Microservice 2 → Business Service
Microservice 3 → Processing/Worker Service
```

Flow:

```
Frontend
   |
API Gateway
   |
Application Load Balancer
   |
ECS Fargate Cluster
   |
--------------------------------
| User Service                 |
| Business Service             |
| Processing Service           |
--------------------------------
```

---

# 5. Persistence Layer

Databases run inside private subnets.

Components:

* **Amazon RDS**
  PostgreSQL database.

* **Amazon DocumentDB**
  MongoDB-compatible database.

* **Amazon ElastiCache**
  Redis cache.

Example usage:

```
User Service → PostgreSQL
Business Service → MongoDB
All services → Redis
```

---

# 6. Container Registry

Container images must be stored somewhere.

Component:

* **Amazon Elastic Container Registry**

Flow:

```
Build Docker image locally
      |
Push image to ECR
      |
ECS pulls image
      |
Runs microservice container
```

---

# 7. Networking

All backend infrastructure runs inside a **Amazon Virtual Private Cloud**.

Typical layout:

```
VPC
│
├─ Public Subnet
│   └ Application Load Balancer
│
└─ Private Subnets
    ├ ECS Fargate services
    ├ RDS (PostgreSQL)
    ├ DocumentDB (MongoDB)
    └ ElastiCache (Redis)
```

Databases are **not exposed to the internet**.

---

# 8. Complete Architecture Overview

```
Users
  |
CloudFront URL
  |
S3 (React + Vite Frontend)
  |
Redirect to Cognito Login
  |
Cognito Hosted UI
  |
JWT Authentication
  |
API Gateway
  |
Application Load Balancer
  |
ECS Fargate Cluster
  |
-----------------------------------
| User Service                    |
| Business Service                |
| Processing Service              |
-----------------------------------
     |            |            |
  PostgreSQL     MongoDB      Redis
   (RDS)       (DocumentDB) (ElastiCache)

User Signup Event:
Cognito → Lambda → SQS → User Service
```

---

# Why this architecture is good for a prototype

✅ Fully serverless frontend
✅ Secure authentication with Cognito
✅ Event-driven user creation
✅ Containerized microservices
✅ Managed databases
✅ No custom domain required
