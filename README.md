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
flowchart LR
    %% Actors
    USR([Usuario])
    OCI([OCI Infrastructure])
    ADM([Administrador])

    %% Components
    C1[C1: Web Application]
    C2[C2: Telegram Bot]
    C3[C3: Load Balancer OCI]
    C4[C4: Task API]
    C5[C5: Auth Service]
    C6[C6: Redis Cache]
    C7[C7: Background Job Scheduler]
    C8[C8: Primary Database]
    C9[C9: Replica Database]
    C10[C10: Health Monitor]

    %% Usuario flows
    USR -->|Accede vía browser| C1
    USR -->|Envía cmd Telegram| C2

    %% Hacia Load Balancer
    C1 -->|HTTPS| C3
    C2 -->|HTTPS| C3

    %% Load Balancer → Task API
    C3 -->|Distribuye tráfico| C4

    %% Task API → servicios
    C4 -->|Valida token| C5
    C4 -->|Cache-Aside| C6
    C4 -->|SQL write/read| C8
    C4 -->|SQL read-only| C9
    C4 -->|Encola tareas| C7

    %% Replicación
    C8 -->|Async replication| C9

    %% OCI flows
    OCI -->|CPU mayor 70% auto-scaling| C3
    OCI -->|Ping periódico| C10

    %% Health Monitor
    C10 -->|Health check| C4
    C10 -->|Nodo unhealthy| C3

    %% Administrador
    ADM -->|Monitorea sistema| C10
```

---

## 3. Component Table

| Actor | Event / Action | Componente | Responsabilidades del Componente |
|---|---|---|---|
| Usuario | Ingresa a la aplicación desde el navegador | C1: Web Application | Mostrar la interfaz al usuario, permitir la visualización de tareas y enviar las solicitudes necesarias al sistema para consultar o actualizar información. |
| Usuario | Gestiona tareas mediante comandos en Telegram | C2: Telegram Bot | Recibir los mensajes enviados por el usuario, interpretar los comandos y devolver respuestas claras y organizadas dentro de Telegram. |
| Load Balancer | Gestiona las solicitudes entrantes del sistema | C3: Load Balancer (OCI) | Distribuir el tráfico entre las instancias disponibles, detectar fallos en los nodos y redirigir las solicitudes para mantener la disponibilidad del sistema. |
| OCI Infrastructure | Detecta incrementos en el uso de recursos | C3: Load Balancer (OCI) | Ajustar automáticamente la cantidad de instancias activas según la demanda del sistema para mantener un rendimiento estable. |
| Task API | Verifica la autenticidad de los usuarios | C5: Auth Service | Validar usuarios y sesiones activas, así como controlar que cada usuario tenga permisos para acceder a determinados recursos. |
| Task API | Procesa operaciones relacionadas con tareas | C4: Task API (App Instances) | Gestionar la creación, consulta, actualización y eliminación de tareas, además de coordinar la comunicación con la caché y la base de datos. |
| Task API | Solicita información de acceso frecuente | C6: Redis Cache | Guardar temporalmente información consultada con frecuencia para acelerar las respuestas y reducir la carga de la base de datos. |
| Task API | Guarda y recupera información persistente | C8: Primary Database (OCI) | Almacenar de forma segura la información principal del sistema, incluyendo usuarios y tareas, asegurando la integridad de los datos. |
| Task API | Realiza consultas de solo lectura | C9: Replica Database (OCI) | Atender consultas de lectura para disminuir la carga de la base de datos principal y mejorar el rendimiento general. |
| Task API | Envía procesos secundarios para ejecución diferida | C7: Background Job Scheduler | Administrar tareas que no requieren ejecución inmediata, como notificaciones o reportes, procesándolas de manera asíncrona. |
| OCI Infrastructure | Supervisa constantemente el estado de las instancias | C10: Health Monitor | Monitorear la salud de las instancias, detectar fallos y notificar al sistema para reemplazar o reiniciar servicios cuando sea necesario. |
| Administrador | Consulta el estado general de la plataforma | C10: Health Monitor | Proporcionar métricas y alertas relacionadas con el rendimiento, disponibilidad y uso de recursos del sistema. |
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
