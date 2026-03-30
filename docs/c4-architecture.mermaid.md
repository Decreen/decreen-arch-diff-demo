# C4 Architecture

Projected from `docs/c4-events.ndjson` (append-only stream). Pass 4 applies label refinements only.

## L1 — System context

```mermaid
flowchart TB
  actor_end_user["actor:end_user<br/>End user"]
  ext_traefik["ext:traefik<br/>Traefik ingress"]
  ext_sentry["ext:sentry<br/>Sentry"]
  ext_smtp["ext:smtp<br/>SMTP / email delivery"]
  ext_postgres["ext:postgres<br/>PostgreSQL"]
  sys_platform["sys:platform<br/>Full-stack web application (Docker Compose stack)"]
  actor_end_user -->|"HTTPS (dashboard / API hosts)"| ext_traefik
  ext_traefik -->|"Route dashboard + API hosts"| sys_platform
  sys_platform -->|"SQL / migrations"| ext_postgres
  sys_platform -->|"Error telemetry"| ext_sentry
  sys_platform -->|"Outbound email"| ext_smtp
```

## L2 — Container diagram

```mermaid
flowchart TB
  subgraph boundary["sys:platform — Full-stack web application (Docker Compose stack)"]
    container_spa["container:spa<br/>Dashboard SPA (Vite, nginx)"]
    container_api["container:api<br/>HTTP API"]
    container_postgres["container:postgres<br/>Relational database"]
    container_prestart["container:prestart<br/>Prestart job"]
  end
  actor_end_user["actor:end_user<br/>End user"]
  ext_traefik["ext:traefik<br/>Traefik ingress"]
  ext_sentry["ext:sentry<br/>Sentry"]
  ext_smtp["ext:smtp<br/>SMTP / email delivery"]
  actor_end_user -->|"HTTPS (dashboard / API hosts)"| ext_traefik
  ext_traefik -->|"Route dashboard host"| container_spa
  ext_traefik -->|"Route API host"| container_api
  container_api -->|"SQL"| container_postgres
  container_prestart -->|"Migrations / startup"| container_postgres
  container_api -->|"Error telemetry"| ext_sentry
  container_api -->|"Outbound email"| ext_smtp
```

## L3 — Components (selected containers)

### container:spa

```mermaid
flowchart TB
  subgraph spa["%% SCOPE: urn:c4:container:container:spa"]
    spa_r["%% KIND: router<br/>component:spa_router<br/>Client router"]
    spa_c["%% KIND: integration<br/>component:spa_api_client<br/>OpenAPI HTTP client"]
  end
  spa_r -->|"UI navigation and data hooks"| spa_c
```

### container:api

```mermaid
flowchart TB
  subgraph api["%% SCOPE: urn:c4:container:container:api"]
    api_h["%% KIND: service_layer<br/>component:api_http_surface<br/>FastAPI route surface"]
    api_p["%% KIND: data_access<br/>component:api_persistence<br/>CRUD and ORM access"]
  end
  api_h -->|"Request handling"| api_p
```

### container:postgres

```mermaid
flowchart TB
  subgraph pg["%% SCOPE: urn:c4:container:container:postgres"]
    db_s["%% KIND: storage<br/>component:db_storage<br/>PostgreSQL server"]
  end
```

### container:prestart

```mermaid
flowchart TB
  subgraph pre["%% SCOPE: urn:c4:container:container:prestart"]
    pre_a["%% KIND: pipeline<br/>component:prestart_alembic<br/>Alembic migrations"]
    pre_seed["%% KIND: processing<br/>component:prestart_seed<br/>DB bootstrap scripts"]
  end
```

## Containment Map

```json
{
  "parentToChildren": {
    "sys:platform": [
      "container:spa",
      "container:api",
      "container:postgres",
      "container:prestart"
    ],
    "container:spa": [
      "component:spa_router",
      "component:spa_api_client"
    ],
    "container:api": [
      "component:api_http_surface",
      "component:api_persistence"
    ],
    "container:postgres": [
      "component:db_storage"
    ],
    "container:prestart": [
      "component:prestart_alembic",
      "component:prestart_seed"
    ]
  },
  "childToParent": {
    "container:spa": "sys:platform",
    "container:api": "sys:platform",
    "container:postgres": "sys:platform",
    "container:prestart": "sys:platform",
    "component:spa_router": "container:spa",
    "component:spa_api_client": "container:spa",
    "component:api_http_surface": "container:api",
    "component:api_persistence": "container:api",
    "component:db_storage": "container:postgres",
    "component:prestart_alembic": "container:prestart",
    "component:prestart_seed": "container:prestart"
  }
}
```
