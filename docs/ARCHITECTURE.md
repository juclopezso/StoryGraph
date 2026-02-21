# StoryGraph - Arquitectura de Microservicios y Eventos en la Nube

## 1. Vista General

```
        ┌──────────────────┐        ┌───────────────────────┐
        │   UI Usuario     │        │  UI Administración     │
        │  (SPA React/Vue) │        │  (SPA React/Vue)       │
        │  Login, lectura, │        │  Gestión de usuarios,  │
        │  exploración,    │        │  roles, configuración  │
        │  creación        │        │  de la plataforma      │
        └────────┬─────────┘        └───────────┬────────────┘
                 │                               │
                 └──────────────┬────────────────┘
                                │
                   ┌────────────▼────────────┐
                   │       API Gateway       │
                   │  - Rate limiting        │
                   │  - Autenticación JWT    │
                   │  - Ruteo/Orquestación   │
                   │  - Logging              │
                   └────────────┬────────────┘
                                │
        ┌───────────┬───────────┼───────────┬────────────┐
        │           │           │           │            │
   ┌────▼─────┐ ┌──▼──────┐ ┌──▼───────┐ ┌▼──────────┐ │
   │  Auth    │ │  User   │ │  Story   │ │  Reading  │ │
   │ Service  │ │ Service │ │ Service  │ │  Service  │ │
   └────┬─────┘ └──┬──────┘ └──┬───────┘ └┬──────────┘ │
        │          │           │           │            │
   ┌────▼─────┐ ┌──▼──────┐ ┌──▼───────┐ ┌▼──────────┐ │
   │PostgreSQL│ │PostgreSQL│ │PostgreSQL│ │PostgreSQL │ │
   │(Creds,   │ │(Perfil, │ │(Metadatos│ │(Progreso, │ │
   │ roles,   │ │ datos   │ │ género,  │ │ historial)│ │
   │ tokens)  │ │ demogr.)│ │ autor..) │ │     +     │ │
   └──────────┘ │    +    │ │    +     │ │  Redis    │ │
                │   S3    │ │ MongoDB  │ │ (Caché)   │ │
                │(Fotos)  │ │(Historias│ └───────────┘ │
                └─────────┘ │ y nodos) │               │
                            └──────────┘               │
                                 │                     │
                                 └──────────┬──────────┘
                                            │
                               ┌────────────▼────────────┐
                               │    Message Broker       │
                               │  (Kafka / RabbitMQ /    │
                               │   Amazon SNS+SQS)      │
                               └────────────┬────────────┘
                                       ┌────┴─────┐
                                       │          │
                             ┌─────────▼──┐  ┌───▼───────────┐
                             │Notification│  │    Search     │
                             │  Service   │  │    Service    │
                             │            │  └───┬───────────┘
                             │            │      │
                             └──────┬─────┘ ┌───▼───────────┐
                                    │       │  OpenSearch /  │
                             ┌──────▼─────┐ │ Elasticsearch │
                             │ PostgreSQL │ └───────────────┘
                             │(Preferenc. │
                             │ y log de   │
                             │ notific.)  │
                             └────────────┘
```

---

## 2. Componentes Frontend

### 2.1 UI Usuario
- **Responsabilidad:** Interfaz principal para usuarios finales (lectores y creadores).
- **Funcionalidades:**
  - Registro e inicio de sesión.
  - Exploración y búsqueda de historias.
  - Lectura interactiva de historias (navegación nodo a nodo, elecciones).
  - Creación y edición de historias (editor de nodos, conexiones, visualizador de grafo).
  - Perfil de usuario (datos personales, foto de perfil).
  - Panel de progreso y historial de lectura.
- **Tecnología:** React.js o Vue.js (SPA), servida vía S3 + CloudFront.

