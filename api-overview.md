# API Overview

## Notes On Style

The backend exposes many task-oriented endpoints rather than a narrow REST resource model. Route names are explicit and often encode the action directly.

Authorization is currently tied to a token passed through the request query string for protected endpoints. That is an implementation detail worth documenting because it affects client integration.

## Authentication And User Endpoints

### Public or bootstrap endpoints

- `GET|POST /` - basic health check endpoint
- `POST /login` - authenticates a user and returns token plus permission graph

### Protected user endpoints

- `GET /get/new/token` - refresh token
- `POST /user/upload` - update user profile image
- `GET /user/get/search` - search users
- `POST /user/insert` - create a user
- `POST /user/update` - update a user
- `DELETE /user/delete` - delete a user
- `GET /user/roles` - retrieve role information and authorization graph data

## Meteo Endpoints

### Read endpoints

- `GET|POST /meteo/get_area_meteo` - retrieve meteo areas
- `GET|POST /meteo/get_fenomeno_meteo` - retrieve meteo phenomena
- `GET|POST /meteo/get_all_dati` - retrieve full meteo bulletin dataset and return PDF output
- `GET|POST /meteo/get_fascia_altimetrica_fenomeno_meteo` - retrieve altitude-band phenomenon data
- `GET|POST /meteo/get_ass_aree_fenomeni` - retrieve area-phenomenon associations
- `GET|POST /meteo/get_ass_scale_fenomeni` - retrieve scale-phenomenon associations
- `GET|POST /meteo/get_bollettino` - retrieve meteo bulletins
- `GET|POST /meteo/get_dato_bollettino` - retrieve meteo bulletin data rows
- `GET|POST /meteo/get_dato_extra_rischio` - retrieve extra-risk or discomfort data
- `GET|POST /meteo/get_note_fenomeno` - retrieve phenomenon notes
- `GET|POST /meteo/get_frasi_predefinite` - retrieve predefined phrases
- `GET|POST /meteo/get_tabelle_pioggia` - retrieve rainfall tables
- `GET|POST /meteo/get_soglie_pluvio` - retrieve rainfall thresholds
- `GET|POST /meteo/get_zona_disagio` - retrieve discomfort zones
- `GET|POST /meteo/get_ass_extra_rischio` - retrieve extra-risk associations
- `GET|POST /meteo/get_all_dati_mi` - generate combined meteo and CFD output

### Write endpoints

- `POST /meteo/set_dato_bollettino` - update meteo bulletin row data
- `POST /meteo/set_dato_extra_rischio` - update extra-risk data
- `POST /meteo/set_note_fenomeno` - update phenomenon notes
- `POST /meteo/set_bollettino` - create or update a meteo bulletin
- `POST /meteo/set_tendenza` - update tendency data
- `POST /meteo/set_descrizione_meteo` - update general meteo description
- `POST /meteo/set_dettaglio_fenomeni` - update detailed phenomenon description
- `POST /meteo/set_tendenza_disagio` - update discomfort tendency text
- `POST /meteo/set_avviso` - update bulletin warning text

## CFD Endpoints

### Read endpoints

- `GET|POST /cfd/get_stato_suolo` - retrieve soil-state data
- `GET /cfd/import_stato_suolo` - trigger soil-state import
- `GET /cfd/import_gimondi_stato_suolo_job` - trigger the soil-state import job path
- `GET|POST /cfd/get_livello_sezioni` - retrieve section-level data
- `GET /cfd/import_livello_sezioni` - trigger section-level import
- `GET|POST /cfd/get_note_rischio` - retrieve CFD risk notes
- `GET|POST /cfd/get_bollettino` - retrieve CFD bulletins
- `GET|POST /cfd/get_dato_bollettino` - retrieve CFD bulletin rows
- `GET /cfd/get_zone_allerta` - retrieve alert zones
- `GET /cfd/get_geojson_zone_allerta` - retrieve alert-zone geometry as GeoJSON
- `GET /cfd/get_livelli_allerta` - retrieve alert-level definitions
- `GET /cfd/get_rischi` - retrieve available risk types
- `GET /cfd/get_modelli` - retrieve model output references
- `GET /cfd/get_run_modellistica` - retrieve model-run data
- `GET|POST /cfd/get_livello_allerta_calcolo` - compute alert levels
- `GET|POST /cfd/get_tendenza` - retrieve tendency entities
- `GET|POST /cfd/get_dati_tendenza` - retrieve tendency row data
- `GET /cfd/get_table_piogge` - retrieve rainfall tables for CFD workflows
- `GET /cfd/get_sezioni` - retrieve configured sections
- `GET|POST /cfd/get_valanghe` - retrieve avalanche data
- `GET|POST /cfd/get_all_dati` - generate full CFD output and return PDF

### Write endpoints

- `POST /cfd/set_note_rischio` - update risk notes
- `POST /cfd/set_bollettino` - create or update a CFD bulletin
- `POST /cfd/set_dato_bollettino` - update a CFD bulletin row
- `POST /cfd/delete_dato_bollettino` - delete a CFD bulletin row
- `POST /cfd/set_multi_dato_bollettino` - batch update multiple CFD rows
- `POST /cfd/copy_note_bollettino` - copy note content into CFD bulletin context
- `GET|POST /cfd/change_livello_allerta` - override an alert level
- `GET|POST /cfd/copy_livello_allerta` - copy alert level data
- `GET|POST /cfd/copy_all_livello_allerte` - copy all alert levels between contexts
- `GET|POST /cfd/copy_oggi_domani_allerte` - copy alert levels between day windows
- `POST /cfd/set_tendenza` - update tendency data
- `GET|POST /cfd/update_valanghe` - update avalanche data

## Socket.IO Events

The HTTP API is paired with real-time events used by clients editing the same bulletin.

Observed event families include:

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

Typical pattern:

1. a client performs an HTTP write
2. the backend persists the change
3. the backend broadcasts a room-scoped event
4. other connected clients refresh local state

## Response Types

The backend returns a mix of response types.

- JSON payloads for most search and update operations
- file responses for generated PDF outputs
- error payloads through Flask error handlers for 401, 403, 404, 415, and 500 cases

## Error Handling Model

Observed route behavior uses:

- explicit `abort(...)` calls for authorization and processing failures
- application logger exception capture
- JSON error responses configured through global Flask error handlers

## Client Integration Notes

Important client-side assumptions to document:

- some endpoints accept both GET and POST
- write endpoints often use form-like request structures rather than clean JSON-only contracts
- authorization is coupled to query-string token transport in the current implementation
- the frontend is expected to react to Socket.IO broadcasts after writes

## Good Public Summary

`The API exposes task-oriented endpoints for authentication, weather bulletin editing, CFD alert workflows, operational imports, and PDF export, with Socket.IO used to synchronize concurrent edits across connected clients.`