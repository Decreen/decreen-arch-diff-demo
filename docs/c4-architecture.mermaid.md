# C4 Architecture — Full Stack FastAPI Project

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack_fastapi
graph TB

  subgraph users["Users"]
    user["End User
    <i>Browser-based user who
    manages items via dashboard</i>"]
    admin_user["Admin User
    <i>Superuser who manages
    users and platform config</i>"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    platform["Full Stack FastAPI Platform
    <i>Web application for user
    and item management with
    JWT auth and email recovery</i>"]
  end

  subgraph external["External Services"]
    smtp["SMTP Server
    <i>Email delivery service</i>"]
    sentry["Sentry
    <i>Error monitoring and
    alerting platform</i>"]
    letsencrypt["Let's Encrypt
    <i>Automated TLS certificate
    authority</i>"]
  end

  user -->|"Uses web dashboard"| platform
  admin_user -->|"Manages users and settings"| platform
  platform -->|"Sends transactional emails via SMTP"| smtp
  platform -->|"Reports errors via SDK"| sentry
  platform -->|"Obtains TLS certificates via ACME"| letsencrypt
```

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fullstack_fastapi
graph TB

  user["End User"]
  admin_user["Admin User"]

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik
    <i>Reverse Proxy / Load Balancer
    Traefik 3.6, HTTPS termination</i>"]
    spa["React SPA
    <i>Single-Page Application
    React 19, Vite 7, TypeScript
    served by Nginx</i>"]
    fastapi_backend["FastAPI Backend
    <i>REST API Server
    Python, FastAPI, Uvicorn
    4 workers</i>"]
    pg["PostgreSQL
    <i>Relational Database
    PostgreSQL 18</i>"]
    adminer["Adminer
    <i>Database Admin UI
    Web-based SQL client</i>"]
  end

  subgraph external["External Services"]
    smtp["SMTP Server
    <i>Email delivery service</i>"]
    sentry["Sentry
    <i>Error monitoring</i>"]
    letsencrypt["Let's Encrypt
    <i>TLS certificate authority</i>"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"Routes dashboard.*"| spa
  traefik -->|"Routes api.*"| fastapi_backend
  traefik -->|"Routes adminer.*"| adminer
  spa -->|"REST API calls /api/v1/*"| fastapi_backend
  fastapi_backend -->|"SQL via SQLModel/psycopg"| pg
  adminer -->|"SQL queries"| pg
  fastapi_backend -->|"Sends emails via SMTP"| smtp
  fastapi_backend -->|"Error reports via SDK"| sentry
  traefik -->|"ACME TLS challenge"| letsencrypt
```

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
graph TB

  spa["React SPA"]
  pg["PostgreSQL"]
  smtp["SMTP Server"]
  sentry["Sentry"]

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware
      <i>Allows cross-origin requests
      from configured frontend origins</i>"]
      %% KIND: integration
      sentry_sdk["Sentry Integration
      <i>FastAPI SDK integration
      for error capture</i>"]
    end

    subgraph auth["Auth Dependencies"]
      %% KIND: boundary
      auth_deps["JWT Auth Dependencies
      <i>Token validation, current-user
      extraction, superuser checks</i>"]
    end

    subgraph routes["API Routes /api/v1"]
      %% KIND: router
      login_routes["Login Routes
      <i>/login/access-token
      /password-recovery/*
      /reset-password/</i>"]
      %% KIND: router
      users_routes["Users Routes
      <i>/users/ CRUD, /users/signup
      /users/me profile management</i>"]
      %% KIND: router
      items_routes["Items Routes
      <i>/items/ CRUD with
      ownership enforcement</i>"]
      %% KIND: router
      utils_routes["Utils Routes
      <i>/utils/health-check
      /utils/test-email</i>"]
      %% KIND: router
      private_routes["Private Routes
      <i>/private/users/ dev-only
      user creation endpoint</i>"]
    end

    subgraph services["Service Layer"]
      %% KIND: service_layer
      crud_layer["CRUD Operations
      <i>User and Item creation,
      authentication, updates</i>"]
      %% KIND: service_layer
      email_utils["Email Utilities
      <i>Jinja2 + MJML templates
      for transactional emails</i>"]
    end

    subgraph core["Core Modules"]
      %% KIND: service_layer
      security_mod["Security Module
      <i>JWT token creation/verification
      Argon2 + Bcrypt hashing</i>"]
      %% KIND: service_layer
      config_mod["Config Module
      <i>Pydantic Settings
      environment-based config</i>"]
    end

    subgraph data_access["Data Access Layer"]
      %% KIND: data_access
      models_layer["SQLModel Models
      <i>User and Item tables
      with Pydantic schemas</i>"]
      %% KIND: data_access
      db_engine["Database Engine
      <i>SQLModel/SQLAlchemy engine
      and session management</i>"]
      %% KIND: data_access
      alembic["Alembic Migrations
      <i>Schema versioning
      5 migration revisions</i>"]
    end

  end

  spa -->|"REST /api/v1/*"| cors_mw
  cors_mw --> auth_deps
  auth_deps --> login_routes
  auth_deps --> users_routes
  auth_deps --> items_routes
  auth_deps --> utils_routes
  auth_deps --> private_routes
  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_utils
  users_routes --> crud_layer
  users_routes --> security_mod
  items_routes --> crud_layer
  utils_routes --> email_utils
  crud_layer --> models_layer
  email_utils --> config_mod
  security_mod --> config_mod
  models_layer --> db_engine
  db_engine -->|"SQL via psycopg"| pg
  alembic -->|"DDL migrations"| pg
  email_utils -->|"SMTP"| smtp
  sentry_sdk -.->|"Error reports"| sentry
```

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

  user["End User"]
  fastapi_backend["FastAPI Backend"]

  subgraph spa["React SPA"]

    subgraph routing["Routing Layer"]
      %% KIND: router
      router["TanStack Router
      <i>File-based routing with
      code splitting and guards</i>"]
    end

    subgraph state["State Management"]
      %% KIND: service_layer
      auth_hook["Auth Hook
      <i>useAuth — login, logout
      token lifecycle in localStorage</i>"]
      %% KIND: service_layer
      query_client["TanStack Query
      <i>Server-state caching
      queries and mutations</i>"]
    end

    subgraph api_layer["API Communication"]
      %% KIND: integration
      api_client["OpenAPI Client
      <i>Auto-generated service classes
      Axios with Bearer tokens</i>"]
    end

    subgraph features["Feature Modules"]
      %% KIND: boundary
      auth_pages["Auth Pages
      <i>Login, Signup, Password
      Recovery and Reset</i>"]
      %% KIND: boundary
      dashboard_page["Dashboard
      <i>Welcome page with
      current-user greeting</i>"]
      %% KIND: boundary
      items_feature["Items Management
      <i>DataTable, CRUD forms
      with ownership display</i>"]
      %% KIND: boundary
      admin_feature["Admin Panel
      <i>User management table
      superuser-only access</i>"]
      %% KIND: boundary
      settings_feature["User Settings
      <i>Profile editing, password
      change, account deletion</i>"]
    end

    subgraph ui["UI Layer"]
      %% KIND: boundary
      ui_components["shadcn/ui Components
      <i>Radix primitives, DataTable
      Form, Dialog, etc.</i>"]
      %% KIND: boundary
      theme_provider["Theme Provider
      <i>Light / Dark / System
      mode with localStorage</i>"]
      %% KIND: boundary
      sidebar["Sidebar Navigation
      <i>Collapsible, role-based
      navigation links</i>"]
    end

  end

  user -->|"Browser HTTPS"| router
  router --> auth_pages
  router --> dashboard_page
  router --> items_feature
  router --> admin_feature
  router --> settings_feature
  auth_pages --> auth_hook
  dashboard_page --> query_client
  items_feature --> query_client
  admin_feature --> query_client
  settings_feature --> query_client
  auth_hook --> api_client
  query_client --> api_client
  api_client -->|"REST /api/v1/*"| fastapi_backend
  auth_pages --> ui_components
  dashboard_page --> ui_components
  items_feature --> ui_components
  admin_feature --> ui_components
  settings_feature --> ui_components
  dashboard_page --> sidebar
  items_feature --> sidebar
  admin_feature --> sidebar
  settings_feature --> sidebar
  ui_components --> theme_provider
```

## L3: Traefik Reverse Proxy

```mermaid
%% SCOPE: urn:c4:container:traefik
graph TB

  user["End User"]
  admin_user["Admin User"]
  spa["React SPA"]
  fastapi_backend["FastAPI Backend"]
  adminer["Adminer"]
  letsencrypt["Let's Encrypt"]

  subgraph traefik["Traefik"]

    %% KIND: router
    entrypoints["Entrypoints
    <i>HTTP :80 and HTTPS :443
    with automatic redirect</i>"]
    %% KIND: router
    routers["Host Routers
    <i>dashboard.* → SPA
    api.* → Backend
    adminer.* → Adminer</i>"]
    %% KIND: service_layer
    tls_resolver["TLS Resolver
    <i>ACME Let's Encrypt
    certificate management</i>"]
    %% KIND: service_layer
    traefik_dashboard["Traefik Dashboard
    <i>Admin UI at traefik.*
    HTTP Basic Auth protected</i>"]

  end

  user -->|"HTTPS"| entrypoints
  admin_user -->|"HTTPS"| entrypoints
  entrypoints --> routers
  entrypoints --> tls_resolver
  routers -->|"dashboard.*"| spa
  routers -->|"api.*"| fastapi_backend
  routers -->|"adminer.*"| adminer
  tls_resolver -->|"ACME challenge"| letsencrypt
  entrypoints --> traefik_dashboard
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [platform]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 container level ──────────────────────────────────────────
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L3: fastapi_backend internals ───────────────────────────────
fastapi_backend    CONTAINS [middleware, auth, routes, services, core, data_access]
  middleware       CONTAINS [cors_mw, sentry_sdk]
  auth             CONTAINS [auth_deps]
  routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes, private_routes]
  services         CONTAINS [crud_layer, email_utils]
  core             CONTAINS [security_mod, config_mod]
  data_access      CONTAINS [models_layer, db_engine, alembic]

%% ── L3: spa internals ──────────────────────────────────────────
spa                CONTAINS [routing, state, api_layer, features, ui]
  routing          CONTAINS [router]
  state            CONTAINS [auth_hook, query_client]
  api_layer        CONTAINS [api_client]
  features         CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
  ui               CONTAINS [ui_components, theme_provider, sidebar]

%% ── L3: traefik internals ──────────────────────────────────────
traefik            CONTAINS [entrypoints, routers, tls_resolver, traefik_dashboard]
```
