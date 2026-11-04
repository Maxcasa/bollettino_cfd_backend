# API Overview

Aligned with `WebServices.py` on 2026-11-04. The backend is a FastAPI application served by uvicorn. Interactive OpenAPI pages are the framework defaults (`/docs`, `/openapi.json`) unless disabled.

## Notes On Style

The backend exposes many task-oriented endpoints rather than a narrow REST resource model. Route names are explicit and often encode the action directly. Each route is registered for a single HTTP method, as listed below.

Authorization is tied to a JWT passed through the `authorization` query-string parameter for protected endpoints. That is an implementation detail worth documenting because it affects client integration.

When the application is reached through the cluster ingress, REST routes are prefixed with `/api` (stripped before reaching the backend) and the WebSocket with `/ws`.

## Public And Bootstrap Endpoints

- `GET /` - basic presence check (returns a text message)
- `GET /health` - liveness/readiness probe used by Kubernetes, returns `{"status": "ok"}`
- `POST /login` - authenticates a user and returns token plus permission graph
- `GET /saml/login` - starts the Microsoft Entra ID login flow (`?email=` required)
- `GET /saml/callback` - callback that finalizes the sign-in and redirects the browser to the frontend with the JWT
- `POST /frontend/log` - receives frontend error reports as query parameters and writes them to the backend log
- `GET /nextclaud_cred` - returns remote file-service credentials (internal-only, high risk, see [security notes](security-and-publication-notes.md))
- `WebSocket /{room_id}` - real-time room endpoint, see [WebSocket](#websocket)

Maintenance and diagnostic routes without the authorization decorator: `GET /cfd/build_hash_chain`, `GET /cfd/check_hash_chain`, `GET /meteo/build_hash_chain`, `GET /meteo/check_hash_chain`, `GET /cfd/test_calcolo`. Do not expose them publicly.

## Protected User Endpoints

- `GET /get/new/token` - refresh token
- `POST /user/upload` - update user profile image
- `GET /user/get/search` - search users
- `GET /user/get/from_id` - retrieve user detail by id
- `POST /user/insert` - create a user
- `POST /user/update` - update a user
- `DELETE /user/delete` - delete a user
- `GET /user/roles` - retrieve role information and authorization graph data

## Meteo Endpoints

### Read endpoints (all `GET`)

- `/meteo/get_area_meteo` - meteo areas
- `/meteo/get_fenomeno_meteo` - meteo phenomena
- `/meteo/get_all_dati` - full meteo bulletin dataset, returned as PDF
- `/meteo/get_all_dati_mi` - combined meteo and CFD output
- `/meteo/get_fascia_altimetrica_fenomeno_meteo` - altitude-band phenomenon data
- `/meteo/get_ass_aree_fenomeni` - area-phenomenon associations
- `/meteo/get_ass_scale_fenomeni` - scale-phenomenon associations
- `/meteo/get_bollettino` - meteo bulletins
- `/meteo/get_dato_bollettino` - meteo bulletin data rows
- `/meteo/get_dato_extra_rischio` - extra-risk or discomfort data
- `/meteo/get_note_fenomeno` - phenomenon notes
- `/meteo/get_frasi_predefinite` - predefined phrases
- `/meteo/get_tabelle_pioggia` - rainfall tables
- `/meteo/get_soglie_pluvio` - rainfall thresholds
- `/meteo/get_zona_disagio` - discomfort zones
- `/meteo/get_ass_extra_rischio` - extra-risk associations

### Write endpoints (all `POST`)

- `/meteo/set_dato_bollettino` - meteo bulletin row data
- `/meteo/set_dato_extra_rischio` - extra-risk data
- `/meteo/set_note_fenomeno` - phenomenon notes
- `/meteo/set_bollettino` - create or update a meteo bulletin
- `/meteo/set_firma` - bulletin signature text
- `/meteo/set_titolo` - bulletin title text
- `/meteo/set_note_e_contatti` - notes and contacts text
- `/meteo/set_tendenza` - tendency data
- `/meteo/set_descrizione_meteo` - general meteo description
- `/meteo/set_dettaglio_fenomeni` - detailed phenomenon description
- `/meteo/set_tendenza_disagio` - discomfort tendency text
- `/meteo/set_avviso` - bulletin warning text

## CFD Endpoints

### Read endpoints (all `GET`)

- `/cfd/get_stato_suolo` - soil-state data
- `/cfd/get_livello_sezioni` - section-level data
- `/cfd/get_note_rischio` - CFD risk notes
- `/cfd/get_bollettino` - CFD bulletins
- `/cfd/get_dato_bollettino` - CFD bulletin rows
- `/cfd/get_zone_allerta` - alert zones
- `/cfd/get_geojson_zone_allerta` - alert-zone geometry as GeoJSON
- `/cfd/get_livelli_allerta` - alert-level definitions
- `/cfd/get_rischi` - available risk types
- `/cfd/get_run_modellistica` - model-run data
- `/cfd/get_tendenza` - tendency entities
- `/cfd/get_dati_tendenza` - tendency row data
- `/cfd/get_table_piogge` - rainfall tables for CFD workflows
- `/cfd/get_sezioni` - configured sections
- `/cfd/get_valanghe` - avalanche data
- `/cfd/get_all_dati` - full CFD output, returned as PDF. The response carries the custom header `X-info_dir: 1` when the infographic integration is configured (`infografica.input_dir` set); the frontend uses it to show the "GENERA INFOGRAFICA" button

### Import triggers (all `GET`)

- `/cfd/import_stato_suolo` - trigger soil-state import
- `/cfd/import_gimondi_stato_suolo_job` - trigger the soil-state import job path
- `/cfd/import_livello_sezioni` - trigger section-level import

Regular imports are normally run by the Kubernetes CronJobs, not through these endpoints.

### Infographic generation (`GET`)

- `/cfd/generazione_infografica` - trigger the headless generation of the CFD infographic assets (maps and tables for social posts/stories, plus a textual forecast). The backend drives an external infographic generator web app through Playwright/Chromium (the equivalent of its "Genera Tutto" button), waits for completion (timeout `infografica.timeout_sec`, default 240 s), then copies the generated files from the shared output folder to the configured publication folder (`infografica.copy_dir`). On success it publishes the WebSocket command `up_infografica` (with `files` and `warnings` lists) and returns `{"status": "ok", "files": [...], "warnings": [...], "messaggio": ...}`. Requires the `infografica` configuration section; returns HTTP 500 when `infografica.url` is not set.

### Write endpoints (all `POST`)

- `/cfd/set_note_rischio` - risk notes
- `/cfd/set_bollettino` - create or update a CFD bulletin
- `/cfd/set_dato_bollettino` - update a CFD bulletin row
- `/cfd/delete_dato_bollettino` - delete a CFD bulletin row
- `/cfd/set_multi_dato_bollettino` - batch update of CFD rows
- `/cfd/copy_note_bollettino` - copy note content into CFD bulletin context
- `/cfd/get_livello_allerta_calcolo` - compute alert levels (a `POST` despite the name)
- `/cfd/change_livello_allerta` - override an alert level
- `/cfd/copy_livello_allerta` - copy alert level data
- `/cfd/copy_all_livello_allerte` - copy all alert levels between contexts
- `/cfd/copy_oggi_domani_allerte` - copy alert levels between day windows
- `/cfd/set_tendenza` - tendency data

## WebSocket

The HTTP API is paired with a native WebSocket endpoint used by clients editing the same bulletin.

- Endpoint: `/{room_id}` on the backend, `/ws/{room_id}` through the ingress. The room name comes from `room` in `db_config.json`.
- Messages are JSON text frames. After a successful write the backend publishes on Redis a message such as `{"command": "up_boll_cfd", "socket_sid": "...", ...ids}` and every replica forwards it to its connected clients.
- Messages sent by a client are republished unchanged to the whole room.
- During a rollout the server closes connections with code `1001`; clients are expected to reconnect.

Observed `command` values:

- `up_dato_meteo`
- `up_dato_extra`
- `up_note_fenomeno`
- `up_note_meteo`
- `up_boll_meteo`
- `up_boll_cfd`
- `up_dato_cfd`
- `up_note_cfd`
- `up_zona_allerta`
- `up_tendenza`
- `up_infografica`

Typical pattern:

1. a client performs an HTTP write
2. the backend persists the change
3. the backend publishes a room-scoped message on Redis
4. other connected clients refresh local state (the sender is recognised through `socket_sid`)

## Response Types

- JSON payloads for most search and update operations
- file responses for generated PDF outputs
- `HTTPException` responses with `401` (missing/invalid token or insufficient level) and `500` (generic route error, with a short fixed `detail`); validation errors from FastAPI use `422`

## Error Handling Model

- every route catches exceptions, logs them with the stack trace (`logger.exception`) and re-raises an `HTTPException`
- stack traces therefore end up in the stdout log collected by Loki; the HTTP response carries only a short code-like message

## SQL Trigger Error Codes

Bulletin insert and update checks in database triggers raise custom SQLSTATE codes.
These codes are stable identifiers that clients can use instead of parsing message text.

### Update checks (`update_check_bollettino`)

- `BLC01` - update denied because the bulletin is already validated (`validazione` 4 or 6)
- `BLC02` - update denied because `data_validita` change is not allowed
- `BLC03` - update denied because another same-day bulletin already exists with `validazione <= 4`
- `BLC04` - update denied because a later revision exists on the same day and is not finalized
- `BLC05` - update denied because another same-day bulletin is not yet validated

### Insert checks (`insert_check_bollettino`)

- `BLI01` - insert denied because another same-day bulletin already exists with `validazione <= 4`
- `BLI02` - insert denied because another same-day bulletin with later or equal `data_validita` exists
- `BLI03` - insert denied because another same-day bulletin is not yet validated

### Integration guidance

- Treat SQLSTATE as the primary machine-readable error key.
- Keep localized user messages in the client layer and map them by code.
- Log both SQLSTATE and message text for operational diagnostics (they can be searched in Grafana, see [setup-and-operations.md](setup-and-operations.md#logging)).

## Client Integration Notes

- write endpoints often use form-like or query-string request structures rather than clean JSON-only contracts
- authorization is coupled to query-string token transport in the current implementation
- CORS is open to all origins
- the frontend is expected to react to WebSocket messages after writes

## Authentication Modes

- token-based route authorization via the `authorization` query parameter (JWT, with `user_level` checked against the level required by the route)
- Microsoft Entra ID bootstrap endpoints for browser login handoff

For external integrations, document both modes explicitly and prefer moving token transport to bearer headers in future revisions.

## Good Public Summary

`The API exposes task-oriented endpoints for authentication, weather bulletin editing, CFD alert workflows, operational imports, and PDF export, with a WebSocket channel backed by Redis Pub/Sub used to synchronize concurrent edits across connected clients and backend replicas.`
