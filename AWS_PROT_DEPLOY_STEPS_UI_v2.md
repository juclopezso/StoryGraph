# Guía Completa AWS Console — Prototipo

> Hacer todo en orden. Usar siempre la región **us-east-1**.
> Tener un bloc de notas abierto para guardar IDs y endpoints.

---

## 1. VPC y Networking

### 1.1 Crear la VPC

1. Buscar **VPC** → **Create VPC**
2. Seleccionar **"VPC and more"**
3. Configurar:
   - Name tag: `app-vpc`
   - IPv4 CIDR: `10.0.0.0/16`
   - Availability Zones: **1**
   - Public subnets: **1**
   - Private subnets: **1**
   - NAT Gateways: **Regional - new** *(nueva opción multi-AZ, más simple)*
   - VPC endpoints: **S3 Gateway** *(gratis, reduce costos del NAT)*
   - DNS hostnames: ✅ activado
   - DNS resolution: ✅ activado
4. Click **Create VPC**

📋 Guardar: VPC ID, ID subnet pública, ID subnet privada.

---

### 1.2 Crear segundo subnet público (requerido por el ALB)

El ALB requiere mínimo 2 subnets en 2 AZs diferentes.

**VPC → Subnets → Create subnet**

| Campo | Valor |
|-------|-------|
| VPC | `app-vpc-vpc` |
| Subnet name | `app-vpc-subnet-public2-us-east-1b` |
| Availability Zone | **us-east-1b** |
| IPv4 CIDR | `10.0.16.0/20` |

Después de crear, asociarlo a la route table pública:
1. **VPC → Route tables** → buscar la que tiene ruta `0.0.0.0/0` → Internet Gateway
2. Click en ella → **Subnet associations → Edit subnet associations**
3. Marcar el nuevo subnet público → **Save**

---

### 1.3 Crear Security Groups

**VPC → Security groups → Create security group** — crear los 3 grupos con VPC = `app-vpc`:

#### `alb-sg`
| Inbound | Protocolo | Puerto | Origen |
|---------|-----------|--------|--------|
| HTTP | TCP | 80 | `0.0.0.0/0` |

#### `ecs-sg`
| Inbound | Protocolo | Puerto | Origen |
|---------|-----------|--------|--------|
| Custom TCP | TCP | 5000 | `alb-sg` *(seleccionar el SG, no una IP)* |

> ⚠️ Usar el puerto real de tus contenedores. Si usas 3000, poner 3000.

#### `databases-sg`
| Inbound | Protocolo | Puerto | Origen |
|---------|-----------|--------|--------|
| Custom TCP | TCP | 5432 | `ecs-sg` |
| Custom TCP | TCP | 27017 | `ecs-sg` |
| Custom TCP | TCP | 6379 | `ecs-sg` |

Dejar todos los **Outbound rules** como default (All traffic permitido).

📋 Guardar los 3 IDs de security groups.

---

## 2. S3 + CloudFront

### 2.1 Crear el bucket S3

1. **S3 → Create bucket**
2. Nombre: `my-app-frontend-yourname` *(debe ser único globalmente)*
3. Region: `us-east-1`
4. Block all public access: ✅ dejar marcado
5. **Create bucket**
6. Subir archivos: ejecutar `npm run build` → subir contenido de `dist/` al bucket

### 2.2 Crear distribución CloudFront

1. **CloudFront → Create distribution**
2. Origin domain: seleccionar el bucket S3
3. Origin access: **Origin access control (recommended)** → **Create new OAC** → Create
4. Viewer protocol policy: **Redirect HTTP to HTTPS**
5. Default root object: `index.html`
6. WAF: **Do not enable security protections**
7. Click **Create distribution**
8. Copiar la política del banner azul → S3 → tu bucket → **Permissions → Bucket policy** → pegar → Save

> Si el banner desapareció, construir la policy manualmente:
> ```json
> {
>   "Version": "2008-10-17",
>   "Statement": [{
>     "Effect": "Allow",
>     "Principal": { "Service": "cloudfront.amazonaws.com" },
>     "Action": "s3:GetObject",
>     "Resource": "arn:aws:s3:::TU-BUCKET/*",
>     "Condition": {
>       "ArnLike": {
>         "AWS:SourceArn": "arn:aws:cloudfront::TU-ACCOUNT-ID:distribution/TU-DISTRIBUTION-ID"
>       }
>     }
>   }]
> }
> ```

9. En la distribución → **General → Settings → Edit** → poner `index.html` en **Default root object** → Save
10. **Error pages → Create custom error response**:
    - `403` → `/index.html` → `200`
    - `404` → `/index.html` → `200`

