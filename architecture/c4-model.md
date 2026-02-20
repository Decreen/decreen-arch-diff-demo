# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

Shows the Full Stack FastAPI Platform in its environment: the actors who use it
and the external systems it depends on.

```mermaid
graph TB
  %% L1 System Context — Full Stack FastAPI Platform

  subgraph users["Users"]
    end_user["End User
    [Person]
    Registers, logs in, manages own items"]
    admin_user["Administrator
    [Person]
    Manages all users, items, and system"]
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    fullstack_system["Full Stack FastAPI Platform
    [Software System]
    Web application for user and item management
    with JWT authentication and email notifications"]
  end

  subgraph external["External Services"]
    smtp_service["SMTP Email Service
    [External System]
    Delivers transactional emails
    (password recovery, account creation)"]
    sentry_service["Sentry
    [External System]
    Error monitoring and performance tracing"]
  end

  end_user -- "Uses web UI to
  manage items" --> fullstack_system
  admin_user -- "Administers users
  and system settings" --> fullstack_system
  fullstack_system -- "Sends emails via SMTP" --> smtp_service
  fullstack_system -- "Reports errors and traces" --> sentry_service
```

---

## L2: Container

Zooms into the Full Stack FastAPI Platform to show its deployable containers,
how they communicate, and how external actors and systems connect.

```mermaid
graph TB
  %% L2 Container — Full Stack FastAPI Platform

  subgraph users["Users"]
    end_user["End User
    [Person]"]
    admin_user["Administrator
    [Person]"]
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik
    [Container: Reverse Proxy / Go]
    Routes HTTPS traffic, TLS termination,
    load balancing, path-based routing"]

    spa["React SPA
    [Container: TypeScript / React / Nginx]
    Single-page application with TanStack Router,
    React Query, shadcn/ui, Tailwind CSS"]

    fastapi_backend["FastAPI Backend
    [Container: Python / FastAPI / Uvicorn]
    REST API with JWT auth, CRUD operations,
    email utilities, Alembic migrations"]

    pg["PostgreSQL
    [Container: PostgreSQL 18]
    Stores users, items,
    and application data"]

    adminer["Adminer
    [Container: PHP / Adminer]
    Web-based database administration UI"]
  end

  subgraph external["External Services"]
    smtp_service["SMTP Email Service
    [External System]"]
    sentry_service["Sentry
    [External System]"]
  end

  end_user -- "HTTPS" --> traefik
  admin_user -- "HTTPS" --> traefik
  traefik -- "HTTP :80
  dashboard.*" --> spa
  traefik -- "HTTP :8000
  api.*" --> fastapi_backend
  traefik -- "HTTP :8080
  adminer.*" --> adminer

  spa -- "REST API calls
  [JSON/HTTP]" --> fastapi_backend
  fastapi_backend -- "SQL queries
  [psycopg / PostgreSQL wire protocol]" --> pg
  fastapi_backend -- "Sends emails
  [SMTP]" --> smtp_service
  fastapi_backend -- "Error reports
  [HTTPS / Sentry SDK]" --> sentry_service
  adminer -- "SQL queries" --> pg
```

---

## L3: FastAPI Backend

