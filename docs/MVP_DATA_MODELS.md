# StoryGraph - Modelos de Datos MVP

Versión simplificada de los modelos de datos, diseñada para operar el sistema end-to-end con las funcionalidades clave: registrarse, crear historias interactivas, leerlas y recibir notificaciones básicas.

> **Referencia completa:** Los modelos de datos completos se encuentran en `DATA_MODELS.md`.

---

## Alcance del MVP

| Servicio | Incluido en MVP | Diferido a versión posterior |
|----------|----------------|------------------------------|
| **Auth Service** | `credentials`, `roles`, `user_roles`, `refresh_tokens` | `permissions`, `role_permissions` — autorización basada solo en rol |
| **User Service** | `user_profiles` (campos esenciales) | `user_preferences`, campos demográficos (`birth_date`, `gender`, `country`, `city`, `phone`), bucket S3 para fotos |
| **Story Service** | `stories` (PG), `genres` (PG), colección `stories` (Mongo), colección `nodes` (Mongo) | `narrative_styles`, `tags`, `story_tags`, `cover_image_url`, `is_featured` |
| **Reading Service** | `reading_sessions`, `choice_history`, 2 patrones Redis | `reading_stats`, patrón Redis de nodos adyacentes |
| **Notification Service** | `notifications` (solo `in_app`), `subscriptions` | `notification_preferences`, canales `email`/`push`, campos `channel`/`sent_at` |
| **Search Service** | — | Diferido completamente al post-MVP |

---

## 1. Auth Service (PostgreSQL)

### 1.1 Tablas

#### `credentials`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador único del usuario en todo el sistema |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Correo electrónico (login) |
| `password_hash` | VARCHAR(255) | NOT NULL | Hash de la contraseña (bcrypt/argon2) |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | Cuenta activa |
| `is_email_verified` | BOOLEAN | NOT NULL, DEFAULT false | Email verificado |
| `last_login_at` | TIMESTAMP | NULL | Último inicio de sesión |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Última actualización |

#### `roles`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador del rol |
| `name` | VARCHAR(50) | UNIQUE, NOT NULL | Nombre del rol |
| `description` | VARCHAR(255) | NULL | Descripción |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |

