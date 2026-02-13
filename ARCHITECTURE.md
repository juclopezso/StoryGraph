# StoryGraph - Arquitectura de Microservicios y Eventos en la Nube

## 1. Vista General

```
                            ┌─────────────────────────────┐
                            │      CDN (CloudFront)       │
                            │   Frontend SPA (React/Vue)  │
                            └─────────────┬───────────────┘
                                          │
                            ┌─────────────▼───────────────┐
                            │       API Gateway           │
                            │   (AWS API Gateway / Kong)  │
                            │  - Rate limiting             │
                            │  - Autenticación JWT         │
                            │  - Ruteo a microservicios    │
                            └─────────────┬───────────────┘
                                          │
                 ┌────────────────────────┼────────────────────────┐
                 │                        │                        │
    ┌────────────▼──────┐   ┌─────────────▼──────┐   ┌────────────▼──────┐
    │  Auth Service      │   │  Story Service      │   │  Reader Service   │
    │  (Cuentas/Roles)   │   │  (Creación/Edición) │   │  (Lectura/Progr.) │
    └────────┬───────────┘   └──────────┬──────────┘   └────────┬──────────┘
             │                          │                        │
             │               ┌──────────▼──────────┐            │
             │               │  Graph Service       │            │
             │               │  (Motor de Grafo)    │            │
             │               └──────────┬──────────┘            │
             │                          │                        │
    ┌────────▼───────────┐   ┌──────────▼──────────┐   ┌────────▼──────────┐
    │    PostgreSQL       │   │      MongoDB         │   │     Redis         │
    │  (Usuarios/Meta)   │   │  (Nodos/Historias)   │   │  (Cache/Sesiones) │
    └────────────────────┘   └─────────────────────┘   └───────────────────┘
                                          │
                            ┌─────────────▼───────────────┐
                            │      Event Bus              │
                            │  (Amazon EventBridge /      │
                            │   Apache Kafka / SNS+SQS)   │
                            └─────────────┬───────────────┘
                 ┌────────────────────────┼────────────────────────┐
                 │                        │                        │
    ┌────────────▼──────┐   ┌─────────────▼──────┐   ┌────────────▼──────┐
    │ Analytics Service  │   │ Notification Svc    │   │ Search Service    │
    │ (Estadísticas)     │   │ (Emails/Push)       │   │ (Elasticsearch)   │
    └───────────────────┘   └────────────────────┘   └───────────────────┘
```

---

## 2. Microservicios

### 2.1 Auth Service
- **Responsabilidad:** Registro, login, gestión de roles (Creador/Lector), tokens JWT.
- **Base de datos:** PostgreSQL (tabla `users`, `roles`, `sessions`).
- **Tecnología:** Node.js + Express o Python + FastAPI.
- **Endpoints:**
  - `POST /auth/register`
  - `POST /auth/login`
  - `POST /auth/refresh`
  - `GET /auth/profile`
- **Eventos que emite:**
  - `user.registered` — cuando un usuario se registra.
  - `user.role_changed` — cuando cambia de Lector a Creador.

### 2.2 Story Service
- **Responsabilidad:** CRUD de historias, gestión de metadatos, publicación.
- **Base de datos:** PostgreSQL (metadatos: título, autor, fecha, estado) + MongoDB (estructura de nodos y conexiones del grafo).
- **Endpoints:**
  - `POST /stories` — crear historia.
  - `GET /stories/:id` — obtener metadatos.
  - `PUT /stories/:id` — actualizar historia.
  - `POST /stories/:id/publish` — publicar historia.
  - `DELETE /stories/:id`
- **Eventos que emite:**
  - `story.created`
  - `story.published` — dispara indexación en búsqueda y notificaciones.
  - `story.updated`
  - `story.deleted`