### 2.2 UI Administración
- **Responsabilidad:** Interfaz para usuarios administradores de la plataforma.
- **Funcionalidades:**
  - Gestión de usuarios (crear, editar, desactivar cuentas).
  - Gestión de roles y permisos (Lector, Creador, Administrador).
  - Moderación de contenido (revisar/eliminar historias reportadas).
  - Dashboard de métricas de la plataforma.
  - Configuración general de la aplicación.
- **Tecnología:** React.js o Vue.js (SPA), servida vía S3 + CloudFront.

---

## 3. API Gateway

- **Responsabilidad:** Punto de entrada único que recibe todas las peticiones de las aplicaciones frontend y las orquesta hacia los microservicios correspondientes.
- **Funcionalidades:**
  - Ruteo de peticiones a los microservicios según la ruta.
  - Validación de tokens JWT (autenticación).
  - Rate limiting y throttling.
  - Logging y trazabilidad de peticiones.
  - Transformación de requests/responses si es necesario.
  - Agregación de respuestas de múltiples microservicios.
- **Tecnología:** AWS API Gateway / Kong / Express Gateway.

### Tabla de ruteo

| Ruta | Microservicio destino |
|------|----------------------|
| `/auth/*` | Auth Service |
| `/users/*` | User Service |
| `/stories/*` | Story Service |
| `/read/*` | Reading Service |
| `/search/*` | Search Service |
| `/admin/*` | User Service / Auth Service (según operación) |

---

## 4. Microservicios

### 4.1 Auth Service (Autenticación y Autorización)
- **Responsabilidad:** Registro, login, gestión de tokens JWT, verificación de permisos y roles (Lector, Creador, Administrador).
- **Base de datos:** PostgreSQL (tablas `credentials`, `roles`, `permissions`, `refresh_tokens`).
- **Tecnología:** Node.js + Express o Python + FastAPI.
- **Endpoints:**
  - `POST /auth/register` — registro de nuevo usuario.
  - `POST /auth/login` — inicio de sesión, retorna access + refresh token.
  - `POST /auth/refresh` — renovar access token.
  - `POST /auth/logout` — invalidar tokens.
  - `GET /auth/verify` — verificar validez de un token (usado internamente por el API Gateway).
- **Eventos que emite:**
  - `user.registered` — cuando un usuario se registra exitosamente.

### 4.2 User Service (Gestión de Usuarios)
- **Responsabilidad:** Gestión de toda la información relacionada a los usuarios: datos demográficos, foto de perfil, preferencias, etc.
- **Base de datos:** PostgreSQL (tablas `user_profiles`, `user_preferences`) + S3 (almacenamiento de fotos de perfil y otros archivos del usuario).
- **Tecnología:** Node.js + Express o Python + FastAPI.
- **Endpoints:**
  - `GET /users/:id` — obtener perfil de usuario.
  - `PUT /users/:id` — actualizar datos del perfil.
  - `PUT /users/:id/photo` — subir/actualizar foto de perfil.
  - `GET /users/:id/photo` — obtener foto de perfil.
  - `DELETE /users/:id` — eliminar cuenta de usuario.
  - `GET /users` — listar usuarios (admin).
  - `PUT /users/:id/role` — cambiar rol de un usuario (admin).
- **Eventos que consume:**
  - `user.registered` — crea el perfil inicial del usuario tras el registro.

### 4.3 Story Service (Gestión de Historias y Nodos)
- **Responsabilidad:** CRUD de historias, gestión de clasificaciones (género, estilo narrativo), gestión de los nodos narrativos que componen cada historia, sus conexiones y validación de la estructura del grafo.
- **Base de datos:**
  - **PostgreSQL:** Metadatos de historias (género, autor, estilo narrativo, duración estimada, fecha de creación, versión, estado de publicación, clasificación).
  - **MongoDB:** Historias y sus nodos (contenido de texto, elecciones/conexiones entre nodos, estructura del grafo).
