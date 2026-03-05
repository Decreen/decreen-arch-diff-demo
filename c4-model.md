# C4 Architecture Model — Full Stack FastAPI Project

---

## L1: System Context

```mermaid
%% L1 – System Context
%% SCOPE: urn:c4:system:platform
graph TB

  subgraph users["Users"]
    end_user["👤 End User<br/><i>Regular application user<br/>who manages personal items</i>"]
    admin_user["👤 Administrator<br/><i>Superuser who manages<br/>all users and system data</i>"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    platform["📦 Full Stack FastAPI Platform<br/><i>Web application providing user<br/>registration, authentication, and<br/>item management capabilities</i>"]
  end

  subgraph external["External Services"]
    smtp["📧 SMTP Provider<br/><i>Email delivery service<br/>(Mailgun / SendGrid / etc.)</i>"]
    sentry["📊 Sentry<br/><i>Error monitoring and<br/>performance tracking</i>"]
  end

  end_user -->|"Uses via browser<br/>[HTTPS]"| platform
  admin_user -->|"Manages users & config<br/>[HTTPS]"| platform
  platform -->|"Sends transactional emails<br/>[SMTP/TLS]"| smtp
  platform -->|"Reports errors & traces<br/>[HTTPS]"| sentry
```

---

## L2: Container

```mermaid
%% L2 – Container
%% SCOPE: urn:c4:system:platform
graph TB

  subgraph users["Users"]
    end_user["👤 End User"]
    admin_user["👤 Administrator"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["🔀 Traefik Reverse Proxy<br/><i>[Docker: traefik:3.6]<br/>TLS termination, routing,<br/>Let's Encrypt certificates</i>"]
    spa["🌐 React SPA<br/><i>[React 19 · Vite · Nginx]<br/>Single-page application<br/>with TanStack Router & Query</i>"]
    fastapi_api["⚙️ FastAPI Backend<br/><i>[Python · FastAPI · Uvicorn]<br/>REST API server<br/>with JWT authentication</i>"]
    pg["🗄️ PostgreSQL<br/><i>[PostgreSQL 18]<br/>Relational database<br/>for users and items</i>"]
  end

  subgraph external["External Services"]
    smtp["📧 SMTP Provider"]
    sentry["📊 Sentry"]
  end

  end_user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.domain → :80"| spa
  traefik -->|"api.domain → :8000"| fastapi_api
  spa -->|"REST API calls<br/>[JSON/HTTPS]"| fastapi_api
  fastapi_api -->|"Reads & writes data<br/>[SQL via psycopg]"| pg
  fastapi_api -->|"Sends transactional emails<br/>[SMTP/TLS]"| smtp
  fastapi_api -->|"Error & performance data<br/>[HTTPS]"| sentry
```

---

## L3: FastAPI Backend

```mermaid
%% L3 – FastAPI Backend Components
%% SCOPE: urn:c4:container:fastapi_api
graph TB

  spa["🌐 React SPA"]
  pg["🗄️ PostgreSQL"]
  smtp["📧 SMTP Provider"]
  sentry["📊 Sentry"]

  subgraph fastapi_api["FastAPI Backend"]

    subgraph middleware["Middleware & Dependencies"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware<br/>cross-origin request handling</i>"]
      %% KIND: boundary
      auth_deps["Auth Dependencies<br/><i>OAuth2 Bearer + JWT<br/>token validation</i>"]
      %% KIND: boundary
      superuser_guard["Superuser Guard<br/><i>is_superuser privilege<br/>enforcement</i>"]
      %% KIND: data_access
      db_session["DB Session Provider<br/><i>SQLModel Session<br/>dependency injection</i>"]
    end

    subgraph routes["API Routes · /api/v1"]
      %% KIND: router
      login_routes["Login Routes<br/><i>/login/* — access tokens,<br/>password recovery & reset</i>"]
      %% KIND: router
      user_routes["User Routes<br/><i>/users/* — CRUD, signup,<br/>profile, admin management</i>"]
      %% KIND: router
      item_routes["Item Routes<br/><i>/items/* — create, read,<br/>update, delete items</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>/utils/* — health check,<br/>test email</i>"]
    end

    subgraph services["Business & Data Layer"]
      %% KIND: data_access
      crud_layer["CRUD Functions<br/><i>create_user, authenticate,<br/>create_item, update_user</i>"]
      %% KIND: storage
      models_layer["SQLModel Models<br/><i>User, Item, Token<br/>entities & Pydantic schemas</i>"]
    end

    subgraph core["Core Infrastructure"]
      %% KIND: service_layer
      security_core["Security Module<br/><i>JWT creation (HS256),<br/>Argon2 & Bcrypt hashing</i>"]
      %% KIND: service_layer
      config_core["Configuration<br/><i>Pydantic Settings<br/>(env-driven config)</i>"]
      %% KIND: data_access
      db_engine["Database Engine<br/><i>SQLAlchemy engine<br/>+ connection pool</i>"]
    end

    subgraph utilities["Utilities"]
      %% KIND: integration
      email_utils["Email Service<br/><i>SMTP sending via emails lib,<br/>Jinja2 + MJML templates</i>"]
      %% KIND: storage
      alembic_mig["Alembic Migrations<br/><i>Schema versioning<br/>& database evolution</i>"]
    end

  end

  spa -->|"HTTP requests"| cors_mw
  cors_mw --> login_routes
  cors_mw --> user_routes
  cors_mw --> item_routes
  cors_mw --> utils_routes

  login_routes -.->|"uses"| auth_deps
  user_routes -.->|"uses"| auth_deps
  item_routes -.->|"uses"| auth_deps
  utils_routes -.->|"uses"| auth_deps
  superuser_guard -.->|"extends"| auth_deps

  login_routes --> crud_layer
  login_routes --> security_core
  login_routes --> email_utils
  user_routes --> crud_layer
  user_routes --> security_core
  item_routes --> crud_layer
  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> db_session
  auth_deps --> security_core
  auth_deps --> db_session
  db_session --> db_engine
  db_engine -->|"SQL queries"| pg

  security_core --> config_core
  email_utils --> config_core
  email_utils -->|"SMTP/TLS"| smtp

  alembic_mig -->|"DDL migrations"| pg

  fastapi_api -.->|"Sentry SDK<br/>error reporting"| sentry
```

