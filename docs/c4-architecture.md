# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:platform

  subgraph users["Users"]
    user["End User
    <i>Person</i>
    Browses dashboard, manages own items and profile"]
    admin_user["Administrator
    <i>Person</i>
    Manages all users and system configuration"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    platform["Full Stack FastAPI Platform
    <i>Software System</i>
    Web application providing user management,
    item CRUD, and authentication"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server
    <i>External System</i>
    Delivers transactional emails"]
    sentry["Sentry
    <i>External System</i>
    Error tracking and performance monitoring"]
    letsencrypt["Let's Encrypt
    <i>External System</i>
    Automated TLS certificate provisioning"]
  end

  user -->|"Uses web application"| platform
  admin_user -->|"Manages users via web application"| platform
  platform -->|"Sends password-reset and welcome emails"| smtp
  platform -->|"Reports errors and traces"| sentry
  platform -->|"Obtains TLS certificates"| letsencrypt
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:platform

  subgraph users["Users"]
    user["End User
    <i>Person</i>"]
    admin_user["Administrator
    <i>Person</i>"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik
    <i>Container: Reverse Proxy</i>
    Routes HTTP/HTTPS traffic,
    TLS termination via Let's Encrypt"]

    spa["React SPA
    <i>Container: React 19 · TypeScript · Vite 7</i>
    Single-page application served by Nginx"]

    fastapi_api["FastAPI Backend
    <i>Container: Python · FastAPI · SQLModel</i>
    REST API — authentication, users, items"]

    pg["PostgreSQL
    <i>Container: PostgreSQL 18</i>
    Stores users, items, and application state"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server
    <i>External System</i>"]
    sentry["Sentry
    <i>External System</i>"]
    letsencrypt["Let's Encrypt
    <i>External System</i>"]
  end

  user -->|"Browses"| traefik
  admin_user -->|"Manages"| traefik
  traefik -->|"Serves static assets"| spa
  traefik -->|"Proxies /api/v1/* requests"| fastapi_api
  spa -->|"API calls · JSON over HTTPS"| fastapi_api
  fastapi_api -->|"Reads / writes · psycopg"| pg
  fastapi_api -->|"Sends transactional emails"| smtp
  fastapi_api -->|"Reports errors · sentry-sdk"| sentry
  traefik -->|"ACME challenge · cert renewal"| letsencrypt
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_api

  spa["React SPA
  <i>Container</i>"]

  subgraph fastapi_api["FastAPI Backend"]

    %% KIND: boundary
    cors_mw["CORS Middleware
    <i>Component: Starlette CORSMiddleware</i>
    Enforces allowed origins"]

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes
      <i>Component: /login/*</i>
      OAuth2 token, password recovery and reset"]
      %% KIND: router
      users_routes["Users Routes
      <i>Component: /users/*</i>
      User CRUD, profile, signup"]
      %% KIND: router
      items_routes["Items Routes
      <i>Component: /items/*</i>
      Item CRUD with ownership"]
      %% KIND: router
      utils_routes["Utils Routes
      <i>Component: /utils/*</i>
      Health check, test email"]
    end

    subgraph deps["Dependencies"]
      %% KIND: boundary
      auth_dep["Auth Dependency
      <i>Component: OAuth2PasswordBearer</i>
      JWT validation, resolves current user"]
      %% KIND: data_access
      db_dep["DB Session Dependency
      <i>Component: SessionDep</i>
      Yields SQLModel Session per request"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      config["Config
      <i>Component: Pydantic Settings</i>
      Loads env vars, validates secrets"]
      %% KIND: service_layer
      security["Security
      <i>Component: pwdlib · PyJWT</i>
      JWT creation, Argon2/Bcrypt password hashing"]
      %% KIND: data_access
      db_engine["DB Engine
      <i>Component: SQLAlchemy create_engine</i>
      Connection pool, database initialisation"]
    end

    %% KIND: data_access
    crud["CRUD Layer
    <i>Component: crud.py</i>
    Data-access functions for User and Item"]

    %% KIND: storage
    models["Models
    <i>Component: SQLModel entities</i>
    User, Item, Token schemas and DB tables"]

    %% KIND: integration
    email_utils["Email Utils
    <i>Component: utils.py</i>
    Jinja2 templates, SMTP send, token generation"]

  end

  pg["PostgreSQL
  <i>Container</i>"]
  smtp["SMTP Server
  <i>External System</i>"]
  sentry["Sentry
  <i>External System</i>"]

  spa -->|"JSON / HTTPS"| cors_mw
  cors_mw --> routes
  routes --> deps
  routes --> crud
  routes --> email_utils
  auth_dep --> security
  auth_dep --> models
  db_dep --> db_engine
  crud --> models
  crud --> db_dep
  db_engine -->|"psycopg"| pg
  email_utils -->|"SMTP"| smtp
  config -.->|"SENTRY_DSN"| sentry
  security --> config
  db_engine --> config
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  traefik["Traefik
  <i>Container</i>"]

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      router["TanStack Router
      <i>Component: File-based routing</i>
      Route tree, guards, lazy loading"]
      %% KIND: boundary
      auth_pages["Auth Pages
      <i>Component: Login · Signup · Recovery · Reset</i>
      Public authentication flows"]
      %% KIND: boundary
      dashboard_page["Dashboard Page
      <i>Component: _layout/index</i>
      Welcome screen with user info"]
      %% KIND: boundary
      items_page["Items Page
      <i>Component: _layout/items</i>
      Item CRUD with DataTable"]
      %% KIND: boundary
      admin_page["Admin Page
      <i>Component: _layout/admin</i>
      User management — superuser only"]
      %% KIND: boundary
      settings_page["Settings Page
      <i>Component: _layout/settings</i>
      Profile, password, account deletion"]
    end

    subgraph state["State Management"]
      %% KIND: service_layer
      query_client["TanStack Query
      <i>Component: QueryClient</i>
      Server-state cache, background refetch,
      global 401/403 error handling"]
      %% KIND: service_layer
      auth_hook["Auth Hook
      <i>Component: useAuth</i>
      Login, logout, signup, current-user query"]
    end

    %% KIND: integration
    api_client["API Client
    <i>Component: @hey-api/openapi-ts · Axios</i>
    Generated type-safe HTTP client"]

    subgraph ui["UI Components"]
      %% KIND: boundary
      layout_components["Layout
      <i>Component: AppSidebar · Main · Footer</i>
      Collapsible sidebar, responsive shell"]
      %% KIND: service_layer
      data_table["DataTable
      <i>Component: TanStack Table</i>
      Sortable, paginated, row-action table"]
      %% KIND: boundary
      forms["Forms
      <i>Component: react-hook-form · Zod</i>
      Validated form fields with error display"]
      %% KIND: service_layer
      theme_provider["Theme Provider
      <i>Component: ThemeProvider</i>
      Dark / light / system mode toggle"]
    end

  end

  fastapi_api["FastAPI Backend
  <i>Container</i>"]

  traefik -->|"Serves static assets"| spa
  router --> auth_pages
  router --> dashboard_page
  router --> items_page
  router --> admin_page
  router --> settings_page
  items_page --> data_table
  admin_page --> data_table
  auth_pages --> forms
  settings_page --> forms
  items_page --> forms
  query_client --> api_client
  auth_hook --> api_client
  api_client -->|"JSON / HTTPS"| fastapi_api
  dashboard_page --> auth_hook
  auth_pages --> auth_hook
  settings_page --> auth_hook
```

---

## L3: Traefik

```mermaid
graph TB
  %% SCOPE: urn:c4:container:traefik

  user["End User
  <i>Person</i>"]
  admin_user["Administrator
  <i>Person</i>"]

  subgraph traefik["Traefik"]

    %% KIND: router
    entrypoints["Entrypoints
    <i>Component: HTTP :80 · HTTPS :443</i>
    Listener ports, HTTP→HTTPS redirect"]

    %% KIND: router
    routers["Routers
    <i>Component: Host-based rules</i>
    dashboard.domain → SPA,
    api.domain → Backend"]

    %% KIND: service_layer
    tls_resolver["TLS Resolver
    <i>Component: Let's Encrypt ACME</i>
    Automatic certificate provisioning"]

    %% KIND: service_layer
    middlewares["Middlewares
    <i>Component: Redirect, BasicAuth</i>
    HTTPS redirect, dashboard auth"]

  end

  spa["React SPA
  <i>Container</i>"]
  fastapi_api["FastAPI Backend
  <i>Container</i>"]
  letsencrypt["Let's Encrypt
  <i>External System</i>"]

  user -->|"HTTPS"| entrypoints
  admin_user -->|"HTTPS"| entrypoints
  entrypoints --> middlewares
  middlewares --> routers
  routers -->|"dashboard.* → frontend"| spa
  routers -->|"api.* → backend:8000"| fastapi_api
  tls_resolver -->|"ACME challenge"| letsencrypt
  entrypoints --> tls_resolver
```

---

## L3: PostgreSQL

```mermaid
graph TB
  %% SCOPE: urn:c4:container:pg

  fastapi_api["FastAPI Backend
  <i>Container</i>"]

  subgraph pg["PostgreSQL"]

    %% KIND: storage
    user_table["User Table
    <i>Component: public.user</i>
    id (UUID), email, hashed_password,
    is_active, is_superuser, full_name, created_at"]

    %% KIND: storage
    item_table["Item Table
    <i>Component: public.item</i>
    id (UUID), title, description,
    owner_id (FK → user), created_at"]

    %% KIND: data_access
    alembic_versions["Alembic Versions
    <i>Component: alembic_version</i>
    Migration tracking table"]

  end

  fastapi_api -->|"psycopg"| user_table
  fastapi_api -->|"psycopg"| item_table
  fastapi_api -->|"alembic upgrade"| alembic_versions
  item_table -->|"FK owner_id CASCADE"| user_table
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi_api, pg]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 → L3 internal containment ────────────────────────────────

%% FastAPI Backend
fastapi_api  CONTAINS [cors_mw, routes, deps, core, crud, models, email_utils]
  routes     CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  deps       CONTAINS [auth_dep, db_dep]
  core       CONTAINS [config, security, db_engine]

%% React SPA
spa          CONTAINS [routing, state, api_client, ui]
  routing    CONTAINS [router, auth_pages, dashboard_page, items_page, admin_page, settings_page]
  state      CONTAINS [query_client, auth_hook]
  ui         CONTAINS [layout_components, data_table, forms, theme_provider]

%% Traefik
traefik      CONTAINS [entrypoints, routers, tls_resolver, middlewares]

%% PostgreSQL
pg           CONTAINS [user_table, item_table, alembic_versions]
```