- **Tecnología:** Node.js + Express o Python + FastAPI.
- **Endpoints:**
  - `POST /stories` — crear historia.
  - `GET /stories/:id` — obtener metadatos de una historia.
  - `PUT /stories/:id` — actualizar metadatos.
  - `POST /stories/:id/publish` — publicar historia.
  - `DELETE /stories/:id` — eliminar historia.
  - `GET /stories/:id/nodes` — obtener todos los nodos de una historia.
  - `POST /stories/:id/nodes` — crear nodo.
  - `PUT /stories/:id/nodes/:nodeId` — editar nodo.
  - `DELETE /stories/:id/nodes/:nodeId` — eliminar nodo.
  - `POST /stories/:id/nodes/:nodeId/choices` — agregar elección/conexión a otro nodo.
  - `GET /stories/:id/graph` — obtener grafo completo (para visualizador).
  - `POST /stories/:id/validate` — validar integridad del grafo (ciclos, nodos huérfanos, caminos sin salida).
  - `GET /stories/genres` — listar géneros disponibles.
  - `GET /stories/styles` — listar estilos narrativos.
- **Eventos que emite al broker:**
  - `story.created` — historia creada.
  - `story.published` — historia publicada (dispara indexación y notificaciones).
  - `story.updated` — historia o sus nodos actualizados.
  - `story.deleted` — historia eliminada.

### 4.4 Reading Service (Interacción de Lectura)
- **Responsabilidad:** Gestión de la interacción del usuario con las historias: progreso de lectura, historias finalizadas, historial de elecciones, navegación nodo a nodo.
- **Base de datos:**
  - **PostgreSQL:** Progreso de lectura por usuario/historia, historias finalizadas, historial de elecciones, estadísticas de lectura del usuario.
  - **Redis:** Caché del nodo actual y nodos adyacentes para lectura rápida.
- **Tecnología:** Node.js + Express o Python + FastAPI.
- **Endpoints:**
  - `GET /read/:storyId/start` — iniciar lectura, obtener nodo inicial.
  - `GET /read/:storyId/node/:nodeId` — obtener nodo y sus opciones.
  - `POST /read/:storyId/choose` — registrar elección y avanzar al siguiente nodo.
  - `GET /read/:storyId/progress` — obtener progreso guardado.
  - `GET /read/:storyId/history` — historial de elecciones del lector.
  - `GET /read/my-stories` — listar historias en progreso y finalizadas del usuario.
  - `DELETE /read/:storyId/progress` — reiniciar progreso de una historia.
- **Eventos que emite al broker:**
  - `reader.story_started` — el lector inicia una historia.
  - `reader.choice_made` — cada elección del lector.
  - `reader.story_completed` — el lector llega a un nodo final.

### 4.5 Notification Service (Notificaciones)
- **Responsabilidad:** Envío de notificaciones a lectores y escritores cuando ocurren eventos relevantes. Ejemplos: cuando un escritor publica una historia, se notifica a los lectores; cuando un lector termina una historia, se notifica al escritor.
- **Base de datos:** PostgreSQL (tablas `notification_log`, `notification_preferences`, `subscriptions`).
- **Integración:** Amazon SES / SendGrid (email), Firebase Cloud Messaging (push).
- **Eventos que consume del broker:**
  - `story.published` — notificar a lectores/seguidores del creador.
  - `reader.story_completed` — notificar al escritor que un lector completó su historia.
  - `user.registered` — enviar email de bienvenida.
- **Endpoints (internos o para configuración de preferencias):**
  - `GET /notifications` — listar notificaciones del usuario.
  - `PUT /notifications/preferences` — configurar preferencias de notificación.

### 4.6 Search Service (Búsqueda e Indexación)
- **Responsabilidad:** Indexación y búsqueda full-text de historias para los usuarios.
- **Base de datos:** OpenSearch / Elasticsearch.
- **Eventos que consume del broker:**
  - `story.published` — indexar nueva historia.
  - `story.updated` — re-indexar historia.
  - `story.deleted` — eliminar del índice.