---

## L3: React SPA

```mermaid
%% L3 – React SPA Components
%% SCOPE: urn:c4:container:spa
graph TB

  fastapi_api["⚙️ FastAPI Backend"]

  subgraph spa["React SPA"]

    subgraph routing_layer["Routing"]
      %% KIND: router
      router["TanStack Router<br/><i>File-based routing with<br/>auth guards (beforeLoad)</i>"]
    end

    subgraph data_layer["Data & Auth Layer"]
      %% KIND: data_access
      query_client["TanStack Query<br/><i>Server state caching,<br/>query invalidation,<br/>optimistic mutations</i>"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>Generated SDK via<br/>@hey-api/openapi-ts<br/>(Axios transport)</i>"]
      %% KIND: service_layer
      auth_hook["Auth Hook · useAuth<br/><i>Login, logout, signup,<br/>currentUser state,<br/>JWT token management</i>"]
    end

    subgraph features["Feature Modules"]
      %% KIND: service_layer
      auth_pages["Auth Pages<br/><i>Login, Signup,<br/>Recover Password,<br/>Reset Password</i>"]
      %% KIND: service_layer
      dashboard_feat["Dashboard<br/><i>Summary view of<br/>pending items & users</i>"]
      %% KIND: service_layer
      items_feat["Items Management<br/><i>DataTable with CRUD:<br/>add, edit, delete items</i>"]
      %% KIND: service_layer
      admin_feat["Admin Panel<br/><i>User management table<br/>(superuser only)</i>"]
      %% KIND: service_layer
      settings_feat["User Settings<br/><i>Profile info, password<br/>change, account deletion</i>"]
    end

    subgraph ui_foundation["UI Foundation"]
      %% KIND: boundary
      ui_lib["shadcn/ui Components<br/><i>Radix UI primitives,<br/>Tailwind CSS v4,<br/>Lucide icons, Sonner toasts</i>"]
      %% KIND: boundary
      theme["Theme Provider<br/><i>Dark / light mode toggle<br/>via next-themes</i>"]
    end

  end

  router --> auth_pages
  router --> dashboard_feat
  router --> items_feat
  router --> admin_feat
  router --> settings_feat

  auth_pages --> auth_hook
  auth_pages --> query_client
  dashboard_feat --> query_client
  items_feat --> query_client
  admin_feat --> query_client
  settings_feat --> query_client

  auth_hook --> query_client
  auth_hook --> api_client
  query_client --> api_client
  api_client -->|"REST / JSON"| fastapi_api

  auth_pages --> ui_lib
  dashboard_feat --> ui_lib
  items_feat --> ui_lib
  admin_feat --> ui_lib
  settings_feat --> ui_lib

  ui_lib --> theme
```

---

## Containment Map

```text
%% ── CONTAINMENT MAP ──────────────────────────────────────────────────
%%
%% Every subgraph from every diagram is listed below.
%% Nesting is expressed by indentation.
%%
%% ── L1 top-level groups ─────────────────────────────────────────────

users              CONTAINS [end_user, admin_user]
platform_boundary  CONTAINS [platform, traefik, spa, fastapi_api, pg]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment ──────────────────────────────────────

fastapi_api CONTAINS [middleware, routes, services, core, utilities]
  middleware  CONTAINS [cors_mw, auth_deps, superuser_guard, db_session]
  routes      CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services    CONTAINS [crud_layer, models_layer]
  core        CONTAINS [security_core, config_core, db_engine]
  utilities   CONTAINS [email_utils, alembic_mig]

spa CONTAINS [routing_layer, data_layer, features, ui_foundation]
  routing_layer  CONTAINS [router]
  data_layer     CONTAINS [query_client, api_client, auth_hook]
  features       CONTAINS [auth_pages, dashboard_feat, items_feat, admin_feat, settings_feat]
  ui_foundation  CONTAINS [ui_lib, theme]
```
