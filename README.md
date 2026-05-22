# Component-Based Thinking
**Planeación de Sistemas de Software (Gpo 104)** 
Gustavo García Téllez · Juan Pablo Torres · María Guadalupe Soto Acosta · Juan Pablo Gil

---

## 1. Component Identification process and Rationale

### Methodology: Actions/Actors Approach

Nosotros elegimos la metodología Actors/Actions Approach, debido a que identificamos que nuestro sistema de gestión de tareas cuenta con distintos actores, como usuarios, bots e infraestructura en OCI, entre otros. Este método nos permite organizar las funciones de cada elemento, facilitando la identificación de las responsabilidades de cada componente sin ambigüedades.

El proceso siguió estos pasos:

1. **Identificar actores** – primero identificamos todos los elementos que interactúan con el sistema, como el usuario final, el bot de Telegram, el administrador y algunos componentes de infraestructura en OCI, por ejemplo el Load Balancer y el OCI Scheduler.  

2. **Mapear acciones** – después analizamos qué acciones realiza cada actor y cuáles recibe dentro del sistema, para entender mejor cómo se comunican entre sí.  

3. **Agrupar acciones por responsabilidad** – una vez identificadas las acciones, agrupamos las que estaban relacionadas para formar componentes más organizados y fáciles de mantener, procurando que cada uno tuviera una responsabilidad clara.  

4. **Validar con los Quality Attributes** – finalmente verificamos que cada componente ayudara a cumplir con los ASR definidos en el Sprint 0, especialmente en aspectos como rendimiento, disponibilidad y escalabilidad.

Los componentes resultantes son:

| ID | Componente |
|----|-----------|
| C1 | Web Application (Frontend) |
| C2 | Telegram Bot |
| C3 | Load Balancer (OCI) |
| C4 | Task API (App Instances) |
| C5 | Auth Service |
| C6 | Redis Cache |
| C7 | Background Job Scheduler |
| C8 | Primary Database (OCI DB) |
| C9 | Replica Database (OCI DB Read) |
| C10 | Health Monitor |

---
## 2. Flow Diagram – Actions/Actors Approach

```mermaid
flowchart TD
    %% Actors
    USER([👤 Usuario])
    ADMIN([🔧 Administrador])
    OCI_INFRA([☁️ OCI Infrastructure])

    %% Presentation Layer
    WEB[C1: Web Application]
    BOT[C2: Telegram Bot]

    %% Infrastructure
    LB[C3: Load Balancer]
    HEALTH[C10: Health Monitor]

    %% Application Layer
    API[C4: Task API\nApp Instances x3]
    AUTH[C5: Auth Service]
    SCHEDULER[C7: Background Job Scheduler]

    %% Data Layer
    CACHE[C6: Redis Cache]
    DBPRIMARY[C8: Primary Database]
    DBREPLICA[C9: Replica DB - Read Only]

    %% User flows
    USER -->|Accede vía browser| WEB
    USER -->|Envía comandos| BOT
    WEB -->|HTTP Request| LB
    BOT -->|HTTP Request| LB

    %% Load Balancer routes
    LB -->|Distribuye tráfico| API
    LB -->|Redirige si falla nodo| API

    %% Auth flow
    API -->|Valida token| AUTH
    AUTH -->|Verifica credenciales| DBPRIMARY

    %% Task API data flow
    API -->|Consulta datos frecuentes| CACHE
    CACHE -->|Cache miss - lee| DBPRIMARY
    API -->|Escribe datos| DBPRIMARY
    API -->|Lee datos bajo carga| DBREPLICA

    %% DB Replication
    DBPRIMARY -->|Replicación| DBREPLICA

    %% Background Jobs
    API -->|Encola tareas no críticas| SCHEDULER
    SCHEDULER -->|Ejecuta diferido| DBPRIMARY

    %% Health Monitor
    OCI_INFRA -->|Activa health checks| HEALTH
    HEALTH -->|Ping/Echo a instancias| API
    HEALTH -->|Reporta instancias caídas| LB
    HEALTH -->|Reinicia instancias fallidas| OCI_INFRA

    %% Admin
    ADMIN -->|Monitorea métricas| HEALTH
    ADMIN -->|Configura políticas de scaling| LB
```

---

## 3. Component Table