- **Endpoints:**
  - `GET /search?q=...&genre=...&author=...&style=...`

---

## 5. Message Broker

- **Responsabilidad:** Middleware de mensajería asíncrona que desacopla los servicios productores (Story Service, Reading Service) de los servicios consumidores (Notification Service, Search Service).
- **Tecnología:** Apache Kafka / RabbitMQ / Amazon SNS+SQS.
- **Productores:**
  - **Story Service** — emite eventos de ciclo de vida de historias (`story.created`, `story.published`, `story.updated`, `story.deleted`).
  - **Reading Service** — emite eventos de interacción del lector (`reader.story_started`, `reader.choice_made`, `reader.story_completed`).
- **Consumidores:**
  - **Notification Service** — consume eventos para disparar notificaciones.
  - **Search Service** — consume eventos para mantener el índice de búsqueda actualizado.

### 5.1 Flujo de eventos

```
┌──────────────┐    story.published      ┌──────────────────────┐
│ Story Service │───────────────────────►│                      │
│              │    story.updated        │                      │
│              │───────────────────────►│                      │
│              │    story.deleted        │    Message Broker    │
│              │───────────────────────►│                      │
└──────────────┘                        │  (Kafka / RabbitMQ / │
                                        │   SNS+SQS)          │
┌──────────────┐    reader.choice_made  │                      │
│Reading Service│───────────────────────►│                      │
│              │    reader.completed    │                      │
│              │───────────────────────►│                      │
│              │    reader.started      │                      │
│              │───────────────────────►│                      │
└──────────────┘                        └───────┬──────┬───────┘
                                                │      │
                                   ┌────────────┘      └─────────────┐
                                   ▼                                 ▼
                         ┌─────────────────┐             ┌───────────────────┐
                         │Notification Svc │             │  Search Service   │
                         │                 │             │                   │
                         │ Consume:        │             │ Consume:          │
                         │ - published     │             │ - published       │
                         │ - completed     │             │ - updated         │
                         │ - registered    │             │ - deleted         │
                         └─────────────────┘             └───────────────────┘
```

