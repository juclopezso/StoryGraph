
# Elevator Pitch

## Versión Media                                                                                                                                                                                                                    
  Objetivo: Proporcionar un entorno digital que facilite la creación estructurada de historias ramificadas, la exploración interactiva de narrativas dinámicas, la gestión eficiente de usuarios y contenido, y la construcción de experiencias     personalizadas de lectura.

  Descripción del proyecto: StoryGraph es una plataforma web para crear, publicar y leer historias interactivas no lineales basadas en una estructura narrativa en forma de grafo. Los creadores diseñan relatos donde cada fragmento narrativo se conecta mediante decisiones que el lector selecciona, generando múltiples caminos y desenlaces posibles. El lector no solo consume contenido, sino que participa activamente en la construcción de la historia. La plataforma incluye un editor visual de grafos para creadores (sin necesidad de programación), un sistema de lectura interactiva con seguimiento de progreso, y notificaciones asíncronas entre lectores y autores.

  Prototipo: Un prototipo web funcional que permite crear historias ramificadas con un editor visual y leerlas de forma interactiva, demostrando la arquitectura cloud-native y la comunicación por eventos entre servicios.

  Arquitectura: El sistema sigue una arquitectura de microservicios orientada a eventos. Se compone de una SPA en React, un API Gateway con autenticación JWT, y cuatro microservicios especializados: User Service (Python/Flask), Story Service (Java/Spring Boot), Reader Service (Java/Spring Boot) y Notification Service (Node.js). La capa de datos combina PostgreSQL para datos relacionales, MongoDB para el grafo narrativo, y Redis como caché de lectura. La comunicación asíncrona entre servicios se realiza mediante colas Amazon SQS, desacoplando los productores de eventos (Story y Reading) de los consumidores (Notification). Se aplican patrones como CQRS, Event Sourcing y Database-per-Service.

  Por qué es Cloud Native: La arquitectura fue diseñada con principios cloud-native desde su concepción. Cada microservicio se despliega como contenedor independiente en un clúster EKS (Kubernetes), permitiendo escalabilidad independiente y despliegue autónomo. El desacoplamiento mediante colas SQS habilita elasticidad y resiliencia ante fallos parciales. Se aprovechan servicios gestionados de AWS — Cognito para identidad, RDS para bases relacionales, ElastiCache para Redis, S3 para almacenamiento de objetos, CloudFront como CDN y API Gateway para enrutamiento — eliminando la gestión de infraestructura y permitiendo escalar bajo demanda. El diseño de database-per-service y la comunicación asíncrona por eventos garantizan que los componentes evolucionen de forma aislada sin afectar la totalidad del sistema.


### Ventajas de ser Cloud Native

#### Escalabilidad
- **Escalabilidad independiente por servicio:** Si la lectura de historias crece más que la creación, se escala solo el Reader Service sin tocar los demás.
- **Escalabilidad horizontal automática:** Kubernetes (EKS) permite agregar o reducir réplicas de pods según la demanda.
- **Elasticidad bajo demanda:** Los servicios gestionados (RDS, ElastiCache, SQS) escalan sin intervención manual.

#### Resiliencia y Disponibilidad
- **Tolerancia a fallos parciales:** Si el Notification Service cae, la creación y lectura de historias siguen funcionando normalmente.
- **Desacoplamiento asíncrono:** Las colas SQS actúan como buffer — si un consumidor falla, los mensajes se retienen hasta que se recupere.
- **Health checks y auto-recuperación:** Kubernetes reinicia automáticamente pods que fallen.

#### Agilidad en el Desarrollo
- **Despliegue independiente:** Se puede actualizar el Story Service sin redesplegar el User Service ni el Reader Service.
- **Tecnologías heterogéneas:** Cada equipo elige el stack más adecuado (Flask, Spring Boot, Node.js) sin restricciones.
- **Ciclos de release más rápidos:** Cambios pequeños y aislados reducen el riesgo de cada despliegue.

#### Eficiencia Operativa
- **Cero gestión de infraestructura** en servicios gestionados: Cognito (auth), RDS (PostgreSQL), ElastiCache (Redis), S3 (archivos), CloudFront (CDN).
- **Pago por uso:** Se consume solo lo que se necesita, sin sobredimensionar servidores.
- **Monitoreo y observabilidad** nativa con herramientas del ecosistema AWS/Kubernetes.

#### Rendimiento
- **Caché distribuida con Redis (ElastiCache):** Acceso rápido al progreso de lectura y nodos narrativos sin golpear la base de datos.
- **CDN con CloudFront:** La SPA y assets estáticos se sirven desde edge locations cercanas al usuario.
- **Base de datos especializada por caso de uso:** PostgreSQL para datos relacionales, MongoDB para grafos narrativos, Redis para caché — cada motor optimizado para su propósito.

#### Mantenibilidad
- **Database-per-service:** No hay acoplamiento a nivel de datos entre servicios, facilitando cambios de esquema sin coordinación.
- **Límites claros de responsabilidad:** Cada microservicio tiene un dominio bien definido, reduciendo la complejidad cognitiva.
- **Evolución independiente:** Los componentes pueden cambiar su implementación interna sin afectar al resto del sistema.

## Versión Corta

  Objetivo: Facilitar la creación y lectura de historias interactivas no lineales en una plataforma web accesible para creadores sin conocimientos técnicos.

  Proyecto: StoryGraph es una plataforma web donde los creadores diseñan historias ramificadas como grafos dirigidos y los lectores navegan eligiendo su propio camino. Incluye editor visual, seguimiento de progreso y notificaciones entre autores y lectores.

  Arquitectura: Microservicios + eventos. Cuatro servicios (User, Story, Reader, Notification) con tecnologías heterogéneas (Flask, Spring Boot, Node.js), bases de datos especializadas (PostgreSQL, MongoDB, Redis) y mensajería asíncrona con Amazon SQS. Frontend SPA en React servido desde CloudFront.

  Cloud Native: Contenedores orquestados en EKS (Kubernetes), servicios gestionados de AWS (Cognito, RDS, ElastiCache, S3, CloudFront), escalabilidad independiente por servicio, desacoplamiento asíncrono por eventos y despliegue autónomo — sin infraestructura administrada manualmente.

### Ventajas de ser Cloud Native
- **Escalabilidad:** Escalamiento independiente por servicio y horizontal automático con Kubernetes; servicios gestionados que escalan sin intervención manual.
- **Resiliencia:** Tolerancia a fallos parciales gracias al desacoplamiento asíncrono con SQS y auto-recuperación de pods con health checks.
- **Agilidad:** Despliegue independiente de cada servicio, libertad de elegir tecnologías por equipo y ciclos de release más rápidos.
- **Eficiencia operativa:** Cero gestión de infraestructura en servicios gestionados (Cognito, RDS, ElastiCache, S3, CloudFront) y pago por uso.
- **Rendimiento:** Caché distribuida con Redis, CDN con CloudFront y bases de datos especializadas por caso de uso.
- **Mantenibilidad:** Database-per-service sin acoplamiento de datos, límites claros de responsabilidad y evolución independiente de componentes.