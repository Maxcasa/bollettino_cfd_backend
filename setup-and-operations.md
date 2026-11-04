# Setup And Operations

Aligned with the codebase and the Kubernetes manifests on 2026-11-04.

## Purpose

This document explains how the backend is built, configured, deployed and observed. The production target is **Docker images running on Kubernetes (K3s)**. Local Docker Compose and a plain Python run are supported for development. It is written without live credentials or internal network addresses.

Cluster-wide procedures (first deploy, image transfer, `kubectl` cheat sheet, logging stack) are in the repository-root `docs/` folder:

- `docs/avvio-progetto-kubernetes.md` - load and start the project on the cluster
- `docs/comandi-kubernetes.md` - day-to-day `kubectl` commands
- `docs/logging-observability.md` - stdout, Fluent Bit, Loki and Grafana
- `docs/proposta-kubernetes-alta-disponibilita.md` - high-availability proposal

## Runtime Stack

- Python 3.11 (image `python:3.11-slim`)
- FastAPI and uvicorn
- psycopg2 and PostgreSQL 16
- Redis 7 (WebSocket Pub/Sub, `redis.asyncio`)
- APScheduler
- reportlab, svglib, matplotlib, Playwright/Chromium for report and map generation
- LDAP (`ldap3`) and MSAL for authentication

Dependencies are pinned in `requirements.txt`.

## Docker Image

The `Dockerfile` in the backend root builds the image:

- sets `PYTHONUNBUFFERED=1` so log lines are written immediately to stdout
- sets locale `it_IT.UTF-8` and timezone `Europe/Rome`
- installs system libraries (libpq, Cairo) and the Playwright Chromium browser
- copies the source and starts `python WebServices.py`, which launches uvicorn on `backname:backport`

Every import job under `activations/` has its own small Dockerfile and image (`stato_suolo_job`, `tabelle_pioggia_job`, `dati_valanghe_job`, `modellistica_job`).

Local run with Compose:

```powershell
docker compose up -d --build
curl http://127.0.0.1:5005/health
```

`compose.yaml` publishes host port 5005 on container port 5000 and mounts the three `utils/*.json` files read-only, plus `log/`, `out/` and an external data folder (`EXTERNAL_DATA_DIR`) on `/data`.

## Kubernetes Deployment

The manifests are in `k8s/`. The backend-relevant resources are:

| Resource | File | Notes |
|---|---|---|
| Deployment `multirischio-backend` | `backend.yaml` | 2 replicas, rolling update (`maxUnavailable: 0`), 60 s termination grace, requests 250m CPU / 512Mi, limits 1 CPU / 1Gi, `/health` readiness and liveness probes, soft pod anti-affinity on label `app=backend` |
| Service `backend-service` | `backend.yaml` | NodePort 30010, port 5050 |
| StatefulSet `postgres-db`, Service `postgres-service` | `postgres.yaml` | custom PostgreSQL image, data in a hostPath |
| Deployment/Service `redis-master` | `redis.yaml` | Pub/Sub for WebSocket rooms |
| CronJobs | `bollettino-cron.yaml` | one per import job, pods labelled `app=cron` and `job=<name>` |
| Secret `secrets-config` | `bollettino-secrets.yaml` | `db_config.json`, `path_config.json`, `secret_config.json`, `rclone.conf` |
| Ingress `app-ingress`, `app-ws-ingress`, Middlewares | `ingress.yaml`, `stripprefix.yaml` | Traefik: `/api` and `/ws` to the backend with prefix stripped, `/` to the frontend |
| rclone Deployments | `rclone-doppio-servizio.yaml` | Google Drive and WebDAV exposed as internal services for the avalanche import |
| Local registry | `local-registry.yaml`, `registries.yaml` | private registry on NodePort 30500, K3s mirror configuration |

Important points:

- Images are pulled from the private registry (`<registry>:30500/multirischio-backend:prod`, `imagePullPolicy: Always`), so after pushing a new image a `rollout restart` is enough.
- The Secret is mounted file by file with `subPath` into `/app/utils/`. Changing the Secret requires a `rollout restart` of the Deployment.
- The backend must listen on `0.0.0.0` and on the port declared in `backend-service` (5050): set `backname` and `backport` accordingly in `db_config.json`.
- The WebSocket and the write notifications need Redis: the client is created lazily, so the pod starts without it, but WebSocket connections and the publish after writes fail while Redis is unreachable.
- Generated files go to `/app/out` (an `emptyDir` in the backend pod; the CronJob for avalanches and the frontend use the hostPath `out/` directory).
- Timezone `Europe/Rome` is forced through `TZ` and `PGTZ`.

