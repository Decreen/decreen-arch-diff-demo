# C4 Architecture Model — Full Stack FastAPI Platform

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for every container with internal structure.

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system-context:fastapi_platform

  subgraph users["Users"]
    user["<b>End User</b><br/><i>Manages personal items<br/>via the web dashboard</i>"]
    admin_user["<b>Administrator</b><br/><i>Manages users, system<br/>config, and all items</i>"]
  end

  subgraph fastapi_platform_boundary["Full Stack FastAPI Platform"]
    fastapi_platform["<b>Full Stack FastAPI Platform</b><br/><i>Web application for user<br/>and item management</i>"]
  end

  subgraph external_systems["External Systems"]
    smtp_server["<b>SMTP Server</b><br/><i>Email delivery<br/>(Mailgun / SES / Mailcatcher)</i>"]
    sentry["<b>Sentry</b><br/><i>Error tracking &amp;<br/>performance monitoring</i>"]
  end

  user -->|"Uses web dashboard"| fastapi_platform
  admin_user -->|"Administers platform"| fastapi_platform
  fastapi_platform -->|"Sends transactional emails"| smtp_server
  fastapi_platform -->|"Reports errors &amp; traces"| sentry
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fastapi_platform

  subgraph users["Users"]
    user["<b>End User</b>"]
    admin_user["<b>Administrator</b>"]
  end

  subgraph fastapi_platform_boundary["Full Stack FastAPI Platform"]
    traefik["<b>Traefik</b><br/><i>Reverse Proxy<br/>TLS termination, routing</i>"]
    spa["<b>React SPA</b><br/><i>Vite + React 19 + TypeScript<br/>Served by Nginx</i>"]
    fastapi_api["<b>FastAPI Backend</b><br/><i>Python 3.10 · FastAPI<br/>REST API on port 8000</i>"]
    pg["<b>PostgreSQL 18</b><br/><i>Relational database<br/>Users, Items</i>"]
    adminer["<b>Adminer</b><br/><i>Database admin UI</i>"]
  end

  subgraph external_systems["External Systems"]
    smtp_server["<b>SMTP Server</b><br/><i>Email delivery</i>"]
    sentry["<b>Sentry</b><br/><i>Error tracking</i>"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.*"| spa
  traefik -->|"api.*"| fastapi_api
  traefik -->|"adminer.*"| adminer
  spa -->|"JSON / HTTPS<br/>/api/v1/*"| fastapi_api
  fastapi_api -->|"SQL · psycopg"| pg
  adminer -->|"SQL"| pg
  fastapi_api -->|"SMTP"| smtp_server
  fastapi_api -->|"Sentry SDK"| sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_api

  traefik["<b>Traefik</b>"]
  spa["<b>React SPA</b>"]
  pg["<b>PostgreSQL 18</b>"]
  smtp_server["<b>SMTP Server</b>"]
  sentry["<b>Sentry</b>"]

  subgraph fastapi_api["FastAPI Backend"]

    subgraph mw["Middleware"]
      %% KIND: boundary
      cors_mw["<b>CORS Middleware</b><br/><i>Starlette CORSMiddleware<br/>Origin allowlist</i>"]
    end

    subgraph routes["API Routes · /api/v1"]
      %% KIND: router
      api_router["<b>API Router</b><br/><i>FastAPI APIRouter<br/>Prefix: /api/v1</i>"]
      %% KIND: router
      login_routes["<b>Login Routes</b><br/><i>/login/access-token<br/>/password-recovery/*<br/>/reset-password</i>"]
      %% KIND: router
      users_routes["<b>Users Routes</b><br/><i>/users/* · CRUD<br/>/users/me · /users/signup</i>"]
      %% KIND: router
      items_routes["<b>Items Routes</b><br/><i>/items/* · CRUD<br/>Owner-scoped access</i>"]
      %% KIND: router
      utils_routes["<b>Utils Routes</b><br/><i>/utils/health-check<br/>/utils/test-email</i>"]
    end

    subgraph services["Services"]
      %% KIND: service_layer
      auth_deps["<b>Auth Dependencies</b><br/><i>OAuth2PasswordBearer<br/>JWT decode · get_current_user</i>"]
      %% KIND: data_access
      crud_layer["<b>CRUD Operations</b><br/><i>create/update/get User<br/>create Item · authenticate</i>"]
      %% KIND: integration
      email_utils["<b>Email Utilities</b><br/><i>send_email · Jinja2 templates<br/>Password reset tokens</i>"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      core_config["<b>Configuration</b><br/><i>pydantic-settings<br/>Environment variables</i>"]
      %% KIND: service_layer
      security["<b>Security</b><br/><i>JWT creation · HS256<br/>Argon2 / Bcrypt hashing</i>"]
    end

    subgraph data_access["Data Access"]
      %% KIND: storage
      db_engine["<b>Database Engine</b><br/><i>SQLAlchemy create_engine<br/>Session management</i>"]
      %% KIND: data_access
      models_layer["<b>SQLModel Models</b><br/><i>User · Item tables<br/>Pydantic schemas</i>"]
    end

  end

  traefik -->|"HTTP :8000"| cors_mw
  spa -->|"JSON / HTTPS"| cors_mw
  cors_mw --> api_router
  api_router --> login_routes
  api_router --> users_routes
  api_router --> items_routes
  api_router --> utils_routes

  login_routes --> auth_deps
  login_routes --> crud_layer
  login_routes --> email_utils
  users_routes --> auth_deps
  users_routes --> crud_layer
  users_routes --> email_utils
  items_routes --> auth_deps
  items_routes --> crud_layer
  utils_routes --> email_utils

  auth_deps --> security
  auth_deps --> db_engine
  auth_deps --> models_layer
  crud_layer --> security
  crud_layer --> models_layer
  crud_layer --> db_engine
  email_utils --> core_config
  security --> core_config

  db_engine -->|"SQL · psycopg"| pg
  email_utils -->|"SMTP"| smtp_server
  core_config -.->|"Sentry DSN"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  user["<b>End User</b>"]
  admin_user["<b>Administrator</b>"]
  traefik["<b>Traefik</b>"]
  fastapi_api["<b>FastAPI Backend</b>"]

  subgraph spa["React SPA"]

    subgraph routing["Routing &amp; State"]
      %% KIND: router
      tanstack_router["<b>TanStack Router</b><br/><i>File-based routing<br/>Auth guards · beforeLoad</i>"]
      %% KIND: service_layer
      query_client["<b>TanStack Query</b><br/><i>Server state cache<br/>QueryClient · MutationCache</i>"]
    end

    subgraph features["Features / Pages"]
      %% KIND: boundary
      auth_pages["<b>Auth Pages</b><br/><i>Login · Signup<br/>Recover / Reset Password</i>"]
      %% KIND: boundary
      dashboard_page["<b>Dashboard</b><br/><i>Home page<br/>Overview &amp; stats</i>"]
      %% KIND: boundary
      items_features["<b>Items Management</b><br/><i>CRUD table &amp; forms<br/>Add / Edit / Delete Item</i>"]
      %% KIND: boundary
      admin_features["<b>Admin Panel</b><br/><i>User management<br/>Superuser only</i>"]
      %% KIND: boundary
      settings_features["<b>User Settings</b><br/><i>Profile info · Password<br/>Delete account</i>"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["<b>OpenAPI Client</b><br/><i>@hey-api/openapi-ts<br/>Axios-based SDK</i>"]
      %% KIND: service_layer
      auth_hook["<b>Auth Hook</b><br/><i>useAuth · login/logout<br/>localStorage token</i>"]
    end

    subgraph ui["UI Layer"]
      %% KIND: boundary
      common_ui["<b>Common Components</b><br/><i>AuthLayout · DataTable<br/>ErrorComponent · NotFound</i>"]
      %% KIND: boundary
      sidebar_ui["<b>Sidebar</b><br/><i>AppSidebar · Navigation<br/>User menu</i>"]
      %% KIND: boundary
      ui_primitives["<b>UI Primitives</b><br/><i>Radix + shadcn/ui<br/>Button · Dialog · Form · Table</i>"]
      %% KIND: service_layer
      theme_provider["<b>Theme Provider</b><br/><i>Dark / Light mode<br/>localStorage persistence</i>"]
    end

  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"Serves static assets"| tanstack_router

  tanstack_router --> auth_pages
  tanstack_router --> dashboard_page
  tanstack_router --> items_features
  tanstack_router --> admin_features
  tanstack_router --> settings_features

  auth_pages --> auth_hook
  auth_pages --> common_ui
  dashboard_page --> common_ui
  items_features --> query_client
  items_features --> common_ui
  admin_features --> query_client
  admin_features --> common_ui
  settings_features --> query_client
  settings_features --> common_ui

  auth_hook --> api_client
  query_client --> api_client

  common_ui --> ui_primitives
  sidebar_ui --> ui_primitives
  common_ui --> theme_provider

  api_client -->|"JSON / HTTPS<br/>/api/v1/*"| fastapi_api
```

---

## L3: Traefik

```mermaid
graph TB
  %% SCOPE: urn:c4:container:traefik

  user["<b>End User</b>"]
  admin_user["<b>Administrator</b>"]
  spa["<b>React SPA</b>"]
  fastapi_api["<b>FastAPI Backend</b>"]
  adminer["<b>Adminer</b>"]

  subgraph traefik["Traefik Reverse Proxy"]
    %% KIND: boundary
    tls_termination["<b>TLS Termination</b><br/><i>Let's Encrypt<br/>certresolver: le</i>"]
    %% KIND: router
    http_router["<b>HTTP Router</b><br/><i>Host-based routing<br/>HTTPS redirect middleware</i>"]
    %% KIND: router
    dashboard_route["<b>dashboard.* Route</b><br/><i>→ Frontend :80</i>"]
    %% KIND: router
    api_route["<b>api.* Route</b><br/><i>→ Backend :8000</i>"]
    %% KIND: router
    adminer_route["<b>adminer.* Route</b><br/><i>→ Adminer :8080</i>"]
  end

  user -->|"HTTPS :443"| tls_termination
  admin_user -->|"HTTPS :443"| tls_termination
  tls_termination --> http_router
  http_router --> dashboard_route
  http_router --> api_route
  http_router --> adminer_route
  dashboard_route -->|":80"| spa
  api_route -->|":8000"| fastapi_api
  adminer_route -->|":8080"| adminer
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% Every subgraph and node across all levels.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users                     CONTAINS [user, admin_user]
fastapi_platform_boundary CONTAINS [traefik, spa, fastapi_api, pg, adminer]
external_systems          CONTAINS [smtp_server, sentry]

%% ── L2→L3 internal containment ──────────────────────────────────

%% FastAPI Backend (fastapi_api) ───────────────────────────────────
fastapi_api CONTAINS [mw, routes, services, core, data_access]
  mw          CONTAINS [cors_mw]
  routes      CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes]
  services    CONTAINS [auth_deps, crud_layer, email_utils]
  core        CONTAINS [core_config, security]
  data_access CONTAINS [db_engine, models_layer]

%% React SPA (spa) ─────────────────────────────────────────────────
spa CONTAINS [routing, features, services_layer, ui]
  routing        CONTAINS [tanstack_router, query_client]
  features       CONTAINS [auth_pages, dashboard_page, items_features, admin_features, settings_features]
  services_layer CONTAINS [api_client, auth_hook]
  ui             CONTAINS [common_ui, sidebar_ui, ui_primitives, theme_provider]

%% Traefik (traefik) ───────────────────────────────────────────────
traefik CONTAINS [tls_termination, http_router, dashboard_route, api_route, adminer_route]
```

---

## ID Cross-Reference

| Stable ID | L1 | L2 | L3 Scope | Description |
|---|---|---|---|---|
| `user` | ✓ | ✓ | spa | End User actor |
| `admin_user` | ✓ | ✓ | spa | Administrator actor |
| `fastapi_platform` | ✓ | — | — | System node (L1 only) |
| `fastapi_platform_boundary` | ✓ | ✓ | — | System boundary subgraph |
| `traefik` | — | ✓ | fastapi_api, spa, traefik | Reverse proxy container / L3 scope |
| `spa` | — | ✓ | fastapi_api, spa | React SPA container / L3 scope |
| `fastapi_api` | — | ✓ | fastapi_api | FastAPI backend container / L3 scope |
| `pg` | — | ✓ | fastapi_api | PostgreSQL database |
| `adminer` | — | ✓ | traefik | DB admin UI |
| `smtp_server` | ✓ | ✓ | fastapi_api | External SMTP server |
| `sentry` | ✓ | ✓ | fastapi_api | External monitoring |
