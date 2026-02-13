StoryGraph es una plataforma web para la creación y lectura de historias interactivas no lineales. Permite que los creadores diseñen historias tipo novela visual solo con texto, utilizando una estructura en grafo donde cada "nodo" es un fragmento narrativo que se conecta con otros mediante elecciones del lector. Los lectores avanzan en la historia seleccionando distintas opciones, generando caminos únicos según sus decisiones.

Características Principales:

- Arquitectura híbrida: base de datos relacional para cuentas de usuario y metadatos, y base de datos NoSQL para la estructura flexible de los nodos y rutas de la historia.
- Modelo en grafo: Cada historia está compuesta de nodos (fragmentos de texto) conectados entre sí mediante elecciones.
- Editor para creadores: Proporciona un área de texto para escribir cada fragmento y una interfaz para crear opciones de decisión que enlazan con otros nodos.
- Visualizador opcional: Permite que el creador vea el mapa completo de la historia mediante un grafo interactivo.
- Experiencia de lector: Interfaz simple que muestra el texto actual y las opciones, guardando el progreso y el historial de elecciones.
- Separación de roles: Usuarios diferenciados como Creadores y Lectores, cada uno con su panel de control y funciones propias.

Sugerencia de tecnologías:

- Base de datos relacional: PostgreSQL, por su robustez para manejar usuarios, autenticación y metadatos con integridad.
- Base de datos NoSQL: MongoDB o Firebase Firestore, ideales para documentos JSON flexibles que representen los nodos de la historia.
- Backend API: Node.js con Express o Python con FastAPI; ambas opciones permiten desarrollar APIs REST modernas y eficientes, y se integran bien con datos en JSON.
- Frontend: React.js o Vue.js, frameworks muy adecuados para crear interfaces web dinámicas con gran interactividad, como editores y lectores de historias.
- Visualización de grafo: D3.js o Cytoscape.js, librerías especializadas para representar datos como grafos de manera visual.
- Autenticación: JWT (JSON Web Tokens) como estándar para autenticación segura o servicios como Auth0 o Firebase Auth para simplificar la gestión de cuentas y roles.
- Despliegue: Vercel para el frontend, Heroku o AWS para el backend, y MongoDB Atlas o Firebase para la base de datos, opciones fáciles de usar y escalar.

Esta propuesta es modular y escalable, permitiendo aplicar principios sólidos de arquitectura de software y buenas prácticas en el desarrollo.
