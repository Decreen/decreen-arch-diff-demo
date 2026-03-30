# C4 Architecture

Projected from `docs/c4-events.ndjson` (append-only stream). System label refined in Pass 4.

## L1 — System context

```mermaid
flowchart TB
  actor_end_user["actor:end_user<br/>End user"]
  sys_app["sys:full_stack_app<br/>FastAPI full-stack template application"]
  ext_postgres["ext:postgres<br/>PostgreSQL"]
  ext_sentry["ext:sentry<br/>Sentry"]
  ext_email["ext:outbound_email<br/>Outbound email"]

  actor_end_user -->|"Uses web UI"| sys_app
  actor_end_user -->|"API requests (browser / HTTPS)"| sys_app
  sys_app -->|"SQL / persistence"| ext_postgres
  sys_app -->|"SDK telemetry"| ext_sentry
  sys_app -->|"Send mail"| ext_email
```

## L2 — Containers

```mermaid
flowchart TB
  subgraph sys_full_stack["sys:full_stack_app — FastAPI full-stack template application"]
    container_nginx["container:nginx_spa<br/>Frontend static site (Nginx serving Vite build)"]
    container_api["container:fastapi_api<br/>Backend HTTP API (FastAPI)"]
  end

  actor_end_user["actor:end_user<br/>End user"]
  ext_postgres["ext:postgres<br/>PostgreSQL"]
  ext_sentry["ext:sentry<br/>Sentry"]
  ext_email["ext:outbound_email<br/>Outbound email"]

  actor_end_user -->|"Uses web UI"| container_nginx
  actor_end_user -->|"API requests (browser / HTTPS)"| container_api
  container_api -->|"SQL / persistence"| ext_postgres
  container_api -->|"SDK telemetry"| ext_sentry
  container_api -->|"Send mail"| ext_email
```

## L3 — container:nginx_spa

```mermaid
flowchart TB
  %% SCOPE: urn:c4:container:nginx_spa
  comp_nginx_router["component:nginx_router<br/>%% KIND: router"]
  comp_static["component:static_spa_delivery<br/>%% KIND: storage"]
  comp_nginx_router -->|"Serves static files"| comp_static
```

## L3 — container:fastapi_api

```mermaid
flowchart TB
  %% SCOPE: urn:c4:container:fastapi_api
  comp_agg["component:api_aggregate_router<br/>%% KIND: router"]
  comp_handlers["component:http_route_handlers<br/>%% KIND: service_layer"]
  comp_sql["component:sqlmodel_access<br/>%% KIND: data_access"]
  comp_agg -->|"Includes route modules"| comp_handlers
  comp_handlers -->|"Uses DB session / CRUD"| comp_sql
  comp_sql -->|"SQL over psycopg"| ext_postgres["ext:postgres<br/>PostgreSQL"]
```

## Containment Map

```json
{
  "parentToChildren": {
    "full_stack_app": ["nginx_spa", "fastapi_api"],
    "nginx_spa": ["nginx_router", "static_spa_delivery"],
    "fastapi_api": ["api_aggregate_router", "http_route_handlers", "sqlmodel_access"]
  },
  "childToParent": {
    "nginx_spa": "full_stack_app",
    "fastapi_api": "full_stack_app",
    "nginx_router": "nginx_spa",
    "static_spa_delivery": "nginx_spa",
    "api_aggregate_router": "fastapi_api",
    "http_route_handlers": "fastapi_api",
    "sqlmodel_access": "fastapi_api"
  }
}
```