### 2.3 Graph Service
- **Responsabilidad:** Motor del grafo narrativo. Gestiona nodos, conexiones (edges) y validación de la estructura (ciclos, nodos huérfanos, caminos sin salida).
- **Base de datos:** MongoDB (documentos de nodos con referencias a sus conexiones).
- **Endpoints:**
  - `POST /stories/:id/nodes` — crear nodo.
  - `PUT /stories/:id/nodes/:nodeId` — editar texto del nodo.
  - `POST /stories/:id/nodes/:nodeId/choices` — agregar elección/conexión.
  - `GET /stories/:id/graph` — obtener grafo completo (para visualizador).
  - `POST /stories/:id/validate` — validar integridad del grafo.
- **Eventos que emite:**
  - `graph.node_created`
  - `graph.structure_changed` — para invalidar caches.

### 2.4 Reader Service
- **Responsabilidad:** Experiencia de lectura, progreso del lector, historial de elecciones, navegación nodo a nodo.
- **Base de datos:** MongoDB (progreso, historial) + Redis (caché del nodo actual).
- **Endpoints:**
  - `GET /read/:storyId/start` — obtener nodo inicial.
  - `GET /read/:storyId/node/:nodeId` — obtener nodo y sus opciones.
  - `POST /read/:storyId/choose` — registrar elección y avanzar.
  - `GET /read/:storyId/progress` — obtener progreso guardado.
  - `GET /read/:storyId/history` — historial de elecciones.
- **Eventos que emite:**
  - `reader.choice_made` — cada elección del lector (para analytics).
  - `reader.story_completed` — cuando el lector llega a un nodo final.
  - `reader.story_started`

### 2.5 Analytics Service
- **Responsabilidad:** Consumir eventos de lectura para generar estadísticas: caminos más populares, tasa de abandono, nodos más visitados.
- **Base de datos:** PostgreSQL o ClickHouse (datos analíticos agregados).
- **Solo consume eventos**, no expone API pública directamente (o expone endpoints de solo lectura para dashboards del creador).
- **Eventos que consume:**
  - `reader.choice_made`
  - `reader.story_completed`
  - `reader.story_started`
  - `story.published`

### 2.6 Notification Service
- **Responsabilidad:** Enviar notificaciones por email o push cuando ocurren eventos relevantes.
- **Eventos que consume:**
  - `story.published` — notificar a seguidores del creador.
  - `user.registered` — email de bienvenida.
- **Integración:** Amazon SES / SendGrid para email, Firebase Cloud Messaging para push.

### 2.7 Search Service
- **Responsabilidad:** Indexación y búsqueda full-text de historias.
- **Base de datos:** Elasticsearch / OpenSearch.
- **Eventos que consume:**
  - `story.published` — indexar historia.
  - `story.updated` — re-indexar.
  - `story.deleted` — eliminar del índice.
- **Endpoints:**
  - `GET /search?q=...&genre=...&author=...`

---

## 3. Arquitectura de Eventos

### 3.1 Flujo de eventos

```
┌──────────────┐    story.published     ┌──────────────────────┐
│ Story Service │──────────────────────►│                      │
└──────────────┘                        │                      │
                                        │    Event Bus         │
┌──────────────┐    reader.choice_made  │  (EventBridge /      │
│Reader Service │──────────────────────►│   Kafka / SNS+SQS)   │
└──────────────┘                        │                      │
                                        │                      │
┌──────────────┐    user.registered     │                      │
│ Auth Service  │──────────────────────►│                      │
└──────────────┘                        └───────┬──┬──┬────────┘
                                                │  │  │
                          ┌─────────────────────┘  │  └──────────────────┐
                          ▼                        ▼                     ▼
                 ┌────────────────┐   ┌─────────────────┐   ┌───────────────────┐
                 │Analytics Service│   │Notification Svc │   │  Search Service   │
                 │                │   │                 │   │                   │
                 │ Consume:       │   │ Consume:        │   │ Consume:          │
                 │ - choice_made  │   │ - published     │   │ - published       │
                 │ - completed    │   │ - registered    │   │ - updated         │
                 │ - started      │   │                 │   │ - deleted         │
                 └────────────────┘   └─────────────────┘   └───────────────────┘
```

### 3.2 Formato de eventos