📋 Guardar: CloudFront URL (`https://dxxxxx.cloudfront.net`)

---

## 3. Cognito

1. **Cognito → Create user pool**
2. Sign-in: ✅ **Email**
3. Password policy: **Cognito defaults**
4. MFA: **No MFA**
5. User pool name: `app-users`
6. Hosted UI: ✅ activar
7. Cognito domain: `my-app-login-yourname`
8. App client name: `app-client`
9. Callback URLs — agregar **ambas**:
   - `https://dxxxxx.cloudfront.net/callback`
   - `http://localhost:5173/callback`
10. Sign-out URLs — agregar **ambas**:
    - `https://dxxxxx.cloudfront.net`
    - `http://localhost:5173`
11. **Create user pool**

📋 Guardar: User Pool ID, App Client ID, Hosted UI domain.

---

## 4. Lambda + SQS (Post-Signup)

### 4.1 Crear la cola SQS

1. **SQS → Create queue**
2. Type: **Standard**
3. Name: `user-signup-queue`
4. **Create queue**

📋 Guardar: Queue URL y Queue ARN.

### 4.2 Crear IAM Role para Lambda

1. **IAM → Roles → Create role**
2. Trusted entity: **AWS service → Lambda**
3. Políticas: `AWSLambdaBasicExecutionRole` + `AmazonSQSFullAccess`
4. Name: `lambda-post-signup-role`

### 4.3 Crear la función Lambda

1. **Lambda → Create function → Author from scratch**
2. Name: `post-signup-handler`
3. Runtime: **Node.js 20.x**
4. Execution role: `lambda-post-signup-role`
5. Pegar este código en el editor:

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
  return event;
};
```

6. **Deploy**
7. **Configuration → Environment variables** → agregar:
   - Key: `QUEUE_URL` | Value: tu Queue URL

### 4.4 Adjuntar trigger a Cognito

1. **Cognito → tu user pool → User pool properties → Add Lambda trigger**
2. Trigger: **Sign-up → Post confirmation**
3. Lambda: `post-signup-handler`
4. **Add Lambda trigger**

---

## 5. ECR — Container Registry

**ECR → Create repository** — repetir 3 veces:

| Nombre |
|--------|
| `user-service` |
| `story-service` |
| `reader-service` |

Settings: **Private**, todo lo demás default.

**Push desde terminal:**
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

docker build -t user-service ./services/user-service
docker tag user-service:latest ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
```

---

## 6. ECS — Fargate

### 6.1 Crear el Cluster

**ECS → Clusters → Create cluster**
- Name: `app-cluster`
- Infrastructure: ✅ **AWS Fargate**

### 6.2 IAM Task Execution Role

**IAM → Roles → Create role**
- Use case: **Elastic Container Service Task**
- Policy: `AmazonECSTaskExecutionRolePolicy`
- Name: `ecsTaskExecutionRole`

### 6.3 Task Definitions

**ECS → Task definitions → Create new task definition** — repetir por cada servicio:

| Campo | Valor |
|-------|-------|
| Family name | `user-service` / `story-service` / `reader-service` |
| Launch type | **AWS Fargate** |
| CPU | **0.25 vCPU** |
| Memory | **0.5 GB** |
| Task execution role | `ecsTaskExecutionRole` |
| Container name | mismo nombre del servicio |
| Image URI | URI completo de ECR |
| Container port | **5000** *(o el puerto real de tu app)* |

### 6.4 Application Load Balancer

**EC2 → Load Balancers → Create → Application Load Balancer**

| Campo | Valor |
|-------|-------|
| Name | `app-alb` |
| Scheme | **Internet-facing** |
| IP address type | **IPv4** |
| VPC | `app-vpc-vpc` |
| Subnets | ✅ subnet público 1a + ✅ subnet público 1b |
| Security group | `alb-sg` *(remover el default)* |

**Crear 3 Target Groups** (EC2 → Target Groups → Create):

| Campo | Valor |
|-------|-------|
| Target type | **IP addresses** |
| Protocol:Port | HTTP : **5000** |
| VPC | `app-vpc-vpc` |
| Health check path | `/health` |

Nombres: `tg-user-service`, `tg-story-service`, `tg-reader-service`

**Agregar reglas al listener HTTP:80** (después de crear el ALB):

| Prioridad | Condición (Path) | Acción |
|-----------|-----------------|--------|
| 1 | `/users/*` | Forward → `tg-user-service` |
| 2 | `/stories/*` | Forward → `tg-story-service` |
| 3 | `/reading/*` | Forward → `tg-reader-service` |

> El ALB reenvía el path completo al servicio. Si tu app tiene rutas `/health`, el ALB enviará `/users/health`. Asegúrate de que tus servicios manejen el prefijo completo.