| Actor | Event / Action | Componente | Responsabilidades del Componente |
|---|---|---|---|
| Usuario | Accede a la aplicación vía browser | C1: Web Application | Renderizar la interfaz de usuario; enviar peticiones HTTP al Load Balancer; mostrar listas de tareas, estados y notificaciones |
| Usuario | Envía comandos de tareas por Telegram | C2: Telegram Bot | Recibir mensajes de Telegram; traducir comandos a llamadas HTTP; retornar respuestas formateadas al usuario |
| Load Balancer | Recibe tráfico de C1 y C2 | C3: Load Balancer (OCI) | Distribuir tráfico a la instancia con menor carga; redirigir tráfico ante fallo de nodo; ejecutar health checks periódicos; aplicar auto-scaling según CPU > 70% |
| OCI Infrastructure | Detecta alta carga (CPU > 70%) | C3: Load Balancer (OCI) | Activar políticas de escalado horizontal; lanzar o terminar instancias según demanda |
| Task API | Recibe petición de usuario | C5: Auth Service | Validar tokens de autenticación; gestionar sesiones; verificar permisos de acceso a recursos |
| Task API | Consulta o modifica tareas | C4: Task API (App Instances) | Manejar CRUD de tareas; coordinar acceso a Cache y Base de Datos; encolar trabajos en background bajo alta carga; responder en < 1.5s bajo carga normal, < 3s bajo carga pico |
| Task API | Lee datos frecuentemente consultados | C6: Redis Cache | Almacenar en memoria datos de usuarios y tareas frecuentes; aplicar patrón Cache-Aside; retornar respuestas en < 1s para datos cacheados |
| Task API | Escribe o lee datos persistentes | C8: Primary Database (OCI) | Persistir todos los datos de tareas y usuarios; aplicar particionamiento por user_id; procesar escrituras y lecturas críticas; replicar datos a la réplica |
| Task API | Lee datos bajo carga alta | C9: Replica Database (OCI) | Atender consultas de solo lectura; reducir carga en el Primary DB; mantenerse sincronizada vía replicación |
| Task API | Recibe tarea no crítica en alta carga | C7: Background Job Scheduler | Recibir y encolar tareas no críticas (e.g., notificaciones, reportes); posponer ejecución hasta que carga del sistema sea < 60%; ejecutar procesos diferidos de forma asíncrona |
| OCI Infrastructure | Envía ping periódico a instancias | C10: Health Monitor | Ejecutar health checks cada N segundos; marcar instancias como unhealthy tras 3 fallos consecutivos; notificar al Load Balancer para excluir nodos; solicitar reinicio automático de instancias caídas |
| Administrador | Monitorea el estado del sistema | C10: Health Monitor | Exponer métricas de salud, latencia y uso de recursos; generar alertas ante anomalías |

---

## 4. Technical Partitioning

> Organizado por **capas técnicas**: Presentation, Load Balancing, Application, Caching y Data. Refleja cómo la tecnología está distribuida en la infraestructura.

