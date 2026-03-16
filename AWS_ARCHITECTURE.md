## Componentes desplegados — Descripción y justificación arquitectónica

---

### 🌐 Red y Conectividad

**VPC (Virtual Private Cloud)**
Red privada virtual que aísla todos los recursos del proyecto dentro de AWS. Es el contenedor base de toda la infraestructura; sin ella no existe control sobre la topología de red ni sobre qué recursos son públicos o privados.

**Subnet pública**
Subred con acceso directo a internet a través del Internet Gateway. Aquí viven los componentes que necesitan ser alcanzables desde fuera: el ALB y el NAT Gateway. Nunca se ponen aquí los servicios de negocio ni las bases de datos.

**Subnet privada**
Subred sin acceso directo a internet. Aquí viven los contenedores ECS y las bases de datos. Los recursos pueden salir a internet a través del NAT Gateway, pero nadie desde afuera puede iniciar una conexión hacia ellos directamente.

**Segundo subnet público (us-east-1b)**
Subred pública adicional en una segunda Availability Zone. No agrega lógica de negocio, pero el ALB lo requiere obligatoriamente para poder operar; AWS exige que un ALB esté presente en al menos dos AZs diferentes.

**NAT Gateway**
Dispositivo de red administrado que permite a los recursos del subnet privado (contenedores ECS) hacer llamadas salientes a internet —por ejemplo, para hacer pull de imágenes desde ECR— sin que nadie desde internet pueda conectarse directamente a ellos. Es el componente más costoso de la arquitectura (~$35/mes).

**S3 Gateway Endpoint**
Punto de conexión privado entre la VPC y S3 que evita que el tráfico hacia S3 pase por el NAT Gateway. Como los contenedores pueden necesitar leer o escribir en S3, sin este endpoint cada byte costaría dinero extra en el NAT; con el endpoint el tráfico va por la red interna de AWS sin costo adicional.

**Security Group `alb-sg`**
Firewall virtual del Application Load Balancer. Solo permite tráfico HTTP entrante en el puerto 80 desde cualquier IP pública. Todo lo demás está bloqueado.

**Security Group `ecs-sg`**
Firewall virtual de los contenedores ECS. Solo acepta tráfico en el puerto 5000 y únicamente si proviene del `alb-sg`. Esto garantiza que ningún cliente externo pueda hablar directamente con los servicios, todo debe pasar por el ALB.

**Security Group `databases-sg`**
Firewall virtual de las tres bases de datos. Solo acepta conexiones en los puertos de PostgreSQL (5432), MongoDB (27017) y Redis (6379), y únicamente desde el `ecs-sg`. Las bases de datos son completamente inaccesibles desde internet o desde el ALB.

---

### 🖥️ Frontend y CDN

**S3 Bucket**
Almacenamiento de objetos donde se depositan los archivos estáticos del frontend compilado (HTML, JS, CSS). El acceso público directo está bloqueado; solo CloudFront puede leerlo. Actúa como el "disco duro" de la aplicación web.

**CloudFront (CDN)**
Red de distribución de contenido global que sirve el frontend al usuario final. Termina HTTPS, cachea los archivos en edge locations cercanas al usuario y redirige errores 403/404 a `index.html` para que el router de React funcione correctamente en el cliente. Es el único punto de entrada al frontend.

**Origin Access Control (OAC)**
Mecanismo de autenticación que permite a CloudFront leer el bucket S3 sin hacerlo público. La bucket policy solo autoriza peticiones firmadas por esa distribución específica de CloudFront, cerrando cualquier acceso directo al bucket.

---

### 🔐 Autenticación

**Cognito User Pool**
Servicio administrado de identidad que gestiona el registro, login y tokens de los usuarios. Elimina la necesidad de implementar auth desde cero: almacena credenciales, maneja flujos OAuth 2.0 y emite JWT que luego valida API Gateway.

**Cognito Hosted UI**
Interfaz de login alojada por AWS bajo un dominio de Cognito. Permite tener pantallas de registro e inicio de sesión funcionales sin desarrollar UI de auth propia.

**App Client de Cognito**
Configuración dentro del User Pool que define qué aplicaciones pueden autenticarse, cuáles son las URLs de callback permitidas y qué scopes (permisos) se otorgan. Sin él, el frontend no puede iniciar el flujo OAuth.

---

### ⚡ Eventos y Procesamiento Asíncrono

**SQS — Cola `user-signup-queue`**
Cola de mensajes que desacopla el evento de registro de usuario de cualquier procesamiento posterior. Cuando un usuario se registra, el mensaje llega a la cola y puede ser consumido por otros servicios de forma independiente, sin bloquear ni ralentizar el flujo de registro.

**Lambda `post-signup-handler`**
Función serverless que se ejecuta automáticamente cada vez que un usuario completa el registro en Cognito. Su único trabajo es tomar los datos del nuevo usuario y publicar un mensaje en SQS. Al ser serverless, no tiene costo cuando no hay registros y no requiere infraestructura dedicada.

**Cognito Lambda Trigger (Post Confirmation)**
Gancho que conecta Cognito con Lambda. Después de que un usuario confirma su cuenta, Cognito invoca automáticamente la función `post-signup-handler`. Es el pegamento entre el sistema de auth y el pipeline de eventos.

---

### 📦 Contenedores

