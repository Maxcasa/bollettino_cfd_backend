# Setup And Operations

## Purpose

This document explains what is required to run the backend in a local or controlled internal environment. It is intentionally written without publishing live credentials or internal network addresses.

## Runtime Stack

- Python 3.8 or newer
- Flask
- Flask-SocketIO
- psycopg2
- APScheduler
- matplotlib and report-generation dependencies
- PostgreSQL database reachable from the backend host

The current workspace already includes a Python virtual environment, but documentation intended for GitHub should not depend on a committed virtual environment.

## Local Setup Outline

1. Clone the private source repository into a working directory.
2. Create a Python virtual environment.
3. Install dependencies from `requirements.txt`.
4. Create the required JSON configuration files under `utils/`.
5. Ensure the PostgreSQL schema is available.
6. Ensure external integrations are reachable or stubbed.
7. Start the backend with `python WebServices.py`.

## Required Configuration Files

The current backend expects JSON files rather than environment variables.

### `utils/db_config.json`

Expected purpose:

- database host, name, username, and password
- backend bind host and port
- local model directory
- run mode
- Socket.IO room name
- optional scheduler configuration

Sanitized example:

```json
{
  "hostname": "DB_HOST",
  "database": "DB_NAME",
  "username": "DB_USER",
  "password": "DB_PASSWORD",
  "backname": "127.0.0.1",
  "backport": 5000,
  "local_model_dir": "D:/path/to/model/output",
  "run_mode": 0,
  "room": "multirischio",
  "schedule": {
    "status": "False",
    "time": "00:00"
  }
}
```

### `utils/secret_config.json`

Expected purpose:

- credentials for external data providers
- credentials for WebDAV or similar remote file services

Sanitized example:

```json
{
  "stato_suolo": {
    "username": "SERVICE_USER",
    "password": "SERVICE_PASSWORD"
  },
  "valanghe": {
    "webdav_username": "WEBDAV_USER",
    "webdav_password": "WEBDAV_PASSWORD"
  }
}
```

### `utils/path_config.json`

Expected purpose:

- enterprise directory configuration
- JWT signing key
- possibly other path or authentication settings used by helper utilities

Sanitized example:

```json
{
  "ldap_server": "ldap://directory.example.internal",
  "ldap_domain": "example.internal",
  "jwt_key": "REPLACE_WITH_STRONG_RANDOM_SECRET"
}
```

## Application Startup

Observed startup sequence:

1. Flask app and Socket.IO server are initialized.
2. Logging is configured to a file in `log/`.
3. Database configuration is loaded from `utils/db_config.json`.
4. Resource classes are instantiated.
5. Authorization graph data is loaded.
6. Optional scheduled jobs are registered.
7. The server starts with the configured host and port.

## Outputs And Generated Files

The backend does not operate as a pure JSON API. It also generates files.

Generated artifacts may include:

- PDF bulletins
- SVG files used for rendered maps
- GeoJSON intermediate data
- log files

These outputs should normally be excluded from public repositories unless they are sanitized examples.

## Operational Dependencies

Before considering a deployment healthy, verify the following:

- PostgreSQL connection works
- required tables and support functions exist
- external providers for soil state and section levels are reachable
- any model-output folder or share is accessible
- the configured Socket.IO room name is accepted by clients
- output directories are writable

## Deployment Notes

The internal notes in the project indicate a deployment model that uses:

- a reverse proxy such as Nginx
- a system service for backend startup
- internal Linux service-management scripts

This suggests the backend is intended to run as a long-lived service behind a reverse proxy rather than as a development-only Flask process.

## Logging

Logging is file-based.

Observed behavior:

- log target is under `log/logfile.log`
- common HTTP verb messages are filtered to reduce noise
- exceptions are logged in route handlers

Operational recommendation:

- rotate logs outside the application
- avoid writing secrets into exception traces

## Scheduler Behavior

The scheduler is configuration-driven.

When enabled, it can run periodic data import tasks. This has two practical consequences:

- a deployment must be allowed outbound network access to required providers
- service credentials must be handled as secrets, not as publishable documentation content

## Practical Publishing Guidance

For a public documentation repository:

- describe configuration by schema, not by real values
- replace internal hostnames with placeholders
- state clearly which integrations are internal-only
- do not publish sample files containing operational data unless reviewed