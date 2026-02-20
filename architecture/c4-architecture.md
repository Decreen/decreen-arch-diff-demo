# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    user["End User
    <i>Browses dashboard, manages own items</i>"]
    admin_user["Administrator
    <i>Manages all users, items & system config</i>"]
  end

  subgraph fullstack_system["Full Stack FastAPI Platform"]
    system_node["Full Stack FastAPI System
    <i>Web application for user & item management
    with JWT auth, email recovery, admin panel</i>"]
  end

  subgraph external["External Systems"]
    smtp_server["SMTP Server
    <i>Email delivery (password recovery, notifications)</i>"]
    sentry["Sentry
    <i>Error monitoring & performance tracing</i>"]
  end

  user -->|"Uses web dashboard"| system_node
  admin_user -->|"Administers users & data"| system_node
  system_node -->|"Sends transactional emails via SMTP"| smtp_server
  system_node -->|"Reports errors & traces via HTTPS"| sentry
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    user["End User"]
    admin_user["Administrator"]
  end

  subgraph fullstack_system["Full Stack FastAPI Platform"]
    traefik["Traefik
    <i>Reverse Proxy / Load Balancer
    TLS termination, routing by Host header</i>"]
    spa["React SPA
    <i>TypeScript · Vite · TanStack Router
    Tailwind CSS · shadcn/ui</i>"]
    fastapi_backend["FastAPI Backend
    <i>Python REST API · SQLModel ORM
    JWT auth · Pydantic validation</i>"]
    pg["PostgreSQL 18
    <i>Relational database
    Users, Items tables</i>"]
    adminer["Adminer
    <i>Database administration UI</i>"]
  end

  subgraph external["External Systems"]
    smtp_server["SMTP Server
    <i>Email delivery</i>"]
    sentry["Sentry
    <i>Error monitoring</i>"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.* → port 80"| spa
  traefik -->|"api.* → port 8000"| fastapi_backend
  traefik -->|"adminer.* → port 8080"| adminer
  spa -->|"REST / JSON over HTTPS"| fastapi_backend
  fastapi_backend -->|"SQL via psycopg (TCP :5432)"| pg
  adminer -->|"SQL (TCP :5432)"| pg
  fastapi_backend -->|"SMTP"| smtp_server
  fastapi_backend -->|"HTTPS (Sentry SDK)"| sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi-backend

  spa["React SPA"]
  pg["PostgreSQL 18"]
  smtp_server["SMTP Server"]
  sentry["Sentry"]

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware"]
      cors_mw["CORS Middleware
      <i>Allows configured origins</i>"]
    end

    %% KIND: router
    subgraph routes["API Routes (/api/v1)"]
      login_routes["Login Routes
      <i>/login/access-token · /password-recovery
      /reset-password</i>"]
      users_routes["Users Routes
      <i>/users CRUD · /users/me
      /users/signup</i>"]
      items_routes["Items Routes
      <i>/items CRUD
      owner-scoped access</i>"]
      utils_routes["Utils Routes
      <i>/utils/health-check
      /utils/test-email</i>"]
    end

    %% KIND: service_layer
    subgraph deps["Dependencies"]
      auth_dep["Auth Dependency
      <i>OAuth2 bearer · JWT decode
      get_current_user · superuser guard</i>"]
      session_dep["Session Dependency
      <i>SQLModel Session per-request</i>"]
    end

    %% KIND: data_access
    crud_layer["CRUD Operations
    <i>create/update/authenticate user
    create item · get_user_by_email</i>"]

    %% KIND: storage
    models_layer["SQLModel Models
    <i>User · Item · Token
    Pydantic schemas (Create/Update/Public)</i>"]

    %% KIND: service_layer
    security_mod["Security Module
    <i>JWT HS256 creation · Argon2/Bcrypt
    password hashing & verification</i>"]

    %% KIND: service_layer
    config_mod["Configuration
    <i>Pydantic Settings · env-based
    DB DSN · CORS · SMTP · Sentry</i>"]

    %% KIND: integration
    email_utils["Email Utilities
    <i>Jinja2 HTML templates
    SMTP send · token generation</i>"]

    %% KIND: data_access
    db_engine["Database Engine
    <i>SQLAlchemy create_engine
    Alembic migrations · init_db</i>"]
  end

  spa -->|"REST / JSON"| cors_mw
  cors_mw --> login_routes
  cors_mw --> users_routes
  cors_mw --> items_routes
  cors_mw --> utils_routes

  login_routes --> auth_dep
  users_routes --> auth_dep
  items_routes --> auth_dep
  utils_routes --> auth_dep

  auth_dep --> security_mod
  auth_dep --> session_dep
  session_dep --> db_engine

  login_routes --> crud_layer
  users_routes --> crud_layer
  items_routes --> models_layer

  crud_layer --> models_layer
  crud_layer --> security_mod

  login_routes --> email_utils
  users_routes --> email_utils
  utils_routes --> email_utils

  email_utils -->|"SMTP"| smtp_server
  db_engine -->|"SQL via psycopg"| pg
  config_mod -.->|"DSN"| db_engine
  config_mod -.->|"Sentry SDK init"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  user["End User"]
  admin_user["Administrator"]
  fastapi_backend["FastAPI Backend"]

  subgraph spa["React SPA"]

    %% KIND: router
    tanstack_router["TanStack Router
    <i>File-based routing
    Auth guard via beforeLoad</i>"]

    %% KIND: service_layer
    subgraph views["Page Views"]
      login_view["Auth Pages
      <i>Login · Signup
      Password Recovery · Reset</i>"]
      admin_view["Admin Dashboard
      <i>User list · Add/Edit/Delete users
      Superuser only</i>"]
      items_view["Items Management
      <i>Item list · Add/Edit/Delete items
      DataTable with pagination</i>"]
      settings_view["User Settings
      <i>Profile info · Change password
      Delete account</i>"]
    end

    %% KIND: integration
    api_client["API Client
    <i>Auto-generated from OpenAPI spec
    ItemsService · LoginService
    UsersService · UtilsService</i>"]

    %% KIND: service_layer
    auth_hooks["Auth Hooks
    <i>useAuth · isLoggedIn
    JWT token in localStorage</i>"]

    %% KIND: service_layer
    query_layer["TanStack Query
    <i>Server-state cache
    Mutations & query invalidation</i>"]

    %% KIND: boundary
    subgraph ui_layer["UI Layer"]
      ui_lib["shadcn/ui Components
      <i>Radix primitives · Tailwind CSS
      Dialog · DataTable · Form · Sidebar</i>"]
      theme_provider["Theme Provider
      <i>next-themes · Dark / Light mode</i>"]
      sidebar_nav["Sidebar Navigation
      <i>AppSidebar · Main nav
      User menu · Footer</i>"]
    end
  end

  user -->|"HTTPS"| tanstack_router
  admin_user -->|"HTTPS"| tanstack_router

  tanstack_router --> login_view
  tanstack_router --> admin_view
  tanstack_router --> items_view
  tanstack_router --> settings_view

  login_view --> auth_hooks
  admin_view --> auth_hooks

  login_view --> query_layer
  admin_view --> query_layer
  items_view --> query_layer
  settings_view --> query_layer

  query_layer --> api_client
  auth_hooks --> api_client

  login_view --> ui_lib
  admin_view --> ui_lib
  items_view --> ui_lib
  settings_view --> ui_lib

  admin_view --> sidebar_nav
  items_view --> sidebar_nav
  settings_view --> sidebar_nav

  api_client -->|"REST / JSON"| fastapi_backend
```

---

## Containment Map

```text
%% ── L1 top-level groups ────────────────────────────────────────────────────
users              CONTAINS [user, admin_user]
fullstack_system   CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp_server, sentry]

%% ── L2 → L3 internal containment ───────────────────────────────────────────

%% FastAPI Backend (L3)
fastapi_backend    CONTAINS [middleware, routes, deps, crud_layer, models_layer, security_mod, config_mod, email_utils, db_engine]
  middleware       CONTAINS [cors_mw]
  routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  deps             CONTAINS [auth_dep, session_dep]

%% React SPA (L3)
spa                CONTAINS [tanstack_router, views, api_client, auth_hooks, query_layer, ui_layer]
  views            CONTAINS [login_view, admin_view, items_view, settings_view]
  ui_layer         CONTAINS [ui_lib, theme_provider, sidebar_nav]
```