**ECR — Elastic Container Registry**
Registro privado de imágenes Docker dentro de AWS. Almacena las imágenes de los tres microservicios. Al estar en la misma región y cuenta que ECS, el pull de imágenes es rápido, seguro y sin costo de transferencia adicional.

**ECS Cluster `app-cluster`**
Agrupador lógico de los servicios de contenedores. No es infraestructura en sí mismo, sino el contexto bajo el cual ECS organiza y monitorea todos los servicios y tareas que corren en Fargate.

**ECS Task Definitions (x3)**
Planos de configuración de cada microservicio: qué imagen usar, cuánta CPU y memoria asignar, qué puertos exponer y qué variables de entorno inyectar. Cada revisión queda guardada, lo que permite hacer rollback si un despliegue falla.

**ECS Services (x3) — `user-service`, `story-service`, `reader-service`**
Los servicios son los que realmente corren los contenedores y los mantienen vivos. Cada uno garantiza que siempre haya el número deseado de tareas activas, se registra en el target group del ALB y, si una tarea muere, la reemplaza automáticamente. Corren en subnet privado con IP pública desactivada.

**AWS Fargate**
Motor de cómputo serverless para contenedores. En lugar de aprovisionar y mantener instancias EC2, Fargate gestiona el hardware subyacente de forma invisible. Solo se paga por el tiempo exacto que los contenedores están corriendo.

**IAM Role `ecsTaskExecutionRole`**
Rol que permite a Fargate, en nombre de las tareas, hacer pull de imágenes desde ECR y escribir logs en CloudWatch. Sin él, los contenedores no podrían ni siquiera arrancar.

---

### 🔀 Balanceo de Carga y Enrutamiento

**Application Load Balancer (ALB) `app-alb`**
Balanceador de carga de capa 7 (HTTP) que recibe todo el tráfico de los microservicios desde API Gateway y lo distribuye al contenedor correcto según el path. Es el único punto de entrada a la red de contenedores desde fuera de la VPC.

**Target Groups (x3)**
Listas de destinos (IPs de los contenedores Fargate) hacia los que el ALB puede enrutar tráfico. Cada target group corresponde a un microservicio y ejecuta health checks periódicos para no enviar tráfico a tareas no saludables.

**Listener Rules de path-based routing**
Reglas configuradas en el listener HTTP:80 del ALB que analizan el path de cada request y lo dirigen al target group correcto: `/users/*` → user-service, `/stories/*` → story-service, `/reading/*` → reader-service.

---

### 🚪 API Gateway

**HTTP API `app-api`**
Puerta de entrada pública a todos los microservicios. Expone un endpoint HTTPS único, aplica autenticación JWT en todas las rutas y hace proxy hacia el ALB. Separa la lógica de seguridad y enrutamiento externo de los servicios internos.

**JWT Authorizer `cognito-auth`**
Mecanismo de validación que inspecciona el token `Authorization: Bearer` de cada request, verifica su firma contra el User Pool de Cognito y rechaza con 401 cualquier llamada sin token válido. Ningún microservicio necesita implementar validación de tokens por su cuenta.

**Integraciones HTTP (x3)**
Configuración que le dice a API Gateway a qué URL del ALB debe reenviar cada request, incluyendo el path completo con el `{proxy}` variable para que el microservicio reciba la ruta correcta.

**Routes (x3)**
Definición de los endpoints públicos de la API: método (ANY), path (`/users/{proxy+}`, etc.) y qué integración y authorizer aplicar. Son la tabla de enrutamiento de API Gateway.

**Configuración CORS**
Política que indica al browser qué orígenes (CloudFront y localhost) tienen permiso para hacer llamadas a la API. Sin esto, el browser bloquea todas las respuestas por política de seguridad mismo-origen.

---

### 🗄️ Bases de Datos

**RDS PostgreSQL `app-postgres`**
Base de datos relacional administrada para datos estructurados con relaciones definidas (probablemente usuarios, perfiles, configuraciones). AWS gestiona backups, parches y alta disponibilidad básica. Corre en subnet privado, inaccesible desde internet.

**DocumentDB `app-docdb`**
Base de datos documental compatible con MongoDB para datos semiestructurados o con esquema flexible (probablemente stories/contenido). Al ser administrada, AWS se encarga del clustering, replicación y mantenimiento. Corre en subnet privado.

**ElastiCache Redis `app-redis`**
Caché en memoria de ultra baja latencia usado como capa de caché para reducir carga en las bases de datos principales, manejo de sesiones o datos temporales frecuentemente accedidos. Al ser in-memory, las lecturas son órdenes de magnitud más rápidas que ir a disco. Corre en subnet privado.

---

### ⚙️ Configuración de Servicios

**Variables de entorno en Task Definitions**
Mecanismo para inyectar configuración sensible (endpoints de bases de datos, URLs de colas) en los contenedores en tiempo de ejecución sin hardcodearlos en las imágenes Docker. Cada revisión de task definition puede tener su propio conjunto de variables, lo que facilita cambiar configuración sin recompilar imágenes.

**Vite Proxy (desarrollo local)**
Configuración del servidor de desarrollo que redirige internamente las llamadas a `/users`, `/stories` y `/reading` hacia API Gateway, haciendo que el browser crea que habla con `localhost`. Esto evita errores de CORS durante el desarrollo sin modificar el código de llamadas a la API.