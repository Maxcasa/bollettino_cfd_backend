# Security And Publication Notes

## Why This Exists

The current backend is strongly tied to internal infrastructure. Publishing source code without review would likely expose secrets, internal addresses, and operational assumptions.

This document explains what must be sanitized before sharing anything beyond documentation.

## Safe Publication Boundary

Publishing this documentation folder by itself is the low-risk option.

It avoids publishing:

- source code
- credentials
- internal IP addresses
- internal hostnames
- deployment artifacts
- operational sample data

## Sensitive Areas Identified In The Codebase

### Configuration files and Kubernetes Secrets

The backend depends on JSON configuration files (`utils/`) that may contain:

- database credentials
- external-service credentials
- JWT signing secrets
- internal hostnames and paths

On Kubernetes these files are provided by the Secret `secrets-config`. The Secret manifest in the repository currently carries real values: it must not be committed or published, and the values should be rotated if it has been shared. Prefer creating the Secret from files (`kubectl create secret generic ... --from-file`) or a secret manager, and generate a strong random `jwt_key`.

These files should never be published as-is.

### Hard-coded credentials in source code

The `GET /nextclaud_cred` route in `WebServices.py` returns WebDAV credentials that are written directly in the source. Remove the route and move the credentials to the Secret before any code is shared.

### Logs

Logs are shipped to Loki and visible to everyone with Grafana access. Do not log tokens, passwords or full query strings (the JWT travels in the `authorization` query parameter, so full request URLs in access logs contain it). Protect Grafana with a strong admin password stored in a Secret (not in the manifest), and avoid exposing its NodePort outside the internal network.

### Cluster exposure

NodePorts are open for the backend (30010), PostgreSQL (30005), Grafana (30030) and the registry (30500). Restrict them with firewall rules or switch the Services to `ClusterIP` behind the Ingress.

## High-Risk Behaviors To Note

### Plain-text secret storage

The codebase structure suggests that credentials are stored in local JSON files. This is acceptable for a tightly controlled internal environment, but it is not appropriate for public publication.

Recommended replacement:

- environment variables
- encrypted secret stores
- deployment-specific secret injection

### Public disclosure risk through routes

There is at least one route in the current application whose purpose is to expose remote file-service credentials. Even if it is useful internally, it should not exist in any publicly shared or open deployment.

Required action before code publication:

- remove the endpoint entirely, or
- restrict it behind strong authentication and redesign the workflow

Known route currently matching this risk profile:

- `GET /nextclaud_cred`

### Query-string token transport

Protected routes currently rely on a token passed as a query parameter.

Operational risk:

- tokens are easier to leak through logs, browser history, reverse proxies, and monitoring tools

Recommended replacement:

- `Authorization: Bearer <token>` header transport

### Federated login exposure

SAML endpoints are part of the authentication surface and should be reviewed before publication for:

- redirect URI leakage
- tenant/domain details in logs and screenshots
- callback error payload disclosure

### Internal network coupling

The backend depends on internal providers, network shares, and enterprise services. Publishing these addresses would reveal more infrastructure detail than necessary.

Recommended documentation pattern:

- describe service roles only
- use placeholders such as `LDAP_HOST` or `MODEL_SHARE_PATH`
- avoid live hostnames and IP addresses

## Documentation Hygiene Rules For GitHub

When preparing public documentation or code excerpts:

- replace real usernames with placeholders
- replace passwords and keys with obvious dummy values
- replace internal hostnames and IP addresses with neutral examples
- do not include exported operational data files unless reviewed line by line
- avoid screenshots that reveal internal URLs, usernames, folder paths, or bulletin content

## Suggested Public Repository Contents

If your goal is a documentation-only repository, keep only:

- project overview
- architecture notes
- API overview
- setup expectations with placeholder configuration
- security considerations

Do not include:

- `utils/*.json` with real values
- `documents/` samples unless sanitized
- generated PDFs or exports
- database dumps
- internal deployment files (Kubernetes manifests with Secrets, registry addresses, hostPaths)
- virtual environments

## Suggested Disclaimer Text

You can reuse this short notice in a public repository:

`This repository contains documentation only. Operational source code, credentials, internal infrastructure details, and environment-specific data have been intentionally omitted.`

## Pre-Publication Checklist

Before publishing anything beyond this folder, verify all of the following:

- secrets are removed
- internal hostnames and IP addresses are replaced
- credential-returning endpoints are removed
- sample data is sanitized
- deployment scripts do not reveal private paths or service names
- screenshots and diagrams are reviewed for hidden metadata

## Recommendation

For your stated goal, keep the GitHub upload limited to this documentation bundle. That is the cleanest way to present the project without taking on avoidable security risk.