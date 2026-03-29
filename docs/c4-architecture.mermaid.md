# C4 Architecture

Projections from `docs/c4-events.ndjson` (Pass 4 frozen). Refined system label: **sys:app** — Full-stack web application (Docker Compose).

## L1 — System context

```mermaid
flowchart TD
  actor_user["actor:user — User"]
  ext_tr["ext:traefik — Traefik reverse proxy"]
  ext_pg["ext:postgresql — PostgreSQL"]
  ext_smtp["ext:smtp — SMTP mail server"]
  ext_sentry["ext:sentry — Sentry"]
  actor_user -->|"edge:user_to_traefik HTTPS"| ext_tr
```

## L2 — Containers

```mermaid
flowchart TD
  actor_user["actor:user — User"]
  ext_tr["ext:traefik — Traefik reverse proxy"]
  ext_pg["ext:postgresql — PostgreSQL"]
  ext_smtp["ext:smtp — SMTP mail server"]
  ext_sentry["ext:sentry — Sentry"]

  subgraph sys_app["sys:app — Full-stack web application (Docker Compose)"]
    c_fe["container:frontend — Web dashboard (static SPA)"]
    c_be["container:backend — HTTP API service"]
    c_db["container:db — PostgreSQL database"]
    c_pre["container:prestart — DB migrate and pre-start job"]
  end

  actor_user -->|"edge:user_to_traefik HTTPS"| ext_tr
  ext_tr -->|"edge:traefik_to_frontend HTTP(S) route"| c_fe
  ext_tr -->|"edge:traefik_to_backend HTTP(S) route"| c_be
  c_be -->|"edge:backend_to_postgres SQL"| ext_pg
  c_be -->|"edge:backend_to_smtp SMTP"| ext_smtp
  c_be -->|"edge:backend_to_sentry errors / tracing"| ext_sentry
  c_pre -->|"edge:prestart_to_postgres SQL"| ext_pg
```

## L3 — Components (selected containers)

### container:frontend

```mermaid
flowchart LR
  %% SCOPE: urn:c4:container:frontend
  %% KIND: router
  fe_r["component:fe_router — TanStack Router"]
  %% KIND: integration
  fe_c["component:fe_openapi_client — OpenAPI HTTP client"]
```

### container:backend

```mermaid
flowchart LR
  %% SCOPE: urn:c4:container:backend
  %% KIND: boundary
  be_api["component:be_http_api — FastAPI application and routers"]
  %% KIND: data_access
  be_da["component:be_persistence — SQLModel engine and CRUD"]
```

_Stream edges (from frozen stream):_ `edge:l3_fe_client_to_be_api` (component:fe_openapi_client → component:be_http_api); `edge:l3_be_persistence_to_pg` (component:be_persistence → component:pg_storage).

### container:db

```mermaid
flowchart LR
  %% SCOPE: urn:c4:container:db
  %% KIND: boundary
  pg_b["component:pg_protocol — Client connections / SQL interface"]
  %% KIND: storage
  pg_s["component:pg_storage — PostgreSQL data store"]
```

### container:prestart

```mermaid
flowchart LR
  %% SCOPE: urn:c4:container:prestart
  %% KIND: pipeline
  pre_al["component:pre_alembic — Alembic migrations"]
  %% KIND: worker
  pre_w["component:pre_db_init — initial_data seeding"]
```

## Containment map

| Parent | Child | Child type |
|--------|-------|------------|
| sys:app | container:frontend | container |
| sys:app | container:backend | container |
| sys:app | container:db | container |
| sys:app | container:prestart | container |
| container:frontend | component:fe_router | component (router) |
| container:frontend | component:fe_openapi_client | component (integration) |
| container:backend | component:be_http_api | component (boundary) |
| container:backend | component:be_persistence | component (data_access) |
| container:db | component:pg_protocol | component (boundary) |
| container:db | component:pg_storage | component (storage) |
| container:prestart | component:pre_alembic | component (pipeline) |
| container:prestart | component:pre_db_init | component (worker) |
