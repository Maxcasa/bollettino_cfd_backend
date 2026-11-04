# Diagrams

This page provides quick visual maps of the backend architecture and the most important operational flows.

## System Architecture

```mermaid
flowchart LR
    FE[Web Client Frontend] -->|HTTP and JSON| WS[WebServices.py Routes]
    FE -->|WebSocket| SO[WebSocket endpoint]

    WS --> AUTH[Authorization Decorator]
    AUTH --> RES[resources package]
    RES --> QRY[query_objects package]
    QRY --> DB[(PostgreSQL)]

    RES --> UTIL[utils package]
    UTIL --> CALC[Alert Calculation Logic]
    UTIL --> RENDER[PDF and SVG Rendering]

    RES --> OUT[(Output Files)]
    OUT --> PDF[Generated PDF Bulletins]
    OUT --> SVG[GeoJSON and SVG Artifacts]

    WS --> REDIS[(Redis Pub/Sub)]
    REDIS --> SO
    SO --> FE

    SCHED[APScheduler Jobs] --> RES
    EXT[External Providers] --> RES
```

## Request Processing Flow

```mermaid
sequenceDiagram
    participant Client
    participant Route as WebServices Route
    participant Auth as Auth Check
    participant Resource as Resource Class
    participant Query as Query Object
    participant DB as PostgreSQL

    Client->>Route: HTTP request
    Route->>Auth: Validate token and permissions
    Auth-->>Route: Authorized
    Route->>Resource: Call domain method
    Resource->>Query: Build and execute query
    Query->>DB: SQL operation
    DB-->>Query: Data rows
    Query-->>Resource: Domain data
    Resource-->>Route: Response payload
    Route-->>Client: JSON or file response
```

## Write And Realtime Synchronization Flow

```mermaid
sequenceDiagram
    participant EditorA as Client A
    participant Route as Write Endpoint
    participant Resource as Resource Class
    participant DB as PostgreSQL
    participant Socket as Redis channel + WebSocket
    participant EditorB as Client B

    EditorA->>Route: POST update
    Route->>Resource: Apply domain update
    Resource->>DB: Persist update
    DB-->>Resource: Commit complete
    Resource-->>Route: Updated state
    Route->>Socket: Publish update message
    Socket-->>EditorB: Forward to clients on every replica
    Route-->>EditorA: HTTP success response
```

## Deployment And Logging Flow

```mermaid
flowchart LR
    USER[Browser] --> TRAEFIK[Traefik Ingress]
    TRAEFIK -->|/| FEPOD[Frontend pods - nginx]
    TRAEFIK -->|/api and /ws| BEPOD[Backend pods - FastAPI]
    BEPOD --> PG[(PostgreSQL StatefulSet)]
    BEPOD --> RD[(Redis)]
    CRON[CronJobs - imports] --> PG

    BEPOD -->|stdout| FB[Fluent Bit DaemonSet]
    PG -->|stdout| FB
    CRON -->|stdout| FB
    FB --> LOKI[(Loki)]
    LOKI --> GRAF[Grafana]
```

## Scheduled Import Flow

```mermaid
flowchart TD
    CFG[CronJob schedule or APScheduler enabled] --> JOB[Import job]
    JOB --> IMP[Import Function]
    IMP --> EXT[External Service]
    EXT --> IMP
    IMP --> RES[Resource Update Logic]
    RES --> DB[(PostgreSQL)]
    RES --> LOG[stdout log]
```

## Infographic Generation Flow

```mermaid
sequenceDiagram
    participant Client
    participant Route as GET /cfd/generazione_infografica
    participant Resource as InfograficaResource
    participant Gen as External infographic generator
    participant FS as Shared folders
    participant Socket as Redis channel + WebSocket

    Client->>Route: authorization, id_bollettino_cfd
    Route->>Resource: genera_infografica(params)
    Resource->>FS: Copy bulletin JSON/XML into input_dir
    Resource->>Gen: Open app headlessly (Playwright) and run "Genera Tutto"
    Gen->>FS: Write maps/tables/text into output_dir
    Resource->>FS: Copy generated files to copy_dir
    Resource-->>Route: files + warnings
    Route->>Socket: Publish up_infografica
    Route-->>Client: status ok + messaggio
```

## Alert Level Calculation Inputs

```mermaid
flowchart LR
    METEO[Meteo Bulletin Data] --> CALC[Alert Evaluation Engine]
    CFD[CFD Bulletin State] --> CALC
    RISK[Risk Zone Associations] --> CALC
    RAIN[Rainfall Tables and Thresholds] --> CALC
    SOIL[Soil State Data] --> CALC
    MODEL[Model Outputs] --> CALC
    AVA[Avalanche and Section Data] --> CALC
    CALC --> RESULT[Alert Levels by Risk and Zone]
```

## How To Use These Diagrams

- Use the system architecture diagram for repository landing pages and presentations.
- Use the request and write-flow diagrams when explaining backend behavior to frontend developers.
- Use the scheduled import and calculation diagrams when onboarding operations and forecasting teams.