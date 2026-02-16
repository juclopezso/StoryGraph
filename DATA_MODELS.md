# StoryGraph - Modelos de Datos por Servicio

---

## 1. Auth Service (PostgreSQL)

### 1.1 Tabla `credentials`

Almacena las credenciales de acceso de cada usuario.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador único del usuario en todo el sistema |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Correo electrónico (usado como login) |
| `password_hash` | VARCHAR(255) | NOT NULL | Hash de la contraseña (bcrypt/argon2) |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | Indica si la cuenta está activa |
| `is_email_verified` | BOOLEAN | NOT NULL, DEFAULT false | Indica si el email fue verificado |
| `last_login_at` | TIMESTAMP | NULL | Fecha del último inicio de sesión |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:**
- `idx_credentials_email` — UNIQUE sobre `email`

### 1.2 Tabla `roles`

Catálogo de roles del sistema.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador del rol |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | Nombre del rol (Lector, Creador, Administrador) |
| `description` | VARCHAR(255) | NULL | Descripción del rol |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |

**Datos iniciales:**

| id | name | description |
|----|------|-------------|
| 1 | Lector | Puede leer historias, guardar progreso y dejar valoraciones |
| 2 | Creador | Puede crear y publicar historias, además de leer |
| 3 | Administrador | Gestión completa de la plataforma |

### 1.3 Tabla `permissions`

Catálogo de permisos granulares del sistema.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador del permiso |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL | Nombre del permiso (ej: `story.create`, `user.manage`) |
| `description` | VARCHAR(255) | NULL | Descripción del permiso |

**Datos iniciales (ejemplos):**

| id | name | description |
|----|------|-------------|
| 1 | story.read | Leer historias publicadas |
| 2 | story.create | Crear nuevas historias |
| 3 | story.publish | Publicar historias |
| 4 | story.delete_any | Eliminar cualquier historia |
| 5 | user.manage | Gestionar usuarios |
| 6 | role.assign | Asignar roles a usuarios |

### 1.4 Tabla `role_permissions`

Relación muchos a muchos entre roles y permisos.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `role_id` | INTEGER | PK (compuesta), FK → roles.id | Rol |
| `permission_id` | INTEGER | PK (compuesta), FK → permissions.id | Permiso |

### 1.5 Tabla `user_roles`

Asignación de roles a usuarios.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK (compuesta), FK → credentials.id | Usuario |
| `role_id` | INTEGER | PK (compuesta), FK → roles.id | Rol asignado |
| `assigned_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de asignación |
| `assigned_by` | UUID | NULL, FK → credentials.id | Quién asignó el rol (NULL si fue por registro) |

### 1.6 Tabla `refresh_tokens`

Tokens de refresco activos para renovar access tokens.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador del token |
| `user_id` | UUID | NOT NULL, FK → credentials.id | Usuario dueño del token |
| `token_hash` | VARCHAR(255) | NOT NULL | Hash del refresh token |
| `device_info` | VARCHAR(255) | NULL | Información del dispositivo/navegador |
| `expires_at` | TIMESTAMP | NOT NULL | Fecha de expiración |
| `revoked_at` | TIMESTAMP | NULL | Fecha de revocación (NULL si está activo) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |

**Índices:**
- `idx_refresh_tokens_user_id` — sobre `user_id`
- `idx_refresh_tokens_expires_at` — sobre `expires_at` (para limpieza de tokens expirados)

### 1.7 Diagrama de relaciones — Auth Service

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  credentials │       │    roles     │       │ permissions  │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │◄──┐   │ id (PK)      │◄──┐   │ id (PK)      │
│ email        │   │   │ name         │   │   │ name         │
│ password_hash│   │   │ description  │   │   │ description  │
│ is_active    │   │   │ created_at   │   │   └──────┬───────┘
│ is_email_    │   │   └──────┬───────┘   │          │
│  verified    │   │          │           │          │
│ last_login_at│   │   ┌──────▼───────┐   │   ┌──────▼───────┐
│ created_at   │   │   │ user_roles   │   │   │role_permis-  │
│ updated_at   │   │   ├──────────────┤   │   │  sions       │
└──────┬───────┘   ├──►│ user_id (FK) │   ├──►│ role_id (FK) │
       │           │   │ role_id (FK) │   │   │ permission_id│
       │           │   │ assigned_at  │   │   │  (FK)        │
       │           │   │ assigned_by  │   │   └──────────────┘
       │           │   └──────────────┘   │
       │           │                      │
┌──────▼───────┐   │                      │
│refresh_tokens│   │                      │
├──────────────┤   │                      │
│ id (PK)      │   │                      │
│ user_id (FK)─┼───┘                      │
│ token_hash   │                          │
│ device_info  │                          │
│ expires_at   │                          │
│ revoked_at   │                          │
│ created_at   │                          │
└──────────────┘                          │
```