```mermaid
graph TB
    subgraph PRESENTATION ["🖥️ Presentation Layer"]
        C1[C1: Web Application\nReact / HTML-CSS-JS]
        C2[C2: Telegram Bot\nPython / Telegram API]
    end

    subgraph LOADBALANCER ["⚖️ Load Balancer Layer"]
        C3[C3: Load Balancer\nOCI Load Balancer\nDynamic Routing · Health Checks · Auto-scaling]
        C10[C10: Health Monitor\nOCI Monitoring · Ping/Echo · Circuit Breaker]
    end

    subgraph APPLICATION ["⚙️ Application Layer"]
        C4A[C4a: Task API\nInstance 1]
        C4B[C4b: Task API\nInstance 2]
        C4C[C4c: Task API\nInstance 3]
        C5[C5: Auth Service\nJWT Validation · Session Management]
        C7[C7: Background Job Scheduler\nAsynchronous Queue · Priority Scheduling]
    end

    subgraph CACHING ["⚡ Caching Layer"]
        C6[C6: Redis Cache\nCache-Aside Pattern · In-Memory Store]
    end

    subgraph DATA ["🗄️ Data Layer"]
        C8[C8: Primary Database\nOCI DB · Key-Based Partitioning · Read-Write]
        C9[C9: Replica Database\nOCI DB · Read Only · Replication Target]
    end

    %% Interactions
    C1 -->|HTTPS| C3
    C2 -->|HTTPS| C3

    C3 -->|Routes request\nLeast Load / Response Time| C4A
    C3 -->|Routes request| C4B
    C3 -->|Routes request| C4C
    C10 -->|Health status| C3
    C10 -.->|Monitors| C4A
    C10 -.->|Monitors| C4B
    C10 -.->|Monitors| C4C

    C4A -->|Validates token| C5
    C4B -->|Validates token| C5
    C4C -->|Validates token| C5

    C4A -->|Cache lookup / store| C6
    C4B -->|Cache lookup / store| C6
    C4C -->|Cache lookup / store| C6

    C4A -->|Write / critical read| C8
    C4B -->|Write / critical read| C8
    C4C -->|Write / critical read| C8

    C4A -->|Read-only queries| C9
    C4B -->|Read-only queries| C9
    C4C -->|Read-only queries| C9

    C6 -->|Cache miss fallback| C8

    C4A -->|Enqueue background tasks| C7
    C7 -->|Deferred write| C8

    C8 -->|Replication| C9
    C5 -->|Credential lookup| C8
```

---

## 5. Domain Partitioning

> Organizado por **dominios de negocio**: User Interface, Infrastructure, Task Management, Identity & Access, Data Management y Background Processing. Refleja cómo las responsabilidades funcionales están agrupadas.

```mermaid
graph TB
    subgraph UI_DOMAIN ["🌐 User Interface Domain"]
        C1[C1: Web Application\nTask views · User dashboard · Forms]
        C2[C2: Telegram Bot\nCommand interface · Notifications]
    end

    subgraph INFRA_DOMAIN ["🏗️ Infrastructure & Routing Domain"]
        C3[C3: Load Balancer\nTraffic routing · Failover · Auto-scaling]
        C10[C10: Health Monitor\nAvailability tracking · Failure detection]
    end

    subgraph TASK_DOMAIN ["✅ Task Management Domain"]
        C4A[C4a: Task API – Instance 1\nCreate · Read · Update · Delete tasks]
        C4B[C4b: Task API – Instance 2\nCreate · Read · Update · Delete tasks]
        C4C[C4c: Task API – Instance 3\nCreate · Read · Update · Delete tasks]
    end

    subgraph AUTH_DOMAIN ["🔐 Identity & Access Domain"]
        C5[C5: Auth Service\nAuthentication · Authorization · Session]
    end

    subgraph DATA_DOMAIN ["💾 Data Management Domain"]
        C6[C6: Redis Cache\nFrequently accessed task & user data]
        C8[C8: Primary Database\nSource of truth · Partitioned by user_id]
        C9[C9: Replica Database\nRead scaling · High availability]
    end

    subgraph BG_DOMAIN ["🔄 Background Processing Domain"]
        C7[C7: Job Scheduler\nDeferred tasks · Non-critical processes]
    end

    %% Cross-domain interactions
    C1 -->|User requests tasks| C3
    C2 -->|Bot commands| C3

    C3 -->|Dispatches to Task domain| C4A
    C3 -->|Dispatches to Task domain| C4B
    C3 -->|Dispatches to Task domain| C4C
    C10 -->|Removes unhealthy nodes| C3
    C10 -.->|Checks health of| C4A
    C10 -.->|Checks health of| C4B
    C10 -.->|Checks health of| C4C

    C4A -->|Authenticate user| C5
    C4B -->|Authenticate user| C5
    C4C -->|Authenticate user| C5

    C4A -->|Retrieve / store task data| C6
    C4B -->|Retrieve / store task data| C6
    C4C -->|Retrieve / store task data| C6
    C4A -->|Persist task data| C8
    C4B -->|Persist task data| C8
    C4C -->|Persist task data| C8
    C4A -->|Read task data at scale| C9
    C4B -->|Read task data at scale| C9
    C4C -->|Read task data at scale| C9

    C4A -->|Defer low-priority tasks| C7
    C7 -->|Write deferred results| C8

    C5 -->|Verify credentials| C8
    C6 -->|Fallback on cache miss| C8
    C8 -->|Async replication| C9
```
