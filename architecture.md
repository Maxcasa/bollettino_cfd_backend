# Architecture

## System Style

The application is a stateful Flask backend with Socket.IO support. It is designed around operational bulletin editing rather than around a generic REST API product. The code favors domain-specific resource classes and query classes over a formal ORM.

For visual versions of the architecture and runtime flows, see [diagrams.md](diagrams.md).

## Main Components

### Application bootstrap

The bootstrap logic lives in `__init__.py`.

Responsibilities:

- create the Flask app instance
- initialize Flask-SocketIO with permissive CORS
- configure file logging
- create the shared database connection helper
- instantiate the resource layer
- expose the `is_authorized` decorator used by protected routes

### Route layer

The route layer is defined in `WebServices.py`.

Responsibilities:

- define HTTP endpoints
- forward requests to resource classes
- standardize exception handling into HTTP responses
- trigger Socket.IO notifications after write operations
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
5. route emits a Socket.IO event to synchronize other clients
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

Socket.IO is used to keep multiple clients aligned during editing sessions.

Current pattern:

- a client connects to the Socket.IO server
- the server places it in a configured room
- write operations emit named events to that room
- the initiating client can be excluded from some updates using the socket session identifier

This is useful when multiple operators work on the same bulletin in parallel.

## Background Scheduling

The application can enable APScheduler background jobs based on configuration.

Observed use case:

- scheduled import of soil-state data from an external source

Operational implication:

- the backend is not only request-driven; it can also behave as an integration worker on a schedule

## External Integrations

The codebase expects several non-local services.

Integration categories:

- PostgreSQL for primary persistence
- enterprise directory service for authentication
- HTTP or WebDAV endpoints for operational data exchange
- internal file shares for model outputs
- external data providers for soil state and section levels

For public documentation, these integrations should be described by function and sensitivity level, not by live hostnames or real credentials.

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
- configuration appears to rely on JSON files rather than modern secret management
- route naming and parameter conventions are highly coupled to the existing frontend
- some behaviors mix service logic, integration logic, and file-generation logic in the same modules

## Good Public Summary

If you need a short architectural description for GitHub, this wording is accurate:

`The backend is a Flask and Socket.IO application that manages weather and multi-risk bulletin workflows through domain-specific resource classes, PostgreSQL-backed query objects, scheduled data imports, and PDF export services.`