---

## 2. User Service (PostgreSQL + S3)

### 2.1 Tabla `user_profiles`

Información personal y demográfica del usuario.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK, FK → Auth.credentials.id (lógica) | Identificador del usuario (mismo ID del Auth Service) |
| `first_name` | VARCHAR(100) | NOT NULL | Nombre |
| `last_name` | VARCHAR(100) | NOT NULL | Apellido |
| `display_name` | VARCHAR(100) | UNIQUE, NOT NULL | Nombre público visible en la plataforma |
| `bio` | TEXT | NULL | Biografía corta del usuario |
| `birth_date` | DATE | NULL | Fecha de nacimiento |
| `gender` | VARCHAR(20) | NULL | Género |
| `country` | VARCHAR(100) | NULL | País |
| `city` | VARCHAR(100) | NULL | Ciudad |
| `phone` | VARCHAR(20) | NULL | Teléfono de contacto |
| `profile_photo_url` | VARCHAR(500) | NULL | URL de la foto de perfil en S3 |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación del perfil |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

**Índices:**
- `idx_user_profiles_display_name` — UNIQUE sobre `display_name`
- `idx_user_profiles_country_city` — sobre `(country, city)` (para filtros demográficos)

### 2.2 Tabla `user_preferences`

Preferencias de configuración del usuario.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK, FK → user_profiles.user_id | Usuario |
| `language` | VARCHAR(10) | NOT NULL, DEFAULT 'es' | Idioma preferido (es, en, etc.) |
| `theme` | VARCHAR(20) | NOT NULL, DEFAULT 'light' | Tema visual (light, dark) |
| `email_notifications` | BOOLEAN | NOT NULL, DEFAULT true | Recibir notificaciones por email |
| `push_notifications` | BOOLEAN | NOT NULL, DEFAULT true | Recibir notificaciones push |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

### 2.3 Almacenamiento S3

**Bucket:** `storygraph-user-assets`

**Estructura de claves:**
```
profiles/{user_id}/photo.{jpg|png|webp}
profiles/{user_id}/photo_thumb.{jpg|png|webp}
```

### 2.4 Diagrama de relaciones — User Service

```
┌───────────────────┐        ┌───────────────────┐
│   user_profiles   │        │ user_preferences  │
├───────────────────┤        ├───────────────────┤
│ user_id (PK)      │◄──────►│ user_id (PK, FK)  │
│ first_name        │        │ language           │
│ last_name         │        │ theme              │
│ display_name      │        │ email_notifications│
│ bio               │        │ push_notifications │
│ birth_date        │        │ updated_at         │
│ gender            │        └───────────────────┘
│ country           │
│ city              │        ┌───────────────────┐
│ phone             │        │    S3 Bucket      │
│ profile_photo_url─┼───────►│ storygraph-user-  │
│ created_at        │        │ assets            │
│ updated_at        │        │ /profiles/{id}/   │
└───────────────────┘        │   photo.*         │
                             └───────────────────┘
```

---

## 3. Story Service (PostgreSQL + MongoDB)

### 3.1 PostgreSQL — Metadatos

#### 3.1.1 Tabla `stories`