Todos los eventos siguen una estructura estándar (CloudEvents spec):

```json
{
  "id": "evt-uuid-1234",
  "source": "storygraph/story-service",
  "type": "story.published",
  "time": "2026-02-13T10:30:00Z",
  "data": {
    "storyId": "story-5678",
    "authorId": "user-1234",
    "title": "El laberinto de espejos",
    "nodeCount": 42
  }
}
```

### 3.3 Patrones de eventos aplicados

| Patrón | Uso en StoryGraph |
|--------|-------------------|
| **Event Notification** | `story.published` notifica a Search y Notification que algo ocurrió. |
| **Event-Carried State Transfer** | `reader.choice_made` lleva el `nodeId`, `storyId`, `userId` y `choiceId` para que Analytics no necesite consultar otros servicios. |
| **Event Sourcing** | El historial de elecciones del lector se reconstruye desde la secuencia de eventos `reader.choice_made`. |
| **CQRS** | Story Service escribe en MongoDB. Search Service mantiene su propio índice en Elasticsearch optimizado para consultas de búsqueda. |

---

## 4. Infraestructura en la Nube (AWS)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS Cloud                                  │
│                                                                     │
│  ┌──────────┐     ┌──────────────┐     ┌─────────────────────────┐ │
│  │Route 53  │────►│ CloudFront   │────►│ S3 (Frontend SPA)       │ │
│  │  (DNS)   │     │  (CDN)       │     └─────────────────────────┘ │
│  └──────────┘     └──────┬───────┘                                  │
│                          │                                          │
│                 ┌────────▼────────┐                                  │
│                 │  API Gateway    │                                  │
│                 └────────┬────────┘                                  │
│                          │                                          │
│         ┌────────────────┼────────────────┐                         │
│         │          ECS Fargate /           │                         │
│         │        EKS (Kubernetes)          │                         │
│         │                                  │                         │
│         │  ┌───────┐ ┌───────┐ ┌───────┐  │                         │
│         │  │ Auth  │ │ Story │ │Reader │  │                         │
│         │  │  Svc  │ │  Svc  │ │  Svc  │  │                         │
│         │  └───────┘ └───────┘ └───────┘  │                         │
│         │  ┌───────┐ ┌───────┐ ┌───────┐  │                         │
│         │  │ Graph │ │Notif. │ │Search │  │                         │
│         │  │  Svc  │ │  Svc  │ │  Svc  │  │                         │
│         │  └───────┘ └───────┘ └───────┘  │                         │
│         │  ┌─────────┐                    │                         │
│         │  │Analytics│                    │                         │
│         │  │   Svc   │                    │                         │
│         │  └─────────┘                    │                         │
│         └─────────────────────────────────┘                         │
│                          │                                          │
│              ┌───────────▼────────────┐                              │
│              │  Amazon EventBridge /  │                              │
│              │  SNS + SQS             │                              │
│              └────────────────────────┘                              │
│                                                                     │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────┐            │
│  │ RDS          │  │ DocumentDB /  │  │ ElastiCache    │            │
│  │ (PostgreSQL) │  │ MongoDB Atlas │  │ (Redis)        │            │
│  └──────────────┘  └───────────────┘  └────────────────┘            │
│                                                                     │
│  ┌──────────────────┐  ┌───────────────────┐                        │
│  │ OpenSearch        │  │ CloudWatch        │                        │
│  │ (Búsqueda)       │  │ (Logs/Monitoreo)  │                        │
│  └──────────────────┘  └───────────────────┘                        │
│                                                                     │
│  ┌──────────────────┐  ┌───────────────────┐                        │
│  │ Cognito          │  │ Secrets Manager   │                        │
│  │ (Auth opcional)  │  │ (Credenciales)    │                        │
│  └──────────────────┘  └───────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.1 Mapeo de servicios a infraestructura AWS