Typical update of the backend:

```bash
docker build -t <registry>/multirischio-backend:prod ./bollettino_cfd_backend
docker push <registry>/multirischio-backend:prod
kubectl rollout restart deployment/multirischio-backend
kubectl rollout status deployment/multirischio-backend --timeout=180s
```

CronJob images are updated the same way; the next scheduled run pulls the new image (`imagePullPolicy: Always`).

## Configuration

Configuration is read from JSON files under `utils/`. **Every value can be overridden by an environment variable** (`DB_HOSTNAME`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `DB_BACKNAME`, `DB_BACKPORT`, `LOCAL_MODEL_DIR`, `RUN_MODE`, `ROOM`, `SCHEDULE_STATUS`, `LOGGING_LEVEL`, `LOG_OUTPUT`, `REDIS_URL`, `INFOGRAFICA_URL`, `INFOGRAFICA_OUTPUT_DIR`, `INFOGRAFICA_COPY_DIR`, `INFOGRAFICA_COPY_NAME`, `INFOGRAFICA_INPUT_DIR`, `INFOGRAFICA_TIMEOUT_SEC`), which takes precedence over the file.

### `utils/db_config.json`

Sanitized example:

```json
{
  "hostname": "postgres-service",
  "database": "DB_NAME",
  "username": "DB_USER",
  "password": "DB_PASSWORD",
  "backname": "0.0.0.0",
  "backport": 5050,
  "local_model_dir": "/data",
  "run_mode": 1,
  "room": "multirischio",
  "logging_level": "INFO",
  "log_output": "stdout",
  "redis_url": "redis://redis-master.default.svc.cluster.local:6379/0",
  "schedule": {
    "status": "False",
    "jobs": {}
  },
  "infografica": {
    "url": "http://infografica-service:3000/",
    "output_dir": "/data/infografica/output",
    "input_dir": "/data/infografica/input",
    "copy_dir": "/data/infografica/publish",
    "copy_name": "PUBLISH_FOLDER_LABEL",
    "timeout_sec": 240
  }
}
```

- `log_output`: `stdout` (default, recommended), `stderr`, or a file name relative to the package (legacy)
- `logging_level`: standard Python level name
- `schedule.status`: keep `"False"` on Kubernetes; imports are run by CronJobs. When `"True"`, `schedule.jobs` selects the in-process jobs (`import_stato_suolo_job` with a `time`, the other jobs with `freq` in seconds)
- `infografica` (optional): enables `GET /cfd/generazione_infografica`. The backend copies the bulletin JSON/XML into `input_dir` (shared with the external infographic generator), drives the generator headlessly at `url` via Playwright/Chromium, waits up to `timeout_sec` seconds, then copies the generated files from `output_dir` to `copy_dir`. `copy_name` is the human-readable label of the publication folder shown in the success message. The feature is considered active only when `url`, `output_dir`, `input_dir`, `copy_dir` and `timeout_sec` are all set; when `input_dir` is configured, the `/cfd/get_all_dati` PDF response carries the `X-info_dir: 1` header

### `utils/secret_config.json`

Credentials of external providers (soil state service), avalanche import settings (local directory, rclone service URLs) and SMTP settings for notification e-mails. Sanitized example:

```json
{
  "stato_suolo": { "username": "SERVICE_USER", "password": "SERVICE_PASSWORD" },
  "valanghe": {
    "local_directory": "/app/out/Dati_Valanghe",
    "remote_gdrive": "http://svc-rclone-gdrive:8080/",
    "remote_nextcloud": "http://svc-rclone-remotewebdav:8080"
  },
  "gmail": {
    "SMTP_HOST": "SMTP_HOST",
    "SMTP_PORT": 25,
    "SENDER_EMAIL": "SENDER",
    "SMTP_USER": "SMTP_USER",
    "SMTP_TO": ["RECIPIENT"],
    "CC_EMAIL": ["CC_RECIPIENT"]
  }
}
```

### `utils/path_config.json`

Directory, SAML and JWT settings:

```json
{
  "jwt_key": "REPLACE_WITH_STRONG_RANDOM_SECRET",
  "ldap_server": "ldap://directory.example.internal",
  "ldap_company": "example.internal",
  "saml_domain": []
}
```

## Application Startup