Metadatos de cada historia.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la historia |
| `author_id` | UUID | NOT NULL | ID del usuario creador (referencia lógica a Auth Service) |
| `title` | VARCHAR(255) | NOT NULL | Título de la historia |
| `synopsis` | TEXT | NULL | Sinopsis o descripción breve |
| `genre_id` | INTEGER | NOT NULL, FK → genres.id | Género principal |
| `narrative_style_id` | INTEGER | NULL, FK → narrative_styles.id | Estilo narrativo |
| `estimated_duration` | INTEGER | NULL | Duración estimada en minutos |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | Estado: draft, published, archived |
| `version` | INTEGER | NOT NULL, DEFAULT 1 | Número de versión |
| `node_count` | INTEGER | NOT NULL, DEFAULT 0 | Cantidad de nodos (desnormalizado para consultas rápidas) |
| `cover_image_url` | VARCHAR(500) | NULL | URL de imagen de portada |
| `is_featured` | BOOLEAN | NOT NULL, DEFAULT false | Marcada como destacada por admin |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de última actualización |
| `published_at` | TIMESTAMP | NULL | Fecha de publicación |

**Índices:**
- `idx_stories_author_id` — sobre `author_id`
- `idx_stories_genre_id` — sobre `genre_id`
- `idx_stories_status` — sobre `status`
- `idx_stories_published_at` — sobre `published_at` DESC (para listados recientes)
- `idx_stories_status_published` — sobre `(status, published_at)` (para búsquedas de publicadas)

#### 3.1.2 Tabla `genres`

Catálogo de géneros literarios.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador del género |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL | Nombre del género |
| `description` | VARCHAR(255) | NULL | Descripción |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | Si está activo para selección |

**Datos iniciales:**

| id | name |
|----|------|
| 1 | Fantasía |
| 2 | Ciencia Ficción |
| 3 | Terror |
| 4 | Romance |
| 5 | Misterio |
| 6 | Aventura |
| 7 | Drama |
| 8 | Comedia |

#### 3.1.3 Tabla `narrative_styles`

Catálogo de estilos narrativos.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador del estilo |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL | Nombre del estilo |
| `description` | VARCHAR(255) | NULL | Descripción |

**Datos iniciales:**

| id | name |
|----|------|
| 1 | Primera persona |
| 2 | Segunda persona |
| 3 | Tercera persona |
| 4 | Narrador omnisciente |
| 5 | Flujo de conciencia |

#### 3.1.4 Tabla `tags`

Catálogo de etiquetas para clasificación libre.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador de la etiqueta |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | Nombre de la etiqueta |

#### 3.1.5 Tabla `story_tags`

Relación muchos a muchos entre historias y etiquetas.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `story_id` | UUID | PK (compuesta), FK → stories.id ON DELETE CASCADE | Historia |
| `tag_id` | INTEGER | PK (compuesta), FK → tags.id | Etiqueta |

### 3.2 MongoDB — Historias y Nodos

#### 3.2.1 Colección `stories`

Documento raíz de cada historia con la configuración del grafo.

```json
{
  "_id": ObjectId,
  "story_id": "uuid-de-postgres",
  "root_node_id": "node-uuid-001",
  "settings": {
    "allow_back_navigation": true,
    "show_choice_count": false,
    "auto_save_progress": true
  },
  "created_at": ISODate,
  "updated_at": ISODate
}
```

**Índices:**
- `{ story_id: 1 }` — UNIQUE

#### 3.2.2 Colección `nodes`

Cada nodo narrativo de una historia.

```json
{
  "_id": ObjectId,
  "node_id": "node-uuid-001",
  "story_id": "uuid-de-postgres",
  "title": "El comienzo",
  "content": "Te encuentras en una habitación oscura. Una luz tenue...",
  "is_start": true,
  "is_ending": false,
  "choices": [
    {
      "choice_id": "choice-uuid-001",
      "text": "Abrir la puerta",
      "target_node_id": "node-uuid-002"
    },
    {
      "choice_id": "choice-uuid-002",
      "text": "Examinar la habitación",
      "target_node_id": "node-uuid-003"
    }
  ],
  "position": {
    "x": 250,
    "y": 100
  },
  "metadata": {
    "word_count": 85,
    "estimated_read_time_seconds": 30
  },
  "created_at": ISODate,
  "updated_at": ISODate
}
```

**Índices:**
- `{ story_id: 1, node_id: 1 }` — UNIQUE
- `{ story_id: 1, is_start: 1 }` — para encontrar el nodo inicial rápidamente
- `{ story_id: 1 }` — para obtener todos los nodos de una historia

### 3.3 Diagrama de relaciones — Story Service

