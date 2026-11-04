# Architecture

## System Style

The application is a stateful FastAPI backend (served by uvicorn) with native WebSocket support. It is designed around operational bulletin editing rather than around a generic REST API product. The code favors domain-specific resource classes and query classes over a formal ORM. The backend is containerized and runs on Kubernetes; see [Runtime Topology](#runtime-topology).

For visual versions of the architecture and runtime flows, see [diagrams.md](diagrams.md).

## Runtime Topology

- **Image**: a single Python 3.11 image (`Dockerfile`, entrypoint `python WebServices.py`, which starts uvicorn programmatically). It includes native libraries (PostgreSQL client, Cairo) and Playwright/Chromium for rendering.
- **Kubernetes workloads** (namespace `default`):
  - `multirischio-backend` Deployment, 2 replicas, Service `backend-service` (port 5050), `/health` used for readiness and liveness probes, 60 s graceful shutdown;
  - `postgres-db` StatefulSet with Service `postgres-service`;
  - `redis-master` Deployment with Service `redis-master`, used for WebSocket Pub/Sub;
  - four CronJobs (`stato-suolo-giornaliero`, `tabelle-pioggia-giornaliero`, `dati-valanghe-giornaliero`, `modellistica-giornaliero`) built from the same code base (`activations/`), one image per job;
  - rclone Deployments exposing Google Drive and a remote WebDAV share as internal services for the avalanche import.
- **Optional infographic integration**: an external infographic generator web app (separate workload) can be driven headlessly by the backend through Playwright/Chromium (`GET /cfd/generazione_infografica`); the bulletin JSON/XML is exchanged through a shared input folder and the generated maps/tables are copied to a publication folder (see `infografica` in [setup-and-operations.md](setup-and-operations.md#utilsdb_configjson)).
- **Ingress (Traefik)**: `/api` is forwarded to the backend with the prefix stripped, `/ws` is forwarded with the prefix stripped to the WebSocket route, everything else goes to the frontend.
- **Configuration**: Kubernetes Secret `secrets-config` mounted as `utils/db_config.json`, `utils/path_config.json` and `utils/secret_config.json`.
- **Images** come from the private cluster registry, tag `prod`.
- **Observability**: logs go to stdout and are collected by Fluent Bit into Loki and viewed in Grafana (see [setup-and-operations.md](setup-and-operations.md#logging)).

## Main Components

### Application bootstrap

The bootstrap logic lives in `__init__.py`.

Responsibilities:

- create the FastAPI app (with lifespan hooks for graceful shutdown) and a permissive CORS policy
- configure console logging (stdout by default)
- create the shared database connection helper and the async Redis client
- instantiate the resource layer
- manage WebSocket connections per room (`ConnectionManager`) and Redis listeners
- expose the `is_authorized` decorator used by protected routes

### Route layer

The route layer is defined in `WebServices.py`.

Responsibilities:

- define HTTP endpoints
- forward requests to resource classes
- standardize exception handling into HTTP responses
- handle login and SAML callback flow orchestration
- trigger WebSocket notifications (published on Redis) after write operations
- return generated files such as PDFs

The route module acts mostly as an orchestration layer, although some endpoints include operational logic such as imports, copies, and transformation steps.

### Resource layer

The `resources/` package contains the main service classes.

Responsibilities:

- parse request arguments and form data
- apply business logic
- call database query objects
- combine data from multiple sources
- prepare output structures consumed by the client
- invoke specialized utilities such as calculation engines or PDF writers

This layer is where most domain behavior lives. Example domains include:

- weather bulletins
- CFD bulletins
- risk notes
- alert zones
- tendency data
- section levels
- avalanche data
- operational imports
- infographic generation triggers (headless browser automation of an external generator)

### Query layer

The `query_objects/` package provides SQL-driven data access.

Responsibilities:

- search by parameter sets
- create and update records
- delete records
- encapsulate joins and domain-specific filters
- keep entity-specific database logic separate from route handling

This behaves like a lightweight data-access layer rather than a full ORM.

### Object metadata layer

The `objects/` package defines entity metadata and links between resource logic and query classes.

Typical responsibilities:

- entity names
- primary key definitions
- response key naming
- default query bindings

### Utility layer

The `utils/` package contains shared technical helpers.

Notable responsibilities:

- PostgreSQL connection setup
- filesystem and path helpers
- custom exception definitions
- alert calculation logic
- geographic conversion helpers for rendered maps
- PDF/graphics support code

### Output generation

The backend generates rendered outputs rather than only JSON.

Main output types:

- PDF bulletins
- SVG map artifacts
- GeoJSON intermediates
- generated files in output directories under the project workspace

## Data And Control Flow

### Read flow

1. client calls an HTTP endpoint
2. route forwards the request to the matching resource
3. resource parses filters or identifiers
4. query objects retrieve data from PostgreSQL
5. resource shapes the payload
6. route returns JSON or a file response

### Write flow

1. client sends update data to a write endpoint
2. route validates authorization where applicable
3. resource updates the database through query objects
4. resource may apply domain rules or calculations
5. route publishes a message on the Redis channel of the room, which every replica forwards to its WebSocket clients
6. route returns the updated object or operation status

### Calculation flow

Some CFD endpoints compute alert levels by combining:

- meteorological bulletin data
- CFD bulletin state
- risk-zone associations
- rainfall thresholds and tables
- soil-state data
- model outputs
- avalanche or section-level data when relevant

This calculation path is considerably richer than a standard CRUD flow and is one of the key domain-specific features of the backend.

## Real-Time Collaboration

WebSockets are used to keep multiple clients aligned during editing sessions. Socket.IO is no longer used (the code is commented out).

Current pattern:

- a client opens a WebSocket on `/{room_id}` (through the ingress: `/ws/{room_id}`)
- the replica registers the connection locally and subscribes to the Redis channel `room:{room_id}`
- write endpoints publish a JSON message (`{"command": "up_...", "socket_sid": ..., ...}`) on that channel
- every replica receives it from Redis and forwards it to its own connected clients, so editors connected to different pods stay aligned
- messages received from a client are republished on the same channel
- on shutdown, connections are closed with code 1001 so clients can reconnect to another pod

This is useful when multiple operators work on the same bulletin in parallel and is what makes running two or more backend replicas possible.

## Background Scheduling

Imports can run in two ways:

- **Kubernetes CronJobs**: each job runs the import script under `activations/` in its own short-lived pod and logs to stdout. This is the production approach.
- **In-process APScheduler**: when `schedule.status` is `"True"` in `db_config.json`, the backend schedules the same imports itself. It must stay disabled when more than one replica runs, otherwise every pod would run the same import.

Operational implication:

- the backend is not only request-driven; it can also behave as an integration worker on a schedule

## External Integrations

The codebase expects several non-local services.

Integration categories:

- PostgreSQL for primary persistence
- Redis for WebSocket Pub/Sub between replicas
- enterprise directory service and Microsoft Entra ID for authentication
- HTTP or WebDAV endpoints for operational data exchange
- internal file shares for model outputs
- external data providers for soil state and section levels

For public documentation, these integrations should be described by function and sensitivity level, not by live hostnames or real credentials.

## Authentication Topology

The application currently combines:

- token-based protection on most business endpoints through an authorization decorator (`authorization` query parameter, JWT)
- username/password login endpoint returning application token context
- SAML/Entra ID entry and callback endpoints for federated sign-in flow (the callback redirects to the frontend with the JWT)

This mixed mode is functional for current clients but should be treated as an explicit architectural constraint when evolving authentication.

## Security-Sensitive Legacy Surface

A legacy endpoint exists to return remote file-service credentials for internal workflows. Keep this endpoint out of any public deployment and treat it as a removal candidate in hardening plans.

## Folder Responsibilities

### `resources/`

Main domain services. This is the most important package for business behavior.

### `query_objects/`

SQL access layer aligned with domain entities.

### `objects/`

Entity metadata and mapping helpers.

### `utils/`

Infrastructure support, configuration access, calculations, rendering helpers.

### `templates/`

Fragments used to build rendered documents or tables.

### `sql/`

Schema, support functions, and migration-like database assets.

### `scripts/`

One-off or support scripts for data import or maintenance.

### `activations/`

Standalone import jobs (`stato_suolo`, `tabelle_pioggia`, `dati_valanghe`, `modellistica`), each with its own Dockerfile, executed by Kubernetes CronJobs and also importable by the backend scheduler.

### `documents/`

Internal operations material, data samples, deployment notes, and diagrams. Useful for maintainers, but not appropriate to publish blindly.

## Architectural Strengths

- clear domain grouping through resource classes
- explicit separation between route handling and SQL access
- practical support for real-time collaboration
- integrated PDF export pipeline
- strong alignment with operational workflows rather than generic abstractions

## Architectural Constraints

- many environment-specific assumptions are embedded in runtime behavior
- configuration relies on JSON files (injected through a Kubernetes Secret) rather than a dedicated secret manager
- route naming and parameter conventions are highly coupled to the existing frontend
- some behaviors mix service logic, integration logic, and file-generation logic in the same modules
- the token travels in the query string

## Good Public Summary

If you need a short architectural description for GitHub, this wording is accurate:

`The backend is a containerized FastAPI application, deployed on Kubernetes, that manages weather and multi-risk bulletin workflows through domain-specific resource classes, PostgreSQL-backed query objects, WebSocket synchronization over Redis, scheduled data imports, and PDF export services.`