### 5.2 Formato de eventos

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
    "genre": "fantasía",
    "nodeCount": 42
  }
}
```

### 5.3 Patrones de eventos aplicados

| Patrón | Uso en StoryGraph |
|--------|-------------------|
| **Event Notification** | `story.published` notifica a Search y Notification que algo ocurrió. |
| **Event-Carried State Transfer** | `reader.choice_made` lleva `nodeId`, `storyId`, `userId` y `choiceId` para que los consumidores no necesiten consultar otros servicios. |
| **Event Sourcing** | El historial de elecciones del lector se puede reconstruir desde la secuencia de eventos `reader.choice_made`. |
| **CQRS** | Story Service escribe en MongoDB/PostgreSQL. Search Service mantiene su propio índice en OpenSearch optimizado para consultas de búsqueda. |

---

## 6. Infraestructura en la Nube (AWS)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS Cloud                                  │
│                                                                     │
│  ┌──────────┐     ┌──────────────┐     ┌─────────────────────────┐ │
│  │Route 53  │────►│ CloudFront   │────►│ S3 (UI Usuario +       │ │
│  │  (DNS)   │     │  (CDN)       │     │     UI Administración)  │ │
│  └──────────┘     └──────┬───────┘                                 │
│                          │                                          │
│                 ┌────────▼────────┐                                  │
│                 │  API Gateway    │                                  │
│                 └────────┬────────┘                                  │
│                          │                                          │
│         ┌────────────────┼──────────────────┐                       │
│         │          ECS Fargate /             │                       │
│         │        EKS (Kubernetes)            │                       │
│         │                                    │                       │
│         │  ┌───────┐ ┌───────┐ ┌──────────┐ │                       │
│         │  │ Auth  │ │ User  │ │  Story   │ │                       │
│         │  │  Svc  │ │  Svc  │ │   Svc    │ │                       │
│         │  └───────┘ └───────┘ └──────────┘ │                       │
│         │  ┌──────────┐ ┌───────┐ ┌───────┐ │                       │
│         │  │ Reading  │ │Notif. │ │Search │ │                       │
│         │  │   Svc    │ │  Svc  │ │  Svc  │ │                       │
│         │  └──────────┘ └───────┘ └───────┘ │                       │
│         └────────────────────────────────────┘                       │
│                          │                                          │
│              ┌───────────▼────────────┐                              │
│              │  Amazon SNS + SQS /   │                              │
│              │  Amazon MSK (Kafka)   │                              │
│              │  (Message Broker)     │                              │
│              └────────────────────────┘                              │
│                                                                     │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────┐            │
│  │ RDS          │  │ DocumentDB /  │  │ ElastiCache    │            │
│  │ (PostgreSQL) │  │ MongoDB Atlas │  │ (Redis)        │            │
│  │ Auth, User,  │  │ Story nodes   │  │ Reading cache  │            │
│  │ Story meta,  │  │ e historias   │  │                │            │
│  │ Reading,     │  │               │  │                │            │
│  │ Notification │  │               │  │                │            │
│  └──────────────┘  └───────────────┘  └────────────────┘            │
│                                                                     │
│  ┌──────────────────┐  ┌───────────────────┐  ┌──────────────────┐  │
│  │ OpenSearch        │  │ S3               │  │ CloudWatch       │  │
│  │ (Búsqueda)       │  │ (Fotos perfil,   │  │ (Logs/Monitoreo) │  │
│  └──────────────────┘  │  archivos)       │  └──────────────────┘  │
│                        └───────────────────┘                        │
│  ┌──────────────────┐  ┌───────────────────┐                        │
│  │ Cognito          │  │ Secrets Manager   │                        │
│  │ (Auth opcional)  │  │ (Credenciales)    │                        │
│  └──────────────────┘  └───────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.1 Mapeo de servicios a infraestructura AWS

| Componente | Servicio AWS | Justificación |
|------------|-------------|---------------|
| UI Usuario | S3 + CloudFront | Hosting estático con CDN global, baja latencia. |
| UI Administración | S3 + CloudFront | Hosting estático separado, mismo patrón de despliegue. |
| API Gateway | Amazon API Gateway | Ruteo, rate limiting, validación JWT integrada. |
| Microservicios | ECS Fargate o EKS | Contenedores sin gestión de servidores (Fargate) o con orquestación completa (EKS). |
| BD Relacional | Amazon RDS (PostgreSQL) | Auth, User, Story (metadatos), Reading, Notification. Backups automáticos, réplicas de lectura. |
| BD Documental | Amazon DocumentDB o MongoDB Atlas | Historias y sus nodos (estructura de grafo narrativo). |
| Caché | ElastiCache (Redis) | Caché de nodos para lectura rápida en Reading Service. |
| Message Broker | Amazon SNS+SQS o Amazon MSK (Kafka) | Desacoplamiento entre servicios productores y consumidores. SQS como cola dead-letter para resiliencia. |
| Búsqueda | Amazon OpenSearch | Búsqueda full-text de historias por título, contenido, género, autor, estilo. |
| Archivos | S3 | Almacenamiento de fotos de perfil y otros archivos de usuario. |
| Monitoreo | CloudWatch + X-Ray | Logs centralizados, métricas, trazas distribuidas entre microservicios. |
| Secretos | Secrets Manager | Rotación automática de credenciales de BD y API keys. |

---

## 7. Flujos clave

### 7.1 Creador publica una historia

```
Creador ──► UI Usuario ──► API Gateway ──► Story Service
                                                │
                                                ├──► Valida grafo internamente
                                                │         └──► MongoDB (verifica integridad)
                                                │
                                                ├──► Actualiza estado en PostgreSQL (published)
                                                │
                                                └──► Emite evento "story.published" al Broker
                                                              │
                                                    ┌─────────┴──────────┐
                                                    ▼                    ▼
                                              Search Svc          Notif. Svc
                                              (indexa historia)   (notifica a
                                                                   lectores)