```
PostgreSQL                                     MongoDB
─────────────────────────────────              ─────────────────────────────

┌──────────────┐   ┌──────────────┐            ┌─────────────────────────┐
│   genres     │   │ narrative_   │            │  stories (colección)   │
├──────────────┤   │  styles      │            ├─────────────────────────┤
│ id (PK)      │   ├──────────────┤            │ _id                    │
│ name         │   │ id (PK)      │            │ story_id ──────────────┼──┐
│ description  │   │ name         │            │ root_node_id           │  │
│ is_active    │   │ description  │            │ settings {}            │  │
└──────┬───────┘   └──────┬───────┘            │ created_at             │  │
       │                  │                    │ updated_at             │  │
       │                  │                    └─────────────────────────┘  │
┌──────▼──────────────────▼──┐                                             │
│        stories             │                 ┌─────────────────────────┐  │
├────────────────────────────┤                 │  nodes (colección)     │  │
│ id (PK) ───────────────────┼─────────────────┤─────────────────────────┤  │
│ author_id                  │                 │ _id                    │  │
│ title                      │          ┌──────┤ node_id                │  │
│ synopsis                   │          │      │ story_id ──────────────┼──┘
│ genre_id (FK) ─────────────┼──────────┘      │ title                  │
│ narrative_style_id (FK)────┼──────────┘      │ content                │
│ estimated_duration         │                 │ is_start               │
│ status                     │                 │ is_ending              │
│ version                    │                 │ choices []             │
│ node_count                 │                 │   ├ choice_id          │
│ created_at                 │                 │   ├ text               │
│ updated_at                 │                 │   └ target_node_id     │
│ published_at               │                 │ position { x, y }     │
└──────────┬─────────────────┘                 │ metadata {}           │
           │                                   │ created_at             │
    ┌──────▼───────┐   ┌──────────┐            │ updated_at             │
    │ story_tags   │   │  tags    │            └─────────────────────────┘
    ├──────────────┤   ├──────────┤
    │ story_id(FK) ├──►│ id (PK)  │
    │ tag_id (FK)──┼──►│ name     │
    └──────────────┘   └──────────┘
```

---

## 4. Reading Service (PostgreSQL + Redis)

### 4.1 Tabla `reading_sessions`

Registro de cada sesión de lectura de un usuario en una historia.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la sesión de lectura |
| `user_id` | UUID | NOT NULL | ID del lector (referencia lógica a Auth Service) |
| `story_id` | UUID | NOT NULL | ID de la historia (referencia lógica a Story Service) |
| `current_node_id` | VARCHAR(100) | NOT NULL | ID del nodo actual donde se encuentra el lector |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'in_progress' | Estado: in_progress, completed, abandoned |
| `started_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de inicio de lectura |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de última actividad |
| `completed_at` | TIMESTAMP | NULL | Fecha de finalización |

**Restricciones:**
- `UNIQUE (user_id, story_id)` — un usuario solo tiene una sesión activa por historia

**Índices:**
- `idx_reading_sessions_user_id` — sobre `user_id`
- `idx_reading_sessions_user_status` — sobre `(user_id, status)` (para listar "mis historias en progreso")
- `idx_reading_sessions_story_id` — sobre `story_id`

### 4.2 Tabla `choice_history`

Historial de cada elección realizada por el lector.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la elección |
| `session_id` | UUID | NOT NULL, FK → reading_sessions.id ON DELETE CASCADE | Sesión de lectura |
| `node_id` | VARCHAR(100) | NOT NULL | Nodo donde se tomó la decisión |
| `choice_id` | VARCHAR(100) | NOT NULL | ID de la elección seleccionada |
| `target_node_id` | VARCHAR(100) | NOT NULL | Nodo destino al que se avanzó |
| `sequence_number` | INTEGER | NOT NULL | Número de orden de la elección (1, 2, 3...) |
| `chosen_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha y hora de la elección |

**Índices:**
- `idx_choice_history_session_id` — sobre `session_id`
- `idx_choice_history_session_seq` — sobre `(session_id, sequence_number)` (para reconstruir la secuencia)

### 4.3 Tabla `reading_stats`