1. `__init__.py` creates the FastAPI app, CORS middleware and the lifespan handler.
2. The database configuration is loaded and logging is configured (stdout by default).
3. The Redis client and the resource classes are instantiated.
4. The routes in `WebServices.py` are registered.
5. If enabled, APScheduler jobs are registered.
6. `uvicorn.run("WebServices:fastapi_app", host=backname, port=backport)` starts the server.

On `SIGTERM` (pod termination) the lifespan handler closes WebSocket connections with code 1001, cancels the Redis listeners and closes Redis, within the 60 s grace period.

## Health Check

`GET /health` returns `{"status": "ok"}` and is used by Kubernetes (readiness and liveness: 30 s initial delay, every 30 s, 10 s timeout, 5 failures). It does not verify the database.

```bash
kubectl get pods -l app=backend
curl http://<node>:30010/health
```

## Logging

Logging goes to **stdout**, never to files, so that Kubernetes and the logging stack can collect it.

- Format of application records: `<timestamp> <LEVEL> <logger name> <message>`.
- `LOG_OUTPUT=stdout` is set explicitly in the Deployment and in `db_config.json`; `PYTHONUNBUFFERED=1` avoids buffering.
- The application logger filters out messages that mention common HTTP verbs, to reduce noise. Uvicorn access and error logs use uvicorn's own format.
- Exceptions in route handlers are logged with `logger.exception` (full traceback).
- Browser errors posted to `POST /frontend/log` are logged as `Frontend log: {...}`.
- CronJob scripts use the same format and also log to stdout.

Collection pipeline (namespace `logging`):

```text
container stdout -> /var/log/containers -> Fluent Bit (DaemonSet) -> Loki -> Grafana
```

Only pods labelled `app=backend`, `app=postgres` and `app=cron` are shipped. Loki labels include `namespace_name`, `container_name`, `app` and `job` (CronJob name). Useful Grafana (Explore, Loki datasource) queries:

```logql
{app="backend"}
{app="backend"} |= "ERROR"
{app="cron", job="stato-suolo"}
{app="postgres"} |= "ERROR"
```

Details, deployment steps and troubleshooting are in `docs/logging-observability.md`. Quick checks:

```bash
kubectl logs deploy/multirischio-backend --tail=100
kubectl -n logging get pods
kubectl -n logging logs ds/fluent-bit --tail=50
```

Operational recommendations:

- avoid writing secrets into log messages and tracebacks
- Loki retention is 30 days; adjust it in the Loki configuration if needed

## Scheduled Imports

| CronJob | Script | Schedule in manifest |
|---|---|---|
| `stato-suolo-giornaliero` | `activations/stato_suolo` | `0 9 * * *` (daily) |
| `tabelle-pioggia-giornaliero` | `activations/tabelle_pioggia` | `*/2 * * * *` |
| `dati-valanghe-giornaliero` | `activations/dati_valanghe` | `1-59/2 * * * *` |
| `modellistica-giornaliero` | `activations/modellistica` | `*/2 * * * *` |

All use `concurrencyPolicy: Forbid`, timezone `Europe/Rome`, keep 3 successful and 5 failed jobs and restart on failure. The two-minute schedules look like test values: review them before production use. Jobs need outbound access to the providers and mount the same Secret as the backend.

```bash
kubectl get cronjob
kubectl get jobs
kubectl create job --from=cronjob/stato-suolo-giornaliero stato-suolo-manuale
```

## Outputs And Generated Files

- PDF bulletins, SVG maps and GeoJSON intermediates are written under `/app/out`
- avalanche data is downloaded to the directory configured in `secret_config.json`

These outputs must not be published unless sanitized.

## Operational Dependencies

Before considering a deployment healthy, verify:

- PostgreSQL connection works and the required tables and support functions exist
- Redis is reachable (`redis-master` Service)
- external providers (soil state, section levels, rclone services) are reachable
- the model-output share is mounted (CronJob `modellistica-giornaliero`)
- the configured WebSocket room name is accepted by clients
- output directories are writable
- Fluent Bit, Loki and Grafana are running in the `logging` namespace

## Local Development Without Docker

1. Create a virtual environment with Python 3.11.
2. `pip install -r requirements.txt`, then `playwright install chromium` if PDF rendering is needed.
3. Create the three JSON files under `utils/` (or set the environment variables above) and make PostgreSQL and Redis reachable.
4. Start with `python WebServices.py`.

## Practical Publishing Guidance

For a public documentation repository:

- describe configuration by schema, not by real values
- replace internal hostnames with placeholders
- state clearly which integrations are internal-only
- do not publish sample files containing operational data unless reviewed