```

### 7.2 Lector avanza en una historia

```
Lector ──► UI Usuario ──► API Gateway ──► Reading Service
                                                │
                                                ├──► Redis (busca nodo en caché)
                                                │     └──► Cache miss? → consulta Story Service
                                                │
                                                ├──► Guarda progreso en PostgreSQL
                                                │
                                                └──► Emite evento "reader.choice_made" al Broker
                                                              │
                                                              ▼
                                                       Notification Svc
                                                       (si completó historia,
                                                        notifica al escritor)
```

### 7.3 Administrador gestiona usuarios

```
Admin ──► UI Administración ──► API Gateway ──► User Service
                                                     │
                                                     ├──► PostgreSQL (CRUD de perfiles)
                                                     │
                                                     └──► Auth Service (cambio de roles/permisos)
```

---

## 8. Resiliencia y escalabilidad

| Aspecto | Estrategia |
|---------|-----------|
| **Escalado horizontal** | Cada microservicio escala independientemente. ECS auto-scaling basado en CPU/memoria o métricas custom. |
| **Circuit Breaker** | Si un microservicio no responde, el API Gateway degrada la funcionalidad informando al cliente. |
| **Dead Letter Queue** | Mensajes fallidos del broker van a SQS DLQ para reprocesamiento posterior. |
| **Idempotencia** | Cada evento lleva un `id` único. Los consumidores ignoran eventos duplicados. |
| **Retry con backoff** | Consumidores de eventos reintentan con backoff exponencial antes de enviar a DLQ. |
| **Health checks** | Cada servicio expone `/health`. ECS/EKS reemplaza instancias no saludables. |
| **Réplicas de lectura** | PostgreSQL con read replicas para consultas intensivas. Redis cluster para caché distribuido. |
| **Caché** | Reading Service utiliza Redis para reducir latencia en lectura de nodos, evitando consultas repetidas a Story Service. |

---

## 9. Resumen de componentes y bases de datos

| Componente | Tipo | Base de datos |
|------------|------|---------------|
| **UI Usuario** | Frontend SPA | — |
| **UI Administración** | Frontend SPA | — |
| **API Gateway** | Infraestructura | — |
| **Auth Service** | Microservicio | PostgreSQL |
| **User Service** | Microservicio | PostgreSQL + S3 |
| **Story Service** | Microservicio | PostgreSQL + MongoDB |
| **Reading Service** | Microservicio | PostgreSQL + Redis |
| **Message Broker** | Infraestructura | — |
| **Notification Service** | Microservicio | PostgreSQL |
| **Search Service** | Microservicio | OpenSearch / Elasticsearch |

---

## 10. Resumen de estilos arquitectónicos aplicados

| Estilo | Dónde se aplica |
|--------|----------------|
| **Microservicios** | Cada dominio (Auth, User, Story, Reading, Notification, Search) es un servicio independiente con su propia base de datos. |
| **Event-Driven** | Story Service y Reading Service publican eventos al Message Broker. Notification y Search consumen eventos asincrónicamente. |
| **CQRS** | Separación de escritura (Story Service → PostgreSQL/MongoDB) y lectura optimizada (Search Service → OpenSearch). |
| **API Gateway** | Punto de entrada único que gestiona autenticación, ruteo, rate limiting y orquestación. |
| **Database per Service** | Cada microservicio gestiona su propia base de datos, evitando acoplamiento directo. |
| **Cloud-Native** | Contenedores, servicios gestionados, auto-scaling, infraestructura como código. |