Estadísticas agregadas de lectura por usuario (desnormalizadas para consultas rápidas).

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK | ID del lector |
| `stories_started` | INTEGER | NOT NULL, DEFAULT 0 | Total de historias iniciadas |
| `stories_completed` | INTEGER | NOT NULL, DEFAULT 0 | Total de historias completadas |
| `stories_abandoned` | INTEGER | NOT NULL, DEFAULT 0 | Total de historias abandonadas |
| `total_choices_made` | INTEGER | NOT NULL, DEFAULT 0 | Total de elecciones realizadas |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de última actualización |

### 4.4 Redis — Caché

**Estrategia de caché:** Se almacenan en Redis los nodos que el lector está leyendo actualmente y los nodos adyacentes (a los que puede navegar) para reducir latencia.

**Patrones de claves:**

```
# Nodo cacheado (contenido completo del nodo)
node:{story_id}:{node_id}           → JSON del nodo (TTL: 1 hora)

# Progreso rápido del lector (para evitar ir a PostgreSQL en cada petición)
progress:{user_id}:{story_id}       → { current_node_id, status } (TTL: 30 min)

# Nodos adyacentes pre-cargados
adjacent:{story_id}:{node_id}       → [node_id_1, node_id_2, ...] (TTL: 1 hora)
```

### 4.5 Diagrama de relaciones — Reading Service

```
PostgreSQL                                     Redis
─────────────────────────────────              ─────────────────────────────

┌──────────────────────┐                       ┌─────────────────────────┐
│  reading_sessions    │                       │  Caché de nodos         │
├──────────────────────┤                       ├─────────────────────────┤
│ id (PK)              │                       │ node:{sid}:{nid}        │
│ user_id              │                       │  → JSON del nodo        │
│ story_id             │                       │                         │
│ current_node_id      │                       │ progress:{uid}:{sid}    │
│ status               │                       │  → { node_id, status }  │
│ started_at           │                       │                         │
│ updated_at           │                       │ adjacent:{sid}:{nid}    │
│ completed_at         │                       │  → [node_ids...]        │
└──────────┬───────────┘                       └─────────────────────────┘
           │
           │ 1:N
           │
┌──────────▼───────────┐
│  choice_history      │
├──────────────────────┤
│ id (PK)              │
│ session_id (FK)      │
│ node_id              │
│ choice_id            │
│ target_node_id       │
│ sequence_number      │
│ chosen_at            │
└──────────────────────┘

┌──────────────────────┐
│  reading_stats       │
├──────────────────────┤
│ user_id (PK)         │
│ stories_started      │
│ stories_completed    │
│ stories_abandoned    │
│ total_choices_made   │
│ updated_at           │
└──────────────────────┘
```

---

## 5. Notification Service (PostgreSQL)

### 5.1 Tabla `subscriptions`

Suscripciones de lectores a creadores (para recibir notificaciones cuando publican).

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la suscripción |
| `subscriber_id` | UUID | NOT NULL | ID del usuario suscriptor (lector) |
| `author_id` | UUID | NOT NULL | ID del creador al que se suscribe |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de suscripción |

**Restricciones:**
- `UNIQUE (subscriber_id, author_id)` — un usuario no puede suscribirse dos veces al mismo creador

**Índices:**
- `idx_subscriptions_author_id` — sobre `author_id` (para buscar suscriptores de un creador)
- `idx_subscriptions_subscriber_id` — sobre `subscriber_id` (para listar suscripciones de un usuario)

### 5.2 Tabla `notifications`

Registro de cada notificación enviada/generada.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la notificación |
| `user_id` | UUID | NOT NULL | ID del usuario destinatario |
| `type` | VARCHAR(50) | NOT NULL | Tipo: story_published, story_completed, welcome |
| `title` | VARCHAR(255) | NOT NULL | Título de la notificación |
| `message` | TEXT | NOT NULL | Contenido del mensaje |
| `data` | JSONB | NULL | Datos adicionales (story_id, author_name, etc.) |
| `channel` | VARCHAR(20) | NOT NULL | Canal: email, push, in_app |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT false | Si fue leída (para notificaciones in_app) |
| `sent_at` | TIMESTAMP | NULL | Fecha de envío efectivo |
| `read_at` | TIMESTAMP | NULL | Fecha en que fue leída |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |

