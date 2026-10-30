<div align="center">

# 📚 sentinelone-api-python

**Python client for the SentinelOne Management Console API, for security automation scripts.**

A thin wrapper around the SentinelOne REST API (`/web/api/v2.1`) that handles login and cursor pagination. It is a library imported by other scripts. It is not a service and has no CLI.

[![Python](https://img.shields.io/badge/Python-%E2%89%A53.5-3776AB?logo=python&logoColor=white)](setup.py)
[![Dependency](https://img.shields.io/badge/requests-unpinned-blue)](requirements.txt)
[![Docker](https://img.shields.io/badge/Docker-python%3A3--alpine-2496ED?logo=docker&logoColor=white)](Dockerfile)
[![License](https://img.shields.io/badge/License-LGPL--3.0-blue)](LICENSE)

</div>

> [!NOTE]
> This repo is a Glovo fork of [`fragtastic/sentinelone-api-python`](https://github.com/fragtastic/sentinelone-api-python). Glovo's own additions are the isolate, scan and quarantine calls (see [Overview](#overview)).

---

## Table of contents

- [Overview](#overview)
- [Build & local development](#build--local-development)
- [Security & data handling](#security--data-handling)
- [Repository layout](#repository-layout)
- [Contributing & conventions](#contributing--conventions)
- [Ownership & support](#ownership--support)

---

## Overview

`sentineloneapi.Client` logs in to a SentinelOne console with an API token and exposes one method per API call (about 43 methods in [`client.py`](sentineloneapi/client.py)). Every call returns a `(data, errors)` tuple. **Pagination is automatic**: `api_call` follows `pagination.nextCursor` until the console stops returning one, and merges all pages into `data`.

| | |
|---|---|
| Language / build | Python `>=3.5`, `setuptools` ([`setup.py`](setup.py)) |
| Dependency | `requests` ([`requirements.txt`](requirements.txt)), unpinned |
| Package name | `sentinelone-api-python-fragtastic` `0.0.2` (upstream name, kept in [`setup.py`](setup.py)) |
| API | SentinelOne Management Console REST API `v2.1` |

### Calls by area

| Area | Examples |
|---|---|
| Agents | `GetAgents`, `CountAgents`, `DecommissionAgent`, `MoveToSite`, `MoveToGroup` |
| Agent actions (Glovo) | `disconnect_agent` (isolate), `reconnect_agent` (un-isolate), `initiate_scan`, `abort_scan` |
| Threats | `GetThreats`, `mitigate_threat` (Glovo, quarantine) |
| Policies | `GlobalPolicy`, `AccountPolicy`, `GroupPolicy` |
| Sites, groups, roles | `GetSites`, `GetSiteById`, `GetGroups`, `GetAllRoles`, `GetSpecificRoleDefinition` |
| Users and tokens | `ListUsers`, `GenerateApiToken`, `RevokeApiToken`, `ChangePassword`, `SendVerificationEmail` |
| Vulnerabilities, alerts | `GetCves`, `GetAlerts` |
| System | `SystemInfo`, `SystemStatus`, `CacheStatus`, `DatabaseStatus`, `GetSystemConfig`, `SetSystemConfig` |

The `disconnect_agent`, `reconnect_agent` and `mitigate_threat` calls act on live endpoints. `disconnect_agent` and `reconnect_agent` read the agent ID from the first entry of `filter.ids` in the payload and use only that one.

---

## Build & local development

> [!NOTE]
> Needs Python `>=3.5` and a SentinelOne API token for a console URL. There are no tests, linters or CI in this repo.

```bash
pip install -r requirements.txt   # installs requests
python setup.py install           # installs the sentineloneapi package
./build.sh                        # builds the Docker image sentineloneapi:latest (python:3-alpine + setup.py install)
```

### Use the client

**All API calls automatically handle pagination.**

```python
import sentineloneapi

client = sentineloneapi.Client(
                    username='username@domain.tld',
                    password='<your-password>',
                    url='example.sentinelone.net'
                )

# One or the other.
# Username/password and apitoken are mutually exclusive.

client = sentineloneapi.Client(
                    apitoken='<your-api-token>',
                    url='example.sentinelone.net'
                )

client.authenticate()

data, errors = client.CountAgents(payload={'computerName__like': 'test'})

data, errors = client.ListUsers(payload={
        'skipCount': True,
        'query': 'Test'
    })
```

> [!WARNING]
> These points from reading the code differ from the example above:
> - `authenticate()` only supports `apitoken`. With username and password it raises `AuthenticationError('Unhandled authentication method.')`.
> - `url` is joined directly with `/web/api/v2.1/...`, so pass the full base URL with scheme (for example `https://example.sentinelone.net`), or `requests` rejects it.
> - The client needs either `apitoken`, or `username` and `password`; otherwise it raises `InvalidParameters`. A missing `url` raises `MissingURL`.

`Client(..., verify='/path/ca-bundle.pem')` passes a certificate bundle to `requests` for TLS verification.

Errors are returned in the `errors` list of the tuple. Exception classes ([`exceptions.py`](sentineloneapi/exceptions.py)): `AuthenticationError`, `Unauthenticated`, `InvalidParameters`, `UnhandledRequestType`, `MissingURL`, `AuthenticationTokenExpired`, `APITokenExpired`, `Unimplemented`.

---

## Security & data handling

- Credentials: pass the API token or password at run time from your secret store. Never commit them.
- The token is sent as `Authorization: Token <token>` on every call.
- `verify` defaults to `None`, which `requests` treats as its default (TLS verification on). Use it only to point at a CA bundle.
- Debug logging (`logging` at DEBUG) prints full API responses, which can include agent and user data.

---

## Repository layout

```text
sentinelone-api-python/
├── sentineloneapi/       ⭐ the package
│   ├── __init__.py       exports Client
│   ├── client.py         Client: auth, pagination, one method per API call
│   ├── exceptions.py     exception classes
│   └── util.py           datetime parsing helper
├── setup.py              packaging (setuptools)
├── requirements.txt      runtime dependency (requests)
├── Dockerfile            python:3-alpine, runs setup.py install
├── build.sh              builds the Docker image sentineloneapi:latest
└── LICENSE
```

---

## Contributing & conventions

- Add a new API call as a `Client` method in [`client.py`](sentineloneapi/client.py) that sets `endpoint` and returns `self.api_call(requests.get|post|put, endpoint, payload)`. Only `GET`, `POST` and `PUT` are supported; anything else raises `UnhandledRequestType`.
- Keep the `(data, errors)` return shape.
- Open a PR against `main`. There is no CI, so run your change against a console before asking for review.
- Upstream: [`fragtastic/sentinelone-api-python`](https://github.com/fragtastic/sentinelone-api-python). Glovo commits are ahead of upstream on `main`.

---

## Ownership & support

- No `CODEOWNERS` file or catalog entry exists in this repo. The last Glovo change is the `SEC-6487` PR (2025-07-07, [#1](https://github.com/Glovo/sentinelone-api-python/pull/1)).