#### `user_roles`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK (compuesta), FK → credentials.id | Usuario |
| `role_id` | INTEGER | PK (compuesta), FK → roles.id | Rol asignado |
| `assigned_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de asignación |
| `assigned_by` | UUID | NULL, FK → credentials.id | Quién asignó el rol |

#### `refresh_tokens`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador del token |
| `user_id` | UUID | NOT NULL, FK → credentials.id | Usuario dueño |
| `token_hash` | VARCHAR(255) | NOT NULL | Hash del refresh token |
| `device_info` | VARCHAR(255) | NULL | Dispositivo/navegador |
| `expires_at` | TIMESTAMP | NOT NULL | Fecha de expiración |
| `revoked_at` | TIMESTAMP | NULL | Fecha de revocación (NULL = activo) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |

### 1.2 DDL SQL

```sql
-- ===========================================
-- Auth Service — DDL MVP
-- ===========================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Credenciales de acceso
CREATE TABLE credentials (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_email_verified BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMP,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_credentials_email ON credentials (email);

-- Catálogo de roles
CREATE TABLE roles (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(50) UNIQUE NOT NULL,
    description VARCHAR(255),
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Asignación de roles a usuarios
CREATE TABLE user_roles (
    user_id     UUID NOT NULL REFERENCES credentials(id) ON DELETE CASCADE,
    role_id     INTEGER NOT NULL REFERENCES roles(id),
    assigned_at TIMESTAMP NOT NULL DEFAULT NOW(),
    assigned_by UUID REFERENCES credentials(id),
    PRIMARY KEY (user_id, role_id)
);

-- Refresh tokens
CREATE TABLE refresh_tokens (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id     UUID NOT NULL REFERENCES credentials(id) ON DELETE CASCADE,
    token_hash  VARCHAR(255) NOT NULL,
    device_info VARCHAR(255),
    expires_at  TIMESTAMP NOT NULL,
    revoked_at  TIMESTAMP,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens (user_id);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens (expires_at);
```

### 1.3 Seed Data

```sql
-- Roles predefinidos
INSERT INTO roles (id, name, description) VALUES
    (1, 'Lector',        'Puede leer historias, guardar progreso y dejar valoraciones'),
    (2, 'Creador',       'Puede crear y publicar historias, además de leer'),
    (3, 'Administrador', 'Gestión completa de la plataforma');
```

### 1.4 Diagrama

```
┌──────────────┐
│  credentials │
├──────────────┤
│ id (PK)      │◄──┐
│ email        │   │
│ password_hash│   │
│ is_active    │   │
│ is_email_    │   │
│  verified    │   │
│ last_login_at│   │
│ created_at   │   │
│ updated_at   │   │
└──────┬───────┘   │
       │           │
       │    ┌──────▼───────┐       ┌──────────────┐
       │    │ user_roles   │       │    roles     │
       │    ├──────────────┤       ├──────────────┤
       │    │ user_id (FK) ├──────►│ id (PK)      │
       │    │ role_id (FK) │       │ name         │
       │    │ assigned_at  │       │ description  │
       │    │ assigned_by  │       │ created_at   │
       │    └──────────────┘       └──────────────┘
       │
┌──────▼───────┐
│refresh_tokens│
├──────────────┤
│ id (PK)      │
│ user_id (FK) │
│ token_hash   │
│ device_info  │
│ expires_at   │
│ revoked_at   │
│ created_at   │
└──────────────┘
```

> **Diferido:** `permissions` y `role_permissions` — la autorización se implementa en código según el rol (Lector/Creador/Administrador) sin permisos granulares.

---

## 2. User Service (PostgreSQL)

### 2.1 Tabla

#### `user_profiles`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `user_id` | UUID | PK | Mismo ID de Auth.credentials.id (FK lógica) |
| `first_name` | VARCHAR(100) | NOT NULL | Nombre |
| `last_name` | VARCHAR(100) | NOT NULL | Apellido |
| `display_name` | VARCHAR(100) | UNIQUE, NOT NULL | Nombre público en la plataforma |
| `bio` | TEXT | NULL | Biografía corta |
| `profile_photo_url` | VARCHAR(500) | NULL | URL de foto de perfil (URL externa o placeholder en MVP) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Última actualización |

### 2.2 DDL SQL

```sql
-- ===========================================
-- User Service — DDL MVP
-- ===========================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE user_profiles (
    user_id           UUID PRIMARY KEY,
    first_name        VARCHAR(100) NOT NULL,
    last_name         VARCHAR(100) NOT NULL,
    display_name      VARCHAR(100) UNIQUE NOT NULL,
    bio               TEXT,
    profile_photo_url VARCHAR(500),
    created_at        TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_user_profiles_display_name ON user_profiles (display_name);
```

### 2.3 Diagrama

```
┌───────────────────┐
│   user_profiles   │
├───────────────────┤
│ user_id (PK)      │  ← FK lógica a Auth.credentials.id
│ first_name        │
│ last_name         │
│ display_name      │
│ bio               │
│ profile_photo_url │  → URL externa / placeholder
│ created_at        │
│ updated_at        │
└───────────────────┘
```

> **Diferido:** `user_preferences` (se usan defaults en código), campos demográficos (`birth_date`, `gender`, `country`, `city`, `phone`), bucket S3 para fotos de perfil.

---

## 3. Story Service (PostgreSQL + MongoDB)

### 3.1 PostgreSQL — Metadatos

#### Tabla `stories`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la historia |
| `author_id` | UUID | NOT NULL | ID del creador (FK lógica a Auth.credentials.id) |
| `title` | VARCHAR(255) | NOT NULL | Título |
| `synopsis` | TEXT | NULL | Sinopsis |
| `genre_id` | INTEGER | NOT NULL, FK → genres.id | Género principal |
| `estimated_duration` | INTEGER | NULL | Duración estimada en minutos |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | Estado: `draft`, `published`, `archived` |
| `version` | INTEGER | NOT NULL, DEFAULT 1 | Número de versión |
| `node_count` | INTEGER | NOT NULL, DEFAULT 0 | Cantidad de nodos (desnormalizado) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Última actualización |
| `published_at` | TIMESTAMP | NULL | Fecha de publicación |

#### Tabla `genres`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | SERIAL | PK | Identificador del género |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL | Nombre del género |
| `description` | VARCHAR(255) | NULL | Descripción |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | Activo para selección |

### 3.2 DDL SQL (PostgreSQL)

```sql
-- ===========================================
-- Story Service — DDL MVP (PostgreSQL)
-- ===========================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Catálogo de géneros
CREATE TABLE genres (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,
    description VARCHAR(255),
    is_active   BOOLEAN NOT NULL DEFAULT true
);

-- Metadatos de historias
CREATE TABLE stories (
    id                 UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    author_id          UUID NOT NULL,
    title              VARCHAR(255) NOT NULL,
    synopsis           TEXT,
    genre_id           INTEGER NOT NULL REFERENCES genres(id),
    estimated_duration INTEGER,
    status             VARCHAR(20) NOT NULL DEFAULT 'draft',
    version            INTEGER NOT NULL DEFAULT 1,
    node_count         INTEGER NOT NULL DEFAULT 0,
    created_at         TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at       TIMESTAMP
);

CREATE INDEX idx_stories_author_id ON stories (author_id);
CREATE INDEX idx_stories_genre_id ON stories (genre_id);
CREATE INDEX idx_stories_status ON stories (status);
CREATE INDEX idx_stories_published_at ON stories (published_at DESC);
CREATE INDEX idx_stories_status_published ON stories (status, published_at);
```

### 3.3 MongoDB — Schemas JSON

#### Colección `stories`

Documento raíz con configuración del grafo narrativo.

```json
{
  "$jsonSchema": {
    "bsonType": "object",
    "required": ["story_id", "root_node_id", "settings", "created_at", "updated_at"],
    "properties": {
      "_id": {
        "bsonType": "objectId"
      },
      "story_id": {
        "bsonType": "string",
        "description": "UUID de PostgreSQL"
      },
      "root_node_id": {
        "bsonType": "string",
        "description": "UUID del nodo inicial"
      },
      "settings": {
        "bsonType": "object",
        "required": ["allow_back_navigation"],
        "properties": {
          "allow_back_navigation": {
            "bsonType": "bool",
            "description": "Permitir retroceder en la lectura"
          }
        }
      },
      "created_at": {
        "bsonType": "date"
      },
      "updated_at": {
        "bsonType": "date"
      }
    }
  }
}
```

**Índices:**

```javascript
db.stories.createIndex({ "story_id": 1 }, { unique: true });
```

**Documento de ejemplo:**

```json
{
  "_id": "ObjectId()",
  "story_id": "550e8400-e29b-41d4-a716-446655440000",
  "root_node_id": "660e8400-e29b-41d4-a716-446655440001",
  "settings": {
    "allow_back_navigation": true
  },
  "created_at": "2026-02-17T10:00:00Z",
  "updated_at": "2026-02-17T10:00:00Z"
}
```

#### Colección `nodes`

Cada nodo narrativo de una historia.

```json
{
  "$jsonSchema": {
    "bsonType": "object",
    "required": ["node_id", "story_id", "title", "content", "is_start", "is_ending", "choices", "position", "created_at", "updated_at"],
    "properties": {
      "_id": {
        "bsonType": "objectId"
      },
      "node_id": {
        "bsonType": "string",
        "description": "UUID del nodo"
      },
      "story_id": {
        "bsonType": "string",
        "description": "UUID de la historia en PostgreSQL"
      },
      "title": {
        "bsonType": "string",
        "description": "Título del nodo"
      },
      "content": {
        "bsonType": "string",
        "description": "Contenido narrativo del nodo"
      },
      "is_start": {
        "bsonType": "bool",
        "description": "Es el nodo inicial de la historia"
      },
      "is_ending": {
        "bsonType": "bool",
        "description": "Es un nodo final de la historia"
      },
      "choices": {
        "bsonType": "array",
        "description": "Opciones que llevan a otros nodos",
        "items": {
          "bsonType": "object",
          "required": ["choice_id", "text", "target_node_id"],
          "properties": {
            "choice_id": {
              "bsonType": "string",
              "description": "UUID de la elección"
            },
            "text": {
              "bsonType": "string",
              "description": "Texto visible de la opción"
            },
            "target_node_id": {
              "bsonType": "string",
              "description": "UUID del nodo destino"
            }
          }
        }
      },
      "position": {
        "bsonType": "object",
        "required": ["x", "y"],
        "properties": {
          "x": { "bsonType": "number" },
          "y": { "bsonType": "number" }
        }
      },
      "metadata": {
        "bsonType": "object",
        "properties": {
          "word_count": { "bsonType": "int" },
          "estimated_read_time_seconds": { "bsonType": "int" }
        }
      },
      "created_at": {
        "bsonType": "date"
      },
      "updated_at": {
        "bsonType": "date"
      }
    }
  }
}
```

**Índices:**

```javascript
db.nodes.createIndex({ "story_id": 1, "node_id": 1 }, { unique: true });
db.nodes.createIndex({ "story_id": 1, "is_start": 1 });
db.nodes.createIndex({ "story_id": 1 });
```

**Documento de ejemplo:**

```json
{
  "_id": "ObjectId()",
  "node_id": "660e8400-e29b-41d4-a716-446655440001",
  "story_id": "550e8400-e29b-41d4-a716-446655440000",
  "title": "El comienzo",
  "content": "Te encuentras en una habitación oscura. Una luz tenue ilumina dos puertas frente a ti...",
  "is_start": true,
  "is_ending": false,
  "choices": [
    {
      "choice_id": "770e8400-e29b-41d4-a716-446655440001",
      "text": "Abrir la puerta izquierda",
      "target_node_id": "660e8400-e29b-41d4-a716-446655440002"
    },
    {
      "choice_id": "770e8400-e29b-41d4-a716-446655440002",
      "text": "Abrir la puerta derecha",
      "target_node_id": "660e8400-e29b-41d4-a716-446655440003"
    }
  ],
  "position": { "x": 250, "y": 100 },
  "metadata": {
    "word_count": 18,
    "estimated_read_time_seconds": 8
  },
  "created_at": "2026-02-17T10:00:00Z",
  "updated_at": "2026-02-17T10:00:00Z"
}
```

### 3.4 Seed Data (géneros)

```sql
INSERT INTO genres (id, name, description, is_active) VALUES
    (1, 'Fantasía',         'Mundos mágicos y criaturas fantásticas',    true),
    (2, 'Ciencia Ficción',  'Futuros posibles y tecnología avanzada',    true),
    (3, 'Terror',           'Suspenso, miedo y lo sobrenatural',         true),
    (4, 'Romance',          'Relaciones y dramas sentimentales',         true),
    (5, 'Misterio',         'Enigmas, crímenes y detectives',            true),
    (6, 'Aventura',         'Viajes, exploración y acción',              true),
    (7, 'Drama',            'Conflictos humanos y emocionales',          true),
    (8, 'Comedia',          'Humor, sátira y situaciones divertidas',    true);
```

### 3.5 Diagrama

```
PostgreSQL                                     MongoDB
─────────────────────────────────              ─────────────────────────────

┌──────────────┐                               ┌─────────────────────────┐
│   genres     │                               │  stories (colección)   │
├──────────────┤                               ├─────────────────────────┤
│ id (PK)      │                               │ _id                    │
│ name         │                               │ story_id ──────────────┼──┐
│ description  │                               │ root_node_id           │  │
│ is_active    │                               │ settings               │  │
└──────┬───────┘                               │   └ allow_back_nav.    │  │
       │                                       │ created_at             │  │
       │                                       │ updated_at             │  │
┌──────▼───────────────────┐                   └─────────────────────────┘  │
│        stories           │                                               │
├──────────────────────────┤                   ┌─────────────────────────┐  │
│ id (PK) ─────────────────┼───────────────────┤ nodes (colección)      │  │
│ author_id                │                   ├─────────────────────────┤  │
│ title                    │                   │ _id                    │  │
│ synopsis                 │                   │ node_id                │  │
│ genre_id (FK)            │                   │ story_id ──────────────┼──┘
│ estimated_duration       │                   │ title                  │
│ status                   │                   │ content                │
│ version                  │                   │ is_start               │
│ node_count               │                   │ is_ending              │
│ created_at               │                   │ choices []             │
│ updated_at               │                   │   ├ choice_id          │
│ published_at             │                   │   ├ text               │
└──────────────────────────┘                   │   └ target_node_id     │
                                               │ position { x, y }     │
                                               │ metadata {}           │
                                               │ created_at             │
                                               │ updated_at             │
                                               └─────────────────────────┘
```

> **Diferido:** `narrative_styles`, `tags`, `story_tags`, campos `cover_image_url` e `is_featured` de stories.

---

## 4. Reading Service (PostgreSQL + Redis)

### 4.1 Tablas

#### `reading_sessions`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la sesión |
| `user_id` | UUID | NOT NULL | ID del lector (FK lógica a Auth.credentials.id) |
| `story_id` | UUID | NOT NULL | ID de la historia (FK lógica a Story.stories.id) |
| `current_node_id` | VARCHAR(100) | NOT NULL | Nodo actual del lector |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'in_progress' | Estado: `in_progress`, `completed`, `abandoned` |
| `started_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Inicio de lectura |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Última actividad |
| `completed_at` | TIMESTAMP | NULL | Fecha de finalización |

#### `choice_history`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la elección |
| `session_id` | UUID | NOT NULL, FK → reading_sessions.id ON DELETE CASCADE | Sesión de lectura |
| `node_id` | VARCHAR(100) | NOT NULL | Nodo donde se tomó la decisión |
| `choice_id` | VARCHAR(100) | NOT NULL | ID de la elección seleccionada |
| `target_node_id` | VARCHAR(100) | NOT NULL | Nodo destino |
| `sequence_number` | INTEGER | NOT NULL | Orden de la elección (1, 2, 3...) |
| `chosen_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de la elección |

### 4.2 DDL SQL

```sql
-- ===========================================
-- Reading Service — DDL MVP
-- ===========================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Sesiones de lectura
CREATE TABLE reading_sessions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL,
    story_id        UUID NOT NULL,
    current_node_id VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    started_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMP,
    UNIQUE (user_id, story_id)
);

CREATE INDEX idx_reading_sessions_user_id ON reading_sessions (user_id);
CREATE INDEX idx_reading_sessions_user_status ON reading_sessions (user_id, status);
CREATE INDEX idx_reading_sessions_story_id ON reading_sessions (story_id);

-- Historial de elecciones
CREATE TABLE choice_history (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id      UUID NOT NULL REFERENCES reading_sessions(id) ON DELETE CASCADE,
    node_id         VARCHAR(100) NOT NULL,
    choice_id       VARCHAR(100) NOT NULL,
    target_node_id  VARCHAR(100) NOT NULL,
    sequence_number INTEGER NOT NULL,
    chosen_at       TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_choice_history_session_id ON choice_history (session_id);
CREATE INDEX idx_choice_history_session_seq ON choice_history (session_id, sequence_number);
```

### 4.3 Redis — Patrones de claves

En el MVP se usan solo 2 patrones de claves (sin pre-carga de nodos adyacentes):

```
# Nodo cacheado (contenido completo del nodo de MongoDB)
node:{story_id}:{node_id}          → JSON del nodo
                                     TTL: 1 hora (3600s)

# Progreso rápido del lector
progress:{user_id}:{story_id}     → { "current_node_id": "...", "status": "..." }
                                     TTL: 30 minutos (1800s)
```

**Ejemplos:**

```
# Clave de nodo
SET node:550e8400-e29b-41d4-a716-446655440000:660e8400-e29b-41d4-a716-446655440001
    '{"node_id":"660e8400...","title":"El comienzo","content":"Te encuentras...","choices":[...]}'
    EX 3600

# Clave de progreso
SET progress:aab38400-f1c2-4e5d-b123-789012345678:550e8400-e29b-41d4-a716-446655440000
    '{"current_node_id":"660e8400-e29b-41d4-a716-446655440001","status":"in_progress"}'
    EX 1800
```

### 4.4 Diagrama

```
PostgreSQL                                     Redis
─────────────────────────────────              ─────────────────────────────

┌──────────────────────┐                       ┌─────────────────────────┐
│  reading_sessions    │                       │  node:{sid}:{nid}       │
├──────────────────────┤                       │  → JSON del nodo        │
│ id (PK)              │                       │  TTL: 1 hora            │
│ user_id              │                       │                         │
│ story_id             │                       │  progress:{uid}:{sid}   │
│ current_node_id      │                       │  → { node_id, status }  │
│ status               │                       │  TTL: 30 min            │
│ started_at           │                       └─────────────────────────┘
│ updated_at           │
│ completed_at         │
└──────────┬───────────┘
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
```

> **Diferido:** `reading_stats` — las estadísticas se calculan con queries sobre `choice_history` y `reading_sessions` cuando se necesiten. Patrón Redis `adjacent:{story_id}:{node_id}` para pre-carga de nodos adyacentes.

---

## 5. Notification Service (PostgreSQL)

### 5.1 Tablas

#### `subscriptions`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la suscripción |
| `subscriber_id` | UUID | NOT NULL | ID del usuario suscriptor (FK lógica a Auth.credentials.id) |
| `author_id` | UUID | NOT NULL | ID del creador al que se suscribe (FK lógica) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de suscripción |

#### `notifications`

| Columna | Tipo | Restricciones | Descripción |
|---------|------|---------------|-------------|
| `id` | UUID | PK, DEFAULT uuid_generate_v4() | Identificador de la notificación |
| `user_id` | UUID | NOT NULL | ID del usuario destinatario (FK lógica) |
| `type` | VARCHAR(50) | NOT NULL | Tipo: `story_published`, `story_completed`, `welcome` |
| `title` | VARCHAR(255) | NOT NULL | Título de la notificación |
| `message` | TEXT | NOT NULL | Contenido del mensaje |
| `data` | JSONB | NULL | Datos adicionales (story_id, author_name, etc.) |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT false | Si fue leída |
| `read_at` | TIMESTAMP | NULL | Fecha en que fue leída |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Fecha de creación |

### 5.2 DDL SQL

```sql
-- ===========================================
-- Notification Service — DDL MVP
-- ===========================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Suscripciones a creadores
CREATE TABLE subscriptions (
    id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscriber_id UUID NOT NULL,
    author_id     UUID NOT NULL,
    created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (subscriber_id, author_id)
);

CREATE INDEX idx_subscriptions_author_id ON subscriptions (author_id);
CREATE INDEX idx_subscriptions_subscriber_id ON subscriptions (subscriber_id);

-- Notificaciones (solo canal in_app en MVP)
CREATE TABLE notifications (
    id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id    UUID NOT NULL,
    type       VARCHAR(50) NOT NULL,
    title      VARCHAR(255) NOT NULL,
    message    TEXT NOT NULL,
    data       JSONB,
    is_read    BOOLEAN NOT NULL DEFAULT false,
    read_at    TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_id ON notifications (user_id);
CREATE INDEX idx_notifications_user_read ON notifications (user_id, is_read);
CREATE INDEX idx_notifications_created_at ON notifications (created_at DESC);
```

### 5.3 Diagrama

```
┌──────────────────────┐
│   subscriptions      │
├──────────────────────┤
│ id (PK)              │
│ subscriber_id        │  ← FK lógica a Auth.credentials.id
│ author_id            │  ← FK lógica a Auth.credentials.id
│ created_at           │
└──────────────────────┘

┌──────────────────────┐
│   notifications      │
├──────────────────────┤
│ id (PK)              │
│ user_id              │  ← FK lógica a Auth.credentials.id
│ type                 │
│ title                │
│ message              │
│ data (JSONB)         │
│ is_read              │
│ read_at              │
│ created_at           │
└──────────────────────┘
```

> **Diferido:** `notification_preferences`, campos `channel` y `sent_at` en notifications, canales `email` y `push`.

---

## 6. Resumen de almacenamiento MVP

| Servicio | Tecnología | Tablas/Colecciones/Índices | Propósito |
|----------|-----------|---------------------------|-----------|
| **Auth Service** | PostgreSQL | `credentials`, `roles`, `user_roles`, `refresh_tokens` | Autenticación, roles, tokens |
| **User Service** | PostgreSQL | `user_profiles` | Perfil público del usuario |
| **Story Service** | PostgreSQL + MongoDB | `stories`, `genres` (PG) + `stories`, `nodes` (Mongo) | Metadatos e historias narrativas |
| **Reading Service** | PostgreSQL + Redis | `reading_sessions`, `choice_history` + 2 patrones de claves | Progreso, historial, caché de lectura |
| **Notification Service** | PostgreSQL | `subscriptions`, `notifications` | Suscripciones, notificaciones in-app |

**Totales MVP:** 10 tablas PostgreSQL + 2 colecciones MongoDB + 2 patrones Redis.

---

## 7. Eventos del Message Broker (MVP)

Eventos mínimos necesarios para los flujos clave del MVP. Todos siguen el formato CloudEvents.

### 7.1 Eventos emitidos

| Evento | Productor | Datos clave | Consumidores |
|--------|-----------|-------------|-------------|
| `user.registered` | Auth Service | `userId`, `email`, `displayName` | Notification Service (bienvenida) |
| `story.published` | Story Service | `storyId`, `authorId`, `title`, `genre`, `nodeCount` | Notification Service (avisar suscriptores) |
| `reader.story_completed` | Reading Service | `userId`, `storyId`, `authorId`, `completedAt` | Notification Service (avisar al creador) |

### 7.2 Formato de ejemplo

```json
{
  "id": "evt-550e8400-e29b-41d4-a716-446655440099",
  "source": "storygraph/story-service",
  "type": "story.published",
  "time": "2026-02-17T10:30:00Z",
  "data": {
    "storyId": "550e8400-e29b-41d4-a716-446655440000",
    "authorId": "aab38400-f1c2-4e5d-b123-789012345678",
    "title": "El laberinto de espejos",
    "genre": "Fantasía",
    "nodeCount": 42
  }
}
```

### 7.3 Flujos cubiertos por los eventos MVP

```
                Auth Service
                     │
                     │ user.registered
                     ▼
              Notification Svc ──► Envía notificación de bienvenida
                     ▲
                     │
Story Service ───────┤ story.published
                     │
                     │
Reading Service ─────┤ reader.story_completed
                     │
                     ▼
              Notification Svc ──► Notifica al creador / suscriptores
```

> **Diferido del MVP:** Eventos `story.created`, `story.updated`, `story.deleted`, `reader.story_started` y `reader.choice_made` — no tienen consumidores activos en el MVP. Se pueden agregar cuando se implementen el Search Service, analíticas o estadísticas en tiempo real.

---

## 8. Relaciones entre servicios (FKs lógicas)

Dado que cada servicio tiene su propia base de datos, las referencias entre servicios son **lógicas** (mismo UUID, sin FK física). El tipo es siempre `UUID`.

```
Auth Service                    User Service
┌──────────────┐               ┌───────────────┐
│ credentials  │               │ user_profiles  │
│              │               │                │
│ id (UUID) ───┼──────────────►│ user_id (UUID) │
│              │               └───────────────┘
│              │
│              │                Story Service
│              │               ┌──────────────┐
│              ├──────────────►│ author_id     │  (stories.author_id)
│              │               └──────────────┘
│              │
│              │                Reading Service
│              │               ┌────────────────┐
│              ├──────────────►│ user_id        │  (reading_sessions.user_id)
│              │               └────────────────┘
│              │
│              │                Notification Service
│              │               ┌────────────────┐
│              ├──────────────►│ user_id        │  (notifications.user_id)
│              ├──────────────►│ subscriber_id  │  (subscriptions.subscriber_id)
│              ├──────────────►│ author_id      │  (subscriptions.author_id)
│              │               └────────────────┘
└──────────────┘

Story Service                   Reading Service
┌──────────────┐               ┌────────────────┐
│ stories      │               │reading_sessions │
│              │               │                 │
│ id (UUID) ───┼──────────────►│ story_id (UUID) │
│              │               └────────────────┘
│              │
└──────────────┘
```

Todas las FKs lógicas usan `UUID v4` como tipo de dato, garantizando consistencia entre servicios sin acoplamiento directo en la base de datos.