Component diagram for the FastAPI Backend container, showing its internal
modules: route handlers, middleware, CRUD layer, security, and data access.

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_backend
  %% L3 Component — FastAPI Backend

  spa["React SPA
  [Container]"]
  pg["PostgreSQL
  [Container]"]
  smtp_service["SMTP Email Service
  [External System]"]
  sentry_service["Sentry
  [External System]"]

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    cors_mw["CORS Middleware
    [Component: Starlette CORSMiddleware]
    Validates cross-origin requests
    against allowed origins list"]

    subgraph api_router["API Router"]

      %% KIND: boundary
      auth_deps["Auth Dependencies
      [Component: FastAPI Depends]
      OAuth2 bearer token extraction,
      JWT decode, DB session injection,
      current-user / superuser guards"]

      %% KIND: router
      login_routes["Login Routes
      [Component: APIRouter /login]
      POST /login/access-token
      POST /password-recovery
      POST /reset-password"]

      %% KIND: router
      user_routes["User Routes
      [Component: APIRouter /users]
      GET/POST /users, GET/PATCH/DELETE /users/me
      POST /users/signup, PATCH/DELETE /users/:id"]

      %% KIND: router
      item_routes["Item Routes
      [Component: APIRouter /items]
      GET/POST /items
      GET/PUT/DELETE /items/:id
      with ownership checks"]

      %% KIND: router
      util_routes["Utility Routes
      [Component: APIRouter /utils]
      GET /utils/health-check
      POST /utils/test-email"]

    end

    %% KIND: data_access
    crud_layer["CRUD Layer
    [Component: Python Module — crud.py]
    create/read/update/delete operations
    for User and Item entities"]

    %% KIND: service_layer
    security_mod["Security Module
    [Component: Python Module — core/security.py]
    JWT token creation (HS256),
    password hashing (Argon2 + Bcrypt)"]

    %% KIND: service_layer
    config_mod["Configuration
    [Component: Pydantic Settings — core/config.py]
    Environment-based settings: DB URI,
    CORS origins, SMTP, secrets"]

    %% KIND: storage
    db_session["DB Engine
    [Component: SQLAlchemy / SQLModel — core/db.py]
    Connection pool via create_engine,
    session management, DB initialization"]

    %% KIND: integration
    email_utils["Email Utilities
    [Component: Python Module — utils.py]
    Jinja2 email templates, SMTP sending,
    password-reset token generation/verification"]

    %% KIND: storage
    models_layer["Data Models
    [Component: SQLModel — models.py]
    User, Item table definitions;
    Pydantic request/response schemas"]

  end

  spa -- "REST API calls" --> cors_mw
  cors_mw --> api_router

  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  util_routes --> auth_deps

  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_utils

  user_routes --> crud_layer
  user_routes --> security_mod
  user_routes --> email_utils

  item_routes --> crud_layer

  crud_layer --> models_layer
  crud_layer --> db_session
  crud_layer --> security_mod

  auth_deps --> security_mod
  auth_deps --> db_session
  auth_deps --> models_layer
  auth_deps --> config_mod

  security_mod --> config_mod
  db_session --> config_mod
  db_session -- "SQL [psycopg]" --> pg

  email_utils --> config_mod
  email_utils --> security_mod
  email_utils -- "SMTP" --> smtp_service

  fastapi_backend -. "Sentry SDK" .-> sentry_service
```

---

## L3: React SPA

Component diagram for the React SPA container, showing its internal structure:
routing, state management, API client, page views, and UI components.

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa
  %% L3 Component — React SPA

  fastapi_backend["FastAPI Backend
  [Container]"]
  end_user["End User
  [Person]"]
  admin_user["Administrator
  [Person]"]

  subgraph spa["React SPA"]

    %% KIND: router
    tanstack_router["TanStack Router
    [Component: @tanstack/react-router]
    File-based routing with code splitting:
    login, signup, recovery, layout routes"]

    %% KIND: service_layer
    query_client["React Query Client
    [Component: @tanstack/react-query]
    Server state management, caching,
    automatic error handling (401/403)"]

    %% KIND: integration
    api_client["OpenAPI Client SDK
    [Component: @hey-api/openapi-ts]
    Auto-generated typed HTTP client
    for all backend API endpoints"]

    %% KIND: service_layer
    auth_hooks["Auth Hooks
    [Component: React Hook — useAuth]
    Login/logout/signup mutations,
    token management in localStorage"]

    subgraph pages["Page Views"]

      %% KIND: boundary
      auth_views["Auth Views
      [Component: React Pages]
      Login, Signup,
      Password Recovery, Password Reset"]

      %% KIND: boundary
      admin_views["Admin Views
      [Component: React Pages]
      User management: list, add,
      edit, delete users (superuser only)"]

      %% KIND: boundary
      item_views["Item Views
      [Component: React Pages]
      Item management: list, add,
      edit, delete items"]

      %% KIND: boundary
      settings_views["Settings Views
      [Component: React Pages]
      User info, change password,
      delete account, appearance toggle"]

    end

    %% KIND: boundary
    ui_lib["UI Component Library
    [Component: shadcn/ui + Radix UI]
    Buttons, dialogs, forms, tables,
    sidebar, data tables, inputs"]

    %% KIND: service_layer
    theme_provider["Theme Provider
    [Component: next-themes]
    Dark/light mode toggle
    with localStorage persistence"]

  end

  end_user -- "Browses" --> tanstack_router
  admin_user -- "Browses" --> tanstack_router

  tanstack_router --> auth_views
  tanstack_router --> admin_views
  tanstack_router --> item_views
  tanstack_router --> settings_views

  auth_views --> auth_hooks
  auth_views --> ui_lib
  admin_views --> query_client
  admin_views --> ui_lib
  item_views --> query_client
  item_views --> ui_lib
  settings_views --> query_client
  settings_views --> auth_hooks
  settings_views --> ui_lib

  auth_hooks --> api_client
  query_client --> api_client

  pages --> theme_provider

  api_client -- "REST API
  [JSON/HTTP]" --> fastapi_backend
```

---

## Containment Map

Every subgraph from every diagram, with its direct children.
Indented entries indicate nesting (child-of-child).

```text
%% ── CONTAINMENT MAP ─────────────────────────────────────────────

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [end_user, admin_user]
system_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external         CONTAINS [smtp_service, sentry_service]

%% ── L2→L3 internal containment (FastAPI Backend) ────────────────
fastapi_backend  CONTAINS [cors_mw, api_router, crud_layer, security_mod, config_mod, db_session, email_utils, models_layer]
  api_router     CONTAINS [auth_deps, login_routes, user_routes, item_routes, util_routes]

%% ── L2→L3 internal containment (React SPA) ─────────────────────
spa              CONTAINS [tanstack_router, query_client, api_client, auth_hooks, pages, ui_lib, theme_provider]
  pages          CONTAINS [auth_views, admin_views, item_views, settings_views]
```