**Índices:**
- `idx_notifications_user_id` — sobre `user_id`
- `idx_notifications_user_read` — sobre `(user_id, is_read)` (para listar no leídas)
- `idx_notifications_created_at` — sobre `created_at` DESC (para ordenar cronológicamente)

### 5.3 Tabla `notification_preferences`

Preferencias de notificación por usuario y tipo de evento.

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK (compuesta) | ID del usuario |
| `notification_type` | VARCHAR(50) | PK (compuesta) | Tipo de notificación |
| `email_enabled` | BOOLEAN | NOT NULL, DEFAULT true | Recibir por email |
| `push_enabled` | BOOLEAN | NOT NULL, DEFAULT true | Recibir por push |
| `in_app_enabled` | BOOLEAN | NOT NULL, DEFAULT true | Recibir in-app |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de actualización |

### 5.4 Diagrama de relaciones — Notification Service

```
┌──────────────────────┐
│   subscriptions      │
├──────────────────────┤
│ id (PK)              │
│ subscriber_id        │
│ author_id            │
│ created_at           │
└──────────────────────┘

┌──────────────────────┐        ┌──────────────────────────┐
│   notifications      │        │ notification_preferences │
├──────────────────────┤        ├──────────────────────────┤
│ id (PK)              │        │ user_id (PK compuesta)   │
│ user_id              │        │ notification_type (PK)   │
│ type                 │        │ email_enabled            │
│ title                │        │ push_enabled             │
│ message              │        │ in_app_enabled           │
│ data (JSONB)         │        │ updated_at               │
│ channel              │        └──────────────────────────┘
│ is_read              │
│ sent_at              │
│ read_at              │
│ created_at           │
└──────────────────────┘
```

---

## 6. Search Service (OpenSearch / Elasticsearch)

### 6.1 Índice `stories`

Índice de búsqueda full-text para historias publicadas.

**Mapping:**

```json
{
  "mappings": {
    "properties": {
      "story_id": { "type": "keyword" },
      "title": {
        "type": "text",
        "analyzer": "spanish",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "synopsis": {
        "type": "text",
        "analyzer": "spanish"
      },
      "author_id": { "type": "keyword" },
      "author_name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "genre": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "narrative_style": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "tags": { "type": "keyword" },
      "node_count": { "type": "integer" },
      "estimated_duration": { "type": "integer" },
      "published_at": { "type": "date" },
      "content_preview": {
        "type": "text",
        "analyzer": "spanish"
      }
    }
  }
}
```

**Documento de ejemplo:**

```json
{
  "story_id": "uuid-5678",
  "title": "El laberinto de espejos",
  "synopsis": "Una aventura interactiva donde cada reflejo esconde un camino diferente...",
  "author_id": "uuid-1234",
  "author_name": "María García",
  "genre": "Fantasía",
  "narrative_style": "Segunda persona",
  "tags": ["aventura", "magia", "decisiones"],
  "node_count": 42,
  "estimated_duration": 25,
  "published_at": "2026-02-13T10:30:00Z",
  "content_preview": "Te encuentras frente a un espejo que refleja un mundo distinto al tuyo..."
}
```

**Configuración del analizador:**
- Analizador `spanish` para soporte de stemming y stopwords en español.
- Campos `.keyword` para filtros exactos y agregaciones (por género, estilo, autor).

---

## 7. Resumen general de almacenamiento

| Servicio | Tecnología | Tablas/Colecciones | Propósito |
|----------|-----------|-------------------|-----------|
| **Auth Service** | PostgreSQL | credentials, roles, permissions, role_permissions, user_roles, refresh_tokens | Autenticación, autorización, tokens |
| **User Service** | PostgreSQL + S3 | user_profiles, user_preferences + bucket S3 | Datos personales, fotos de perfil |
| **Story Service** | PostgreSQL + MongoDB | stories, genres, narrative_styles, tags, story_tags (PG) + stories, nodes (Mongo) | Metadatos, contenido narrativo |
| **Reading Service** | PostgreSQL + Redis | reading_sessions, choice_history, reading_stats + claves de caché | Progreso, historial, caché de lectura |
| **Notification Service** | PostgreSQL | subscriptions, notifications, notification_preferences | Suscripciones, log de notificaciones |
| **Search Service** | OpenSearch | índice stories | Búsqueda full-text de historias |
