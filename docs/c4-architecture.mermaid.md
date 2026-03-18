# C4 Architecture Model – Full Stack FastAPI Project

> Auto-generated from source code analysis. Evidence drawn exclusively from
> source files, Docker/Compose configs, CI definitions, and runtime configuration.

---

## L1: System Context

```mermaid
graph TB
  %% L1: System Context – Full Stack FastAPI Project

  subgraph users["Users"]
    user["Regular User<br>[Person]<br>Manages personal items<br>via web dashboard"]
    admin_user["Admin / Superuser<br>[Person]<br>Manages users, items,<br>and system settings"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    the_system["Full Stack FastAPI Project<br>[Software System]<br>Full-stack web application for<br>user and item management"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server<br>[External System]<br>Delivers transactional emails<br>(password reset, new account)"]
    sentry["Sentry<br>[External System]<br>Error tracking and<br>performance monitoring"]
  end

  user -->|"Browses dashboard,<br>manages items"| the_system
  admin_user -->|"Administers users,<br>manages system"| the_system
  the_system -->|"Sends transactional<br>emails via SMTP"| smtp
  the_system -->|"Reports errors and<br>traces via HTTPS"| sentry
```

---

## L2: Container

```mermaid
graph TB
  %% L2: Container – Full Stack FastAPI Project

  subgraph users["Users"]
    user["Regular User<br>[Person]"]
    admin_user["Admin / Superuser<br>[Person]"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    traefik["Traefik<br>[Container: Traefik 3.6]<br>Reverse proxy, TLS termination,<br>domain-based routing"]
    spa["React SPA<br>[Container: React 19 / Vite / Nginx]<br>Single-page application served<br>as static files by Nginx"]
    fastapi_backend["FastAPI Backend<br>[Container: Python 3.10 / FastAPI / Uvicorn]<br>REST API with JWT auth,<br>CRUD operations, email sending"]
    pg[("PostgreSQL<br>[Container: PostgreSQL 18]<br>Stores users, items,<br>and application data")]
    adminer["Adminer<br>[Container: Adminer]<br>Database administration UI"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server<br>[External System]<br>Transactional email delivery"]
    sentry["Sentry<br>[External System]<br>Error tracking"]
  end

  user -->|"Browses via HTTPS"| traefik
  admin_user -->|"Browses via HTTPS"| traefik
  admin_user -->|"Manages DB via<br>adminer.DOMAIN"| adminer

  traefik -->|"Routes dashboard.DOMAIN"| spa
  traefik -->|"Routes api.DOMAIN"| fastapi_backend

  spa -->|"API calls<br>[HTTPS / JSON]"| fastapi_backend

  fastapi_backend -->|"Reads / writes<br>[SQL via psycopg]"| pg
  fastapi_backend -->|"Sends emails<br>[SMTP / TLS]"| smtp
  fastapi_backend -->|"Reports errors<br>[HTTPS]"| sentry

  adminer -->|"Queries<br>[SQL]"| pg
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% L3: FastAPI Backend
  %% SCOPE: urn:c4:container:fastapi_backend

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    cors_mw["CORS Middleware<br>[Component: Starlette CORSMiddleware]<br>Enforces allowed origins from<br>BACKEND_CORS_ORIGINS config"]

    %% KIND: boundary
    auth_deps["Auth Dependencies<br>[Component: FastAPI Depends]<br>OAuth2 bearer token extraction,<br>JWT validation, user lookup,<br>superuser gate"]

    %% KIND: service_layer
    config_mod["Settings<br>[Component: Pydantic BaseSettings]<br>Loads env vars for DB, SMTP,<br>secrets, Sentry, CORS origins"]

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes<br>[Component: APIRouter /login]<br>OAuth2 token login,<br>password recovery and reset"]

      %% KIND: router
      user_routes["User Routes<br>[Component: APIRouter /users]<br>User CRUD, registration,<br>profile and password management"]

      %% KIND: router
      item_routes["Item Routes<br>[Component: APIRouter /items]<br>Item CRUD with ownership<br>enforcement"]

      %% KIND: router
      util_routes["Utility Routes<br>[Component: APIRouter /utils]<br>Health check endpoint,<br>test email trigger"]
    end

    %% KIND: service_layer
    crud_layer["CRUD Layer<br>[Component: Python module crud.py]<br>User/item create, read, update,<br>authenticate with timing-attack prevention"]

    %% KIND: service_layer
    security_mod["Security Module<br>[Component: Python module security.py]<br>JWT creation (HS256), password<br>hashing (Argon2 / Bcrypt via pwdlib)"]

    %% KIND: integration
    email_svc["Email Service<br>[Component: Python module utils.py]<br>Jinja2 template rendering,<br>SMTP sending via emails lib"]

    %% KIND: data_access
    db_engine["Database Engine<br>[Component: SQLAlchemy create_engine]<br>Connection pool, session factory,<br>psycopg driver"]

    %% KIND: data_access
    models_layer["SQLModel Models<br>[Component: SQLModel definitions]<br>User and Item table models,<br>Pydantic request/response schemas"]

  end

  %% Neighboring containers and external systems
  spa["React SPA<br>[Container]"]
  pg[("PostgreSQL<br>[Container]")]
  smtp["SMTP Server<br>[External System]"]
  sentry["Sentry<br>[External System]"]

  %% Request flow
  spa -->|"API requests<br>[HTTPS / JSON]"| cors_mw
  cors_mw --> routes

  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  util_routes --> auth_deps

  auth_deps --> security_mod
  auth_deps --> db_engine

  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_svc

  user_routes --> crud_layer
  user_routes --> email_svc

  item_routes --> crud_layer
  item_routes --> models_layer

  util_routes --> email_svc

  crud_layer --> security_mod
  crud_layer --> models_layer
  crud_layer --> db_engine

  db_engine -->|"SQL via psycopg"| pg
  email_svc -->|"SMTP"| smtp

  fastapi_backend -.->|"sentry_sdk.init<br>when SENTRY_DSN set"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
  %% L3: React SPA
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    %% KIND: router
    router["TanStack Router<br>[Component: @tanstack/router v1]<br>File-based routing with<br>code splitting and auth guards"]

    %% KIND: service_layer
    query_client["Query Client<br>[Component: @tanstack/react-query v5]<br>Server state caching, background<br>refetch, 401/403 error handling"]

    %% KIND: service_layer
    auth_hook["Auth Hook<br>[Component: React hook useAuth]<br>Login, signup, logout mutations,<br>current user query, token in localStorage"]

    %% KIND: integration
    api_client["API Client<br>[Component: @hey-api/openapi-ts]<br>Generated TypeScript SDK<br>(LoginService, UsersService,<br>ItemsService, UtilsService)"]

    %% KIND: service_layer
    theme_provider["Theme Provider<br>[Component: React Context]<br>Light / dark / system mode,<br>persisted in localStorage"]

    subgraph pages["Pages & Features"]
      %% KIND: boundary
      auth_pages["Auth Pages<br>[Component: React routes]<br>Login, Signup, Password<br>Recovery, Reset Password"]

      %% KIND: boundary
      dashboard_page["Dashboard<br>[Component: React route /]<br>Welcome greeting, user info"]

      %% KIND: boundary
      items_feature["Items Management<br>[Component: React route /items]<br>DataTable with CRUD dialogs<br>(Add, Edit, Delete items)"]

      %% KIND: boundary
      admin_feature["Admin Panel<br>[Component: React route /admin]<br>User management DataTable<br>(superuser only)"]

      %% KIND: boundary
      settings_feature["User Settings<br>[Component: React route /settings]<br>Profile info, change password,<br>delete account"]
    end

    %% KIND: boundary
    ui_components["UI Component Library<br>[Component: shadcn/ui + Radix UI]<br>Buttons, dialogs, forms, tables,<br>sidebar, toasts (Sonner)"]

  end

  %% Neighboring container
  fastapi_backend["FastAPI Backend<br>[Container]"]

  %% Routing
  router --> auth_pages
  router --> dashboard_page
  router --> items_feature
  router --> admin_feature
  router --> settings_feature

  %% Data fetching
  auth_pages --> auth_hook
  dashboard_page --> query_client
  items_feature --> query_client
  admin_feature --> query_client
  settings_feature --> query_client

  auth_hook --> api_client
  query_client --> api_client

  %% UI usage
  auth_pages --> ui_components
  dashboard_page --> ui_components
  items_feature --> ui_components
  admin_feature --> ui_components
  settings_feature --> ui_components

  %% External communication
  api_client -->|"HTTPS / JSON<br>Bearer JWT"| fastapi_backend
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════
%% CONTAINMENT MAP – Full Stack FastAPI Project
%% Covers L1, L2, and all L3 diagrams.
%% Format: <subgraph_id> CONTAINS [<child_id>, ...]
%% Indentation shows nesting depth.
%% ═══════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
system_boundary    CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment: FastAPI Backend ─────────────────
fastapi_backend    CONTAINS [cors_mw, auth_deps, config_mod, routes, crud_layer, security_mod, email_svc, db_engine, models_layer]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes]

%% ── L2→L3 internal containment: React SPA ──────────────────────
spa                CONTAINS [router, query_client, auth_hook, api_client, theme_provider, pages, ui_components]
  pages            CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
```
