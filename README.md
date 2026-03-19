# Bollettino CFD Backend Documentation

![Docs only](https://img.shields.io/badge/repository-documentation--only-0a7ea4)
![Backend](https://img.shields.io/badge/backend-Flask%20%2B%20Socket.IO-3c7d3e)
![Domain](https://img.shields.io/badge/domain-weather%20%26%20multi--risk%20bulletins-b85c00)
![Status](https://img.shields.io/badge/status-public--safe%20docs-ready)

Standalone documentation for a Flask-based backend used to manage weather and multi-risk operational bulletins.

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
- real-time collaboration via Socket.IO events
- PDF bulletin generation and file outputs
- scheduled and on-demand external data imports

Core stack:

- Python
- Flask
- Flask-SocketIO
- PostgreSQL
- APScheduler

## Documentation Index

Start here based on your goal:

- Understand the system design: [architecture.md](architecture.md)
- See visual architecture and flow maps: [diagrams.md](diagrams.md)
- Run or operate it internally: [setup-and-operations.md](setup-and-operations.md)
- Explore the route surface quickly: [api-overview.md](api-overview.md)
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

`Public-safe technical documentation for a Flask and Socket.IO backend that manages weather and multi-risk operational bulletins.`

Suggested topics:

- `flask`
- `socketio`
- `postgresql`
- `backend`
- `weather`
- `risk-management`
- `documentation`

## Scope And Caveats

- This repository is descriptive and implementation-aligned, not a formal specification.
- Terminology reflects the production domain, including Italian route and field naming.
- Configuration in the original codebase appears JSON-file based and environment-specific.
- Security-sensitive implementation details are intentionally abstracted.

## Reuse Notice

You can copy this sentence into the top of your GitHub profile README or pinned repository description:

`This repository contains documentation only. Operational source code, credentials, internal infrastructure details, and environment-specific data are intentionally omitted.`

## Next Upgrade Ideas

- add bilingual documentation (Italian and English) with synchronized sections
- add endpoint examples for the most used meteo and CFD workflows
- add client-specific sequence diagrams for the highest-traffic endpoints