| Componente | Servicio AWS | Justificación |
|------------|-------------|---------------|
| Frontend SPA | S3 + CloudFront | Hosting estático con CDN global, baja latencia. |
| API Gateway | Amazon API Gateway | Ruteo, rate limiting, validación JWT integrada. |
| Microservicios | ECS Fargate o EKS | Contenedores sin gestión de servidores (Fargate) o con orquestación completa (EKS). |
| BD Relacional | Amazon RDS (PostgreSQL) | Usuarios, metadatos, analytics. Backups automáticos, réplicas de lectura. |
| BD Documental | Amazon DocumentDB o MongoDB Atlas | Nodos de historias, estructura de grafos, progreso de lectura. |
| Caché | ElastiCache (Redis) | Sesiones, caché de nodos activos, progreso en tiempo real. |
| Bus de Eventos | EventBridge + SQS | Desacoplamiento entre servicios. SQS como cola dead-letter para resiliencia. |
| Búsqueda | Amazon OpenSearch | Búsqueda full-text de historias por título, contenido, género, autor. |
| Monitoreo | CloudWatch + X-Ray | Logs centralizados, métricas, trazas distribuidas entre microservicios. |
| Secretos | Secrets Manager | Rotación automática de credenciales de BD y API keys. |

---

## 5. Flujos clave

### 5.1 Creador publica una historia

```
Creador ──► API Gateway ──► Story Service
                                │
                                ├──► Valida grafo via Graph Service
                                │         └──► MongoDB (verifica integridad)
                                │
                                ├──► Actualiza estado en PostgreSQL (published)
                                │
                                └──► Emite evento "story.published"
                                              │
                                    ┌─────────┼──────────┐
                                    ▼         ▼          ▼
                              Search Svc  Notif. Svc  Analytics
                              (indexa)    (email a     (registra
                                          seguidores)  publicación)
```

### 5.2 Lector avanza en una historia

```
Lector ──► API Gateway ──► Reader Service
                                │
                                ├──► Redis (busca nodo en caché)
                                │     └──► Cache miss? → MongoDB (obtiene nodo)
                                │
                                ├──► Guarda progreso en MongoDB
                                │
                                └──► Emite evento "reader.choice_made"
                                              │
                                              ▼
                                       Analytics Service
                                       (agrega estadística
                                        de caminos)
```

---

## 6. Resiliencia y escalabilidad

| Aspecto | Estrategia |
|---------|-----------|
| **Escalado horizontal** | Cada microservicio escala independientemente. ECS auto-scaling basado en CPU/memoria o métricas custom. |
| **Circuit Breaker** | Si Graph Service no responde, Story Service degrada la funcionalidad (publica sin validación completa). |
| **Dead Letter Queue** | Eventos fallidos van a SQS DLQ para reprocesamiento posterior. |
| **Idempotencia** | Cada evento lleva un `id` único. Los consumidores ignoran eventos duplicados. |
| **Retry con backoff** | Consumidores de eventos reintentan con backoff exponencial antes de enviar a DLQ. |
| **Health checks** | Cada servicio expone `/health`. ECS/EKS reemplaza instancias no saludables. |
| **Réplicas de lectura** | PostgreSQL con read replicas para consultas de analytics. Redis cluster para caché distribuido. |

---

## 7. Resumen de estilos arquitectónicos aplicados

| Estilo | Dónde se aplica |
|--------|----------------|
| **Microservicios** | Cada dominio (Auth, Story, Graph, Reader, Analytics, Search, Notification) es un servicio independiente con su propia base de datos. |
| **Event-Driven** | Los servicios se comunican asincrónicamente a través del bus de eventos. Los servicios reactivos (Analytics, Search, Notification) solo consumen eventos. |
| **CQRS** | Separación de escritura (Story/Graph Service → MongoDB) y lectura optimizada (Search Service → OpenSearch, Analytics → agregaciones). |
| **API Gateway** | Punto de entrada único que gestiona autenticación, ruteo y rate limiting. |
| **Database per Service** | Cada microservicio gestiona su propia base de datos, evitando acoplamiento. |
| **Cloud-Native** | Contenedores, servicios gestionados, auto-scaling, infraestructura como código. |
