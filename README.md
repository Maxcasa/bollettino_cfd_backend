# Bollettino CFD Backend Documentation

![Docs only](https://img.shields.io/badge/repository-documentation--only-0a7ea4)
![Backend](https://img.shields.io/badge/backend-FastAPI%20%2B%20WebSocket-3c7d3e)
![Runtime](https://img.shields.io/badge/runtime-Docker%20%2B%20Kubernetes-326ce5)
![Domain](https://img.shields.io/badge/domain-weather%20%26%20multi--risk%20bulletins-b85c00)
![Status](https://img.shields.io/badge/status-public--safe%20docs-ready)

Standalone documentation for a FastAPI-based backend used to manage weather and multi-risk operational bulletins. The backend is packaged as a Docker image and runs on Kubernetes (K3s) together with PostgreSQL, Redis and the scheduled import CronJobs.

Last aligned with codebase routes, startup flow and Kubernetes manifests: 2026-11-04.

Cluster-level documentation (deploy procedure, kubectl commands, logging stack) lives in the repository-root `docs/` folder. See [setup-and-operations.md](setup-and-operations.md) for the backend-specific view.

This repository is designed for public sharing of architecture and API knowledge without publishing source code, credentials, or internal infrastructure details.

## Why This Repo Exists

This docs bundle gives a complete technical view of the backend with a safe publication boundary.

It covers:

- architecture and component responsibilities
- setup and operational behavior
- endpoint catalog by functional domain
- security and sanitization guidance for publication

It intentionally excludes:

- application source code
- credentials and secrets
- internal addresses and network details
- operational data dumps

## At A Glance

The backend supports:

- authentication and authorization
- weather and CFD bulletin editing workflows
- multi-risk alert-level computation
- real-time collaboration via native WebSocket rooms, fanned out across replicas with Redis Pub/Sub
- PDF bulletin generation and file outputs
- on-demand CFD infographic asset generation, triggered headlessly via Playwright against an external generator app
- scheduled and on-demand external data imports (in-process APScheduler jobs and Kubernetes CronJobs)
- structured console logging collected centrally (stdout, Fluent Bit, Loki, Grafana)

Core stack:

- Python 3.11
- FastAPI served by uvicorn
- PostgreSQL
- Redis (WebSocket Pub/Sub between replicas)
- APScheduler
- Docker image, deployed on Kubernetes (K3s) with Traefik ingress
- Fluent Bit, Loki and Grafana for logs

## Documentation Index

Start here based on your goal:

- Understand the system design: [architecture.md](architecture.md)
- See visual architecture and flow maps: [diagrams.md](diagrams.md)
- Run or operate it (Docker, Kubernetes, logging): [setup-and-operations.md](setup-and-operations.md)
- Explore the route surface quickly: [api-overview.md](api-overview.md)
- See SQL trigger error code mappings (BLCxx and BLIxx): [api-overview.md](api-overview.md#sql-trigger-error-codes)
- Prepare safe publication of code/assets: [security-and-publication-notes.md](security-and-publication-notes.md)

## Suggested Reading Paths

### Path A: New technical reader

1. [architecture.md](architecture.md)
2. [diagrams.md](diagrams.md)
3. [api-overview.md](api-overview.md)
4. [setup-and-operations.md](setup-and-operations.md)

### Path B: Maintainer preparing publication

1. [security-and-publication-notes.md](security-and-publication-notes.md)
2. [architecture.md](architecture.md)
3. [setup-and-operations.md](setup-and-operations.md)

### Path C: Integrator or frontend collaborator

1. [api-overview.md](api-overview.md)
2. [diagrams.md](diagrams.md)
3. [architecture.md](architecture.md)

## Repository Positioning For GitHub

Suggested repository name options:

- `bollettino-cfd-backend-docs`
- `multirisk-bulletin-backend-docs`

Suggested short description:

`Public-safe technical documentation for a FastAPI and WebSocket backend, running on Docker and Kubernetes, that manages weather and multi-risk operational bulletins.`

Suggested topics:

- `fastapi`
- `websocket`
- `kubernetes`
- `postgresql`
- `backend`
- `weather`
- `risk-management`
- `documentation`

## Scope And Caveats

- This repository is descriptive and implementation-aligned, not a formal specification.
- Terminology reflects the production domain, including Italian route and field naming.
- Configuration is JSON-file based (`utils/*.json`); on Kubernetes these files are injected from a Secret, a few settings can be overridden by environment variables (`LOGGING_LEVEL`, `LOG_OUTPUT`).
- Security-sensitive implementation details are intentionally abstracted.
- Some legacy or internal-only endpoints are documented for completeness and marked accordingly.

## Reuse Notice

You can copy this sentence into the top of your GitHub profile README or pinned repository description:

`This repository contains documentation only. Operational source code, credentials, internal infrastructure details, and environment-specific data are intentionally omitted.`

## Next Upgrade Ideas

- add bilingual documentation (Italian and English) with synchronized sections
- add endpoint examples for the most used meteo and CFD workflows
- add client-specific sequence diagrams for the highest-traffic endpoints