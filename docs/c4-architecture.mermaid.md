# C4 Architecture

Derived from `docs/c4-events.ndjson` (replay projection). Pass 4 adds no new elements.

## L1 — System Context

```mermaid
flowchart TD
  actor_end_user["actor:end_user — End user (browser)"]
  ext_pg["ext:postgresql — PostgreSQL"]
  ext_smtp["ext:smtp — SMTP email service"]
  ext_sentry["ext:sentry — Sentry (error monitoring)"]
  sys["sys:full_stack — Full-stack web application (FastAPI + React SPA)"]
  actor_end_user -->|"Uses web UI"| sys
  sys -->|"SQL (SQLAlchemy)"| ext_pg
  sys -->|"Outbound email (when configured)"| ext_smtp
  sys -->|"Error telemetry (non-local env)"| ext_sentry
```

## L2 — Container Diagram

```mermaid
flowchart TD
  actor_end_user["actor:end_user — End user (browser)"]
  ext_pg["ext:postgresql — PostgreSQL"]
  ext_smtp["ext:smtp — SMTP email service"]
  ext_sentry["ext:sentry — Sentry (error monitoring)"]
  subgraph sys_full_stack["sys:full_stack"]
    direction TB
    %% SCOPE: urn:c4:container:sys:full_stack
    spa["container:spa_web — Web client (Vite SPA, Nginx static host)"]
    api["container:fastapi_api — Backend API (FastAPI ASGI)"]
    db["container:postgres_db — PostgreSQL database server"]
    pre["container:prestart_job — Prestart job (migrations + initial data)"]
  end
  actor_end_user -->|"Uses web UI"| spa
  spa -->|"HTTPS / JSON API (VITE_API_URL)"| api
  api -->|"SQLAlchemy / psycopg"| db
  pre -->|"Alembic migrations + seed scripts"| db
  api -->|"SMTP (when configured)"| ext_smtp
  api -->|"Sentry SDK"| ext_sentry
```

## L3 — container:spa_web

```mermaid
flowchart TD
  subgraph spa_c["container:spa_web — Web client (Vite SPA, Nginx static host)"]
    direction TB
    %% SCOPE: urn:c4:container:container:spa_web
    spa_r["component:spa_tanstack_router — Client-side router"]
    %% KIND: router
    spa_o["component:spa_openapi_client — Generated OpenAPI HTTP client"]
    %% KIND: integration
    spa_q["component:spa_react_query — Server-state cache"]
    %% KIND: storage
  end
```

## L3 — container:fastapi_api

```mermaid
flowchart TD
  subgraph api_c["container:fastapi_api — Backend API (FastAPI ASGI)"]
    direction TB
    %% SCOPE: urn:c4:container:container:fastapi_api
    api_rt["component:api_fastapi_routes — HTTP API surface"]
    %% KIND: router
    api_h["component:api_route_handlers — Route handlers"]
    %% KIND: service_layer
    api_d["component:api_crud_sqlmodel — Persistence layer"]
    %% KIND: data_access
  end
```

## L3 — container:postgres_db

```mermaid
flowchart TD
  subgraph db_c["container:postgres_db — PostgreSQL database server"]
    direction TB
    %% SCOPE: urn:c4:container:container:postgres_db
    db_e["component:db_postgres_engine — PostgreSQL storage engine"]
    %% KIND: storage
    db_m["component:db_schema_migrations — Schema migrations target"]
    %% KIND: pipeline
    db_a["component:db_app_relations — Application relational data"]
    %% KIND: storage
  end
```

## L3 — container:prestart_job

```mermaid
flowchart TD
  subgraph pre_c["container:prestart_job — Prestart job (migrations + initial data)"]
    direction TB
    %% SCOPE: urn:c4:container:container:prestart_job
    pre_r["component:pre_db_readiness — DB readiness check"]
    %% KIND: integration
    pre_a["component:pre_alembic_upgrade — Alembic upgrade head"]
    %% KIND: pipeline
    pre_i["component:pre_initial_data — Initial data seed"]
    %% KIND: worker
  end
```

## Containment Map

```json
{
  "parentToChildren": {
    "sys:full_stack": [
      "container:spa_web",
      "container:fastapi_api",
      "container:postgres_db",
      "container:prestart_job"
    ],
    "container:spa_web": [
      "component:spa_tanstack_router",
      "component:spa_openapi_client",
      "component:spa_react_query"
    ],
    "container:fastapi_api": [
      "component:api_fastapi_routes",
      "component:api_route_handlers",
      "component:api_crud_sqlmodel"
    ],
    "container:postgres_db": [
      "component:db_postgres_engine",
      "component:db_schema_migrations",
      "component:db_app_relations"
    ],
    "container:prestart_job": [
      "component:pre_alembic_upgrade",
      "component:pre_initial_data",
      "component:pre_db_readiness"
    ]
  },
  "childToParent": {
    "container:spa_web": "sys:full_stack",
    "container:fastapi_api": "sys:full_stack",
    "container:postgres_db": "sys:full_stack",
    "container:prestart_job": "sys:full_stack",
    "component:spa_tanstack_router": "container:spa_web",
    "component:spa_openapi_client": "container:spa_web",
    "component:spa_react_query": "container:spa_web",
    "component:api_fastapi_routes": "container:fastapi_api",
    "component:api_route_handlers": "container:fastapi_api",
    "component:api_crud_sqlmodel": "container:fastapi_api",
    "component:db_postgres_engine": "container:postgres_db",
    "component:db_schema_migrations": "container:postgres_db",
    "component:db_app_relations": "container:postgres_db",
    "component:pre_alembic_upgrade": "container:prestart_job",
    "component:pre_initial_data": "container:prestart_job",
    "component:pre_db_readiness": "container:prestart_job"
  }
}
```
