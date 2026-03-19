# Diagrams

This page provides quick visual maps of the backend architecture and the most important operational flows.

## System Architecture

```mermaid
flowchart LR
    FE[Web Client Frontend] -->|HTTP and JSON| WS[WebServices.py Routes]
    FE -->|Socket.IO| SO[Socket.IO Server]

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

    WS --> SO
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
    participant Socket as Socket.IO Room
    participant EditorB as Client B

    EditorA->>Route: POST update
    Route->>Resource: Apply domain update
    Resource->>DB: Persist update
    DB-->>Resource: Commit complete
    Resource-->>Route: Updated state
    Route->>Socket: Emit update event
    Socket-->>EditorB: Broadcast event
    Route-->>EditorA: HTTP success response
```

## Scheduled Import Flow

```mermaid
flowchart TD
    CFG[Schedule Config Enabled] --> JOB[APScheduler Trigger]
    JOB --> IMP[Import Function]
    IMP --> EXT[External Service]
    EXT --> IMP
    IMP --> RES[Resource Update Logic]
    RES --> DB[(PostgreSQL)]
    RES --> LOG[Application Log]
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