📋 Guardar: ALB DNS name.

### 6.5 ECS Services

**ECS → app-cluster → Services → Create** — repetir por cada servicio:

| Campo | Valor |
|-------|-------|
| Launch type | **FARGATE** |
| Task definition | la correspondiente |
| Service name | nombre del servicio |
| Desired tasks | **1** |
| VPC | `app-vpc-vpc` |
| Subnets | ✅ **solo el subnet PRIVADO** |
| Security group | `ecs-sg` |
| Public IP | **Turned off** |
| Load balancer | `app-alb` |
| Target group | el correspondiente |

> ⚠️ Usar **únicamente el subnet privado**. Poner subnets públicos causa el error de ECR pull porque sin IP pública no hay salida a internet.

---

## 7. API Gateway

### 7.1 Crear HTTP API

1. **API Gateway → Create API → HTTP API → Build**
2. Saltar integraciones por ahora → **Next**
3. API name: `app-api`
4. Stage name: `$default`, Auto-deploy: ✅
5. **Create**

📋 Guardar: API URL (`https://XXXXX.execute-api.us-east-1.amazonaws.com`)

### 7.2 JWT Authorizer

1. **Authorization → Manage authorizers → Create**
2. Type: **JWT**
3. Name: `cognito-auth`
4. Identity source: `$request.header.Authorization`
5. Issuer URL: `https://cognito-idp.us-east-1.amazonaws.com/TU_USER_POOL_ID`
6. Audience: tu **App Client ID**
7. **Create**

### 7.3 Crear 3 Integraciones

**Integrations → Manage integrations → Create** — repetir por cada servicio:

| Servicio | Integration URI |
|---------|----------------|
| user-service | `http://TU_ALB_DNS/users/{proxy}` |
| story-service | `http://TU_ALB_DNS/stories/{proxy}` |
| reader-service | `http://TU_ALB_DNS/reading/{proxy}` |

- Integration type: **HTTP URI**
- HTTP method: **ANY**

### 7.4 Crear 3 Routes

**Routes → Create** — repetir por cada servicio:

| Method | Path | Integration | Authorizer |
|--------|------|-------------|------------|
| ANY | `/users/{proxy+}` | user integration | `cognito-auth` |
| ANY | `/stories/{proxy+}` | story integration | `cognito-auth` |
| ANY | `/reading/{proxy+}` | reader integration | `cognito-auth` |

> Nota: el path del route usa `{proxy+}` con `+`. La URI de la integración usa `{proxy}` sin `+`. Ambos son necesarios exactamente así.

### 7.5 Configurar CORS

**CORS → Configure**

| Campo | Valor |
|-------|-------|
| Allow origins | `https://dxxxxx.cloudfront.net` y `http://localhost:5173` |
| Allow headers | `authorization`, `Authorization`, `content-type` |
| Allow methods | GET, POST, PUT, DELETE, PATCH, OPTIONS |
| Allow credentials | **NO** *(usar Authorization header, no cookies)* |

> ⚠️ No usar `*` como origin. Con el header `Authorization` el browser requiere un origin específico, `*` rompe las llamadas autenticadas.

---

## 8. Bases de Datos

### 8.1 PostgreSQL — RDS

**RDS → Create database**
- Engine: **PostgreSQL**
- Template: **Free tier**
- Identifier: `app-postgres`
- Credentials: usuario y contraseña
- Instance: `db.t3.micro`
- VPC: `app-vpc-vpc`
- Public access: **No**
- Security group: `databases-sg`
- Initial DB name: `appdb`

### 8.2 MongoDB — DocumentDB

**DocumentDB → Create**
- Instance: `db.t3.medium` *(mínimo de DocumentDB)*
- Instances: **1**
- Identifier: `app-docdb`
- VPC: `app-vpc-vpc`
- Security group: `databases-sg`

### 8.3 Redis — ElastiCache

**ElastiCache → Create cache → Design your own**
- Engine: **Redis OSS**
- Cluster mode: **Disabled**
- Name: `app-redis`
- Node type: `cache.t3.micro`
- Replicas: **0**
- VPC: `app-vpc-vpc`
- Security group: `databases-sg`

---

## 9. Frontend — Config React + Vite

### Variables de entorno

**`.env.local`** (desarrollo local, no se commitea):
```bash
VITE_COGNITO_AUTHORITY=https://cognito-idp.us-east-1.amazonaws.com/TU_POOL_ID
VITE_COGNITO_CLIENT_ID=TU_CLIENT_ID
VITE_COGNITO_REDIRECT_URI=http://localhost:5173/callback
VITE_COGNITO_DOMAIN=https://TU_DOMINIO.auth.us-east-1.amazoncognito.com
VITE_API_URL=
```

