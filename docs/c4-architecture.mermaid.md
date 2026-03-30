# C4 Architecture

## System Context (L1)

```mermaid
flowchart TB
  actor_app_user["actor:app_user — End user"]
  sys_fullstack_app["sys:fullstack_app — Full-stack dashboard and API"]
  ext_traefik["ext:traefik — Traefik"]
  ext_postgres["ext:postgres — PostgreSQL"]
  ext_smtp["ext:smtp — SMTP email"]
  ext_sentry["ext:sentry — Sentry"]
  ext_adminer["ext:adminer — Adminer"]
  actor_app_user -->|Uses HTTPS| ext_traefik
  ext_traefik -->|Routes HTTPS| sys_fullstack_app
  sys_fullstack_app -->|SQL| ext_postgres
  sys_fullstack_app -->|Send email| ext_smtp
  sys_fullstack_app -->|Error telemetry| ext_sentry
```

## Container (L2)

```mermaid
flowchart TB
  actor_app_user["actor:app_user — End user"]
  ext_traefik["ext:traefik — Traefik"]
  ext_smtp["ext:smtp — SMTP email"]
  ext_sentry["ext:sentry — Sentry"]
  subgraph sys_fullstack_app["sys:fullstack_app — Full-stack dashboard and API"]
    ctn_frontend["container:frontend — Web frontend<br/>Vite / SPA"]
    ctn_backend["container:backend — API backend<br/>FastAPI"]
    ctn_postgres["container:postgres — PostgreSQL database<br/>PostgreSQL 18"]
  end
  actor_app_user --> ext_traefik
  ext_traefik -->|HTTPS| ctn_frontend
  ext_traefik -->|HTTPS| ctn_backend
  ctn_frontend -->|HTTPS API| ctn_backend
  ctn_backend -->|SQL| ctn_postgres
  ctn_backend --> ext_smtp
  ctn_backend --> ext_sentry
```

## Components — Web frontend (L3)

%% SCOPE: urn:c4:container:container:frontend

```mermaid
flowchart TB
  subgraph ctn_frontend["container:frontend"]
    direction TB
    comp_spa_router["component:spa_router — SPA routing"]
    %% KIND: router
    comp_openapi_client["component:openapi_client — Generated OpenAPI HTTP client"]
    %% KIND: integration
    comp_react_query["component:react_query_layer — TanStack Query client state"]
    %% KIND: service_layer
  end
```

## Components — API backend (L3)

%% SCOPE: urn:c4:container:container:backend

```mermaid
flowchart TB
  subgraph ctn_backend["container:backend"]
    direction TB
    comp_fastapi_router["component:fastapi_api_router — FastAPI API router aggregation"]
    %% KIND: router
    comp_http_handlers["component:http_route_handlers — HTTP route handlers"]
    %% KIND: service_layer
    comp_sqlalchemy["component:sqlalchemy_engine — SQLAlchemy engine and sessions"]
    %% KIND: data_access
  end
```

## Components — PostgreSQL database (L3)

%% SCOPE: urn:c4:container:container:postgres

```mermaid
flowchart TB
  subgraph ctn_postgres["container:postgres"]
    direction TB
    comp_pg_server["component:postgres_server — PostgreSQL server"]
    %% KIND: data_access
    comp_pg_storage["component:postgres_storage — PGDATA volume persistence"]
    %% KIND: storage
  end
```

## Cross-container component relations (from stream)

```mermaid
flowchart LR
  comp_openapi_client["component:openapi_client"]
  comp_http_handlers["component:http_route_handlers"]
  comp_sqlalchemy["component:sqlalchemy_engine"]
  comp_pg_server["component:postgres_server"]
  comp_openapi_client -->|HTTPS JSON| comp_http_handlers
  comp_sqlalchemy -->|SQL over network| comp_pg_server
```

## Containment Map

```json
{
  "parentToChildren": {
    "sys_fullstack_app": ["ctn_frontend", "ctn_backend", "ctn_postgres"],
    "ctn_frontend": ["comp_spa_router", "comp_openapi_client", "comp_react_query"],
    "ctn_backend": ["comp_fastapi_router", "comp_http_handlers", "comp_sqlalchemy"],
    "ctn_postgres": ["comp_pg_server", "comp_pg_storage"]
  },
  "childToParent": {
    "ctn_frontend": "sys_fullstack_app",
    "ctn_backend": "sys_fullstack_app",
    "ctn_postgres": "sys_fullstack_app",
    "comp_spa_router": "ctn_frontend",
    "comp_openapi_client": "ctn_frontend",
    "comp_react_query": "ctn_frontend",
    "comp_fastapi_router": "ctn_backend",
    "comp_http_handlers": "ctn_backend",
    "comp_sqlalchemy": "ctn_backend",
    "comp_pg_server": "ctn_postgres",
    "comp_pg_storage": "ctn_postgres"
  }
}
```
