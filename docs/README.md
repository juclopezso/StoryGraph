StoryGraph es una plataforma web para la creación y lectura de historias interactivas no lineales. Permite que los creadores diseñen historias tipo novela visual solo con texto, utilizando una estructura en grafo donde cada "nodo" es un fragmento narrativo que se conecta con otros mediante elecciones del lector. Los lectores avanzan en la historia seleccionando distintas opciones, generando caminos únicos según sus decisiones.

Características Principales:

- Arquitectura de microservicios con comunicación asíncrona vía message broker.
- Dos interfaces frontend: UI Usuario (lectores y creadores) y UI Administración (gestión de la plataforma).
- API Gateway como punto de entrada único para orquestar peticiones a los microservicios.
- Modelo en grafo: Cada historia está compuesta de nodos (fragmentos de texto) conectados entre sí mediante elecciones.
- Editor para creadores: Proporciona un área de texto para escribir cada fragmento y una interfaz para crear opciones de decisión que enlazan con otros nodos.
- Visualizador opcional: Permite que el creador vea el mapa completo de la historia mediante un grafo interactivo.
- Experiencia de lector: Interfaz simple que muestra el texto actual y las opciones, guardando el progreso y el historial de elecciones.
- Separación de roles: Usuarios diferenciados como Creadores, Lectores y Administradores, cada uno con su panel de control y funciones propias.

Componentes del sistema:

- **UI Usuario:** SPA para registro, login, exploración, lectura y creación de historias.
- **UI Administración:** SPA para gestión de usuarios, roles y configuración de la plataforma.
- **API Gateway:** Punto de entrada que recibe peticiones de los frontends y las orquesta hacia los microservicios.
- **Auth Service:** Autenticación y autorización (JWT, roles, permisos).
- **User Service:** Gestión de datos de usuario (perfil, datos demográficos, foto de perfil).
- **Story Service:** Creación, edición y publicación de historias, sus clasificaciones y nodos. Usa PostgreSQL (metadatos) y MongoDB (historias y nodos).
- **Reading Service:** Interacción del lector con las historias, progreso, historial, historias finalizadas. Usa PostgreSQL y Redis (caché).
- **Message Broker:** Mensajería asíncrona que recibe eventos de Story Service y Reading Service.
- **Notification Service:** Notificaciones a lectores y escritores, consume mensajes del broker.
- **Search Service:** Indexación y búsqueda full-text de historias, consume mensajes del broker.

Sugerencia de tecnologías:

- Base de datos relacional: PostgreSQL, por su robustez para manejar usuarios, autenticación, metadatos y progreso de lectura con integridad.
- Base de datos NoSQL: MongoDB, ideal para documentos JSON flexibles que representen las historias y los nodos narrativos.
- Caché: Redis, para reducir latencia en la lectura de nodos.
- Backend API: Node.js con Express o Python con FastAPI; ambas opciones permiten desarrollar APIs REST modernas y eficientes.
- Frontend: React.js o Vue.js, frameworks muy adecuados para crear interfaces web dinámicas con gran interactividad.
- Visualización de grafo: D3.js o Cytoscape.js, librerías especializadas para representar datos como grafos de manera visual.
- Autenticación: JWT (JSON Web Tokens) como estándar para autenticación segura.
- Message Broker: Apache Kafka, RabbitMQ o Amazon SNS+SQS para mensajería asíncrona entre servicios.
- Búsqueda: OpenSearch / Elasticsearch para indexación y búsqueda full-text.
- Despliegue: AWS (ECS Fargate o EKS), API Gateway, S3 + CloudFront para los frontends.

Esta propuesta es modular y escalable, permitiendo aplicar principios sólidos de arquitectura de software y buenas prácticas en el desarrollo.