**`.env.production`** (build para CloudFront):
```bash
VITE_COGNITO_REDIRECT_URI=https://dxxxxx.cloudfront.net/callback
VITE_API_URL=https://XXXXX.execute-api.us-east-1.amazonaws.com
```

### Cognito config

```ts
export const COGNITO_CONFIG = {
  authority: import.meta.env.VITE_COGNITO_AUTHORITY,
  client_id: import.meta.env.VITE_COGNITO_CLIENT_ID,
  redirect_uri: import.meta.env.VITE_COGNITO_REDIRECT_URI,
  response_type: "code",
  scope: "email openid profile",
  cognitoDomain: import.meta.env.VITE_COGNITO_DOMAIN,
};
```

### Proxy Vite (evita CORS en local)

**`vite.config.ts`**:
```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    strictPort: true,
    proxy: {
      '/users': {
        target: 'https://XXXXX.execute-api.us-east-1.amazonaws.com',
        changeOrigin: true,
        secure: true,
      },
      '/stories': {
        target: 'https://XXXXX.execute-api.us-east-1.amazonaws.com',
        changeOrigin: true,
        secure: true,
      },
      '/reading': {
        target: 'https://XXXXX.execute-api.us-east-1.amazonaws.com',
        changeOrigin: true,
        secure: true,
      },
    }
  }
})
```

Con el proxy activo, localmente el browser llama a `/users/health` que Vite redirige internamente a API Gateway. No hay CORS porque el browser cree que habla con `localhost`.

### Llamadas a la API

```ts
const API_URL = import.meta.env.VITE_API_URL || ''

// En local: llama a /users/health → proxy de Vite lo redirige
// En prod:  llama a https://XXXXX.execute-api.../users/health
fetch(`${API_URL}/users/health`, {
  headers: {
    Authorization: `Bearer ${token}`
  }
})
```

---

## 10. Variables de entorno en ECS

Después de crear las bases de datos, agregar los endpoints como variables de entorno en cada task definition.

**ECS → Task definitions → seleccionar task → Create new revision → editar container → Environment variables:**

```
# user-service
DB_URL     = postgres://appuser:password@TU_RDS_ENDPOINT:5432/appdb
REDIS_URL  = redis://TU_REDIS_ENDPOINT:6379
QUEUE_URL  = https://sqs.us-east-1.amazonaws.com/ACCOUNT_ID/user-signup-queue

# story-service
MONGO_URL  = mongodb://appuser:password@TU_DOCDB_ENDPOINT:27017/appdb?tls=true
REDIS_URL  = redis://TU_REDIS_ENDPOINT:6379

# reader-service
REDIS_URL  = redis://TU_REDIS_ENDPOINT:6379
```

Después de guardar cada revisión: **ECS → service → Update service → seleccionar última revisión → Update**.

---

## Checklist Final

```
[ ] VPC con subnet público y privado
[ ] Segundo subnet público (requerido por ALB)
[ ] NAT Gateway regional en subnet público
[ ] S3 Gateway endpoint asociado a route table privada
[ ] 3 security groups: alb-sg, ecs-sg, databases-sg
[ ] S3 bucket + archivos subidos
[ ] CloudFront con OAC, default root object y error pages
[ ] Cognito con callbacks para CloudFront Y localhost
[ ] Lambda post-signup + cola SQS
[ ] ECR repos + imágenes pusheadas
[ ] ECS cluster + 3 task definitions
[ ] ALB con 2 subnets públicos + 3 target groups + reglas de path
[ ] ECS services en subnet PRIVADO únicamente
[ ] API Gateway con authorizer JWT + 3 integraciones + 3 routes
[ ] CORS sin wildcard *, con origins específicos
[ ] RDS, DocumentDB, ElastiCache en subnet privado
[ ] Variables de entorno en task definitions
```

---

## Costo estimado del prototipo

| Servicio | Costo mensual aprox. |
|---------|---------------------|
| NAT Gateway | ~$35 |
| RDS PostgreSQL (t3.micro) | ~$15 |
| DocumentDB (t3.medium) | ~$55 |
| ElastiCache Redis (t3.micro) | ~$12 |
| ECS Fargate (3 servicios) | ~$10 |
| ALB | ~$16 |
| **Total** | **~$143/mes** |

> 💡 Para ahorrar cuando no estás usando el prototipo: detener RDS y DocumentDB desde la consola (se pueden detener hasta 7 días). El NAT Gateway es el más caro — eliminarlo y recrearlo cuando sea necesario ahorra ~$35/mes.