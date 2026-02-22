# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
flowchart TD
%% SCOPE: urn:c4:system:fullstack_fastapi_platform
%% LEVEL: 1 — System Context

  subgraph users["Users"]
    user["Regular User\n<i>Person</i>\nBrowses dashboard, manages own items"]
    admin_user["Admin User\n<i>Person / Superuser</i>\nManages all users and items"]
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    system_box["Full Stack FastAPI Platform\n<i>Software System</i>\nWeb application for user and item management\nwith JWT auth, email notifications, and admin dashboard"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n<i>External System</i>\nEmail delivery for password resets\nand account notifications"]
    sentry["Sentry\n<i>External System</i>\nError monitoring and\nperformance tracking"]
  end

  user -->|"Uses web dashboard\n[HTTPS]"| system_box
  admin_user -->|"Manages users & system\n[HTTPS]"| system_box
  system_box -->|"Sends transactional emails\n[SMTP]"| smtp
  system_box -->|"Reports errors & traces\n[HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
flowchart TD
%% SCOPE: urn:c4:system:fullstack_fastapi_platform
%% LEVEL: 2 — Container

  user["Regular User\n<i>Person</i>"]
  admin_user["Admin User\n<i>Person / Superuser</i>"]

  subgraph system_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik\n<i>Reverse Proxy / Load Balancer</i>\nRoutes HTTPS traffic to frontend\nand backend by hostname"]
    spa["React SPA\n<i>Browser Application</i>\nReact 19, TypeScript, TanStack Router,\nTanStack Query, shadcn/ui, Tailwind CSS"]
    fastapi_api["FastAPI Backend\n<i>Python API Server</i>\nFastAPI, SQLModel, Pydantic,\nUvicorn, Alembic migrations"]
    pg["PostgreSQL 18\n<i>Relational Database</i>\nStores users, items,\nand application data"]
  end

  smtp["SMTP Server\n<i>External System</i>"]
  sentry["Sentry\n<i>External System</i>"]

  user -->|"HTTPS requests"| traefik
  admin_user -->|"HTTPS requests"| traefik
  traefik -->|"Serves SPA static files\n[dashboard.* → Nginx]"| spa
  traefik -->|"Proxies API calls\n[api.* → :8000]"| fastapi_api
  spa -.->|"REST API calls\n[JSON/HTTPS via Traefik]"| fastapi_api
  fastapi_api -->|"Reads/writes data\n[SQL via psycopg]"| pg
  fastapi_api -->|"Sends emails\n[SMTP/TLS]"| smtp
  fastapi_api -->|"Reports errors\n[Sentry SDK / HTTPS]"| sentry
```

---

## L3: FastAPI Backend — Component Diagram

```mermaid
flowchart TD
%% SCOPE: urn:c4:container:fastapi_api
%% LEVEL: 3 — Component

  spa["React SPA"]

  subgraph fastapi_api["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n<i>Starlette Middleware</i>\nEnforces allowed origins,\nmethods, and headers"]
    end

    subgraph deps["Dependencies"]
      %% KIND: boundary
      auth_deps["Auth Dependencies\n<i>FastAPI Depends</i>\nOAuth2 bearer token extraction,\nJWT validation, current user injection"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes\n<i>APIRouter /login/*</i>\nAccess token, test token,\npassword recovery & reset"]
      %% KIND: router
      user_routes["User Routes\n<i>APIRouter /users/*</i>\nUser CRUD, signup, profile,\npassword update"]
      %% KIND: router
      item_routes["Item Routes\n<i>APIRouter /items/*</i>\nItem CRUD with\nowner-based access control"]
      %% KIND: router
      util_routes["Utility Routes\n<i>APIRouter /utils/*</i>\nHealth check,\ntest email"]
    end

    subgraph services["Service Layer"]
      %% KIND: data_access
      crud_layer["CRUD Operations\n<i>Python Module</i>\nUser & Item create/read/update/delete,\nauthentication logic"]
      %% KIND: integration
      email_utils["Email Utilities\n<i>Python Module</i>\nJinja2 template rendering,\nSMTP sending, token generation"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      security_mod["Security Module\n<i>Python Module</i>\nJWT creation, Argon2/Bcrypt\npassword hashing & verification"]
      %% KIND: storage
      db_mod["Database Module\n<i>Python Module</i>\nSQLAlchemy engine creation,\ninit_db with first superuser"]
      %% KIND: service_layer
      config_mod["Configuration\n<i>Pydantic Settings</i>\nEnvironment-based config:\nDB, SMTP, JWT, CORS, Sentry"]
      %% KIND: data_access
      models_layer["Models & Schemas\n<i>SQLModel Module</i>\nUser, Item ORM models;\nrequest/response Pydantic schemas"]
    end

  end

  pg["PostgreSQL 18"]
  smtp["SMTP Server"]
  sentry["Sentry"]

  spa -->|"HTTP requests"| cors_mw
  cors_mw --> auth_deps
  auth_deps --> login_routes
  auth_deps --> user_routes
  auth_deps --> item_routes
  auth_deps --> util_routes
  login_routes --> crud_layer
  login_routes --> email_utils
  login_routes --> security_mod
  user_routes --> crud_layer
  user_routes --> email_utils
  user_routes --> security_mod
  item_routes --> crud_layer
  util_routes --> email_utils
  crud_layer --> models_layer
  crud_layer --> db_mod
  crud_layer --> security_mod
  email_utils --> security_mod
  email_utils -->|"SMTP"| smtp
  db_mod --> config_mod
  db_mod -->|"SQL via psycopg"| pg
  security_mod --> config_mod
  models_layer --> db_mod
  fastapi_api -->|"Sentry SDK"| sentry
```

---

## L3: React SPA — Component Diagram

```mermaid
flowchart TD
%% SCOPE: urn:c4:container:spa
%% LEVEL: 3 — Component

  user["Regular User"]
  admin_user["Admin User"]

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router\n<i>File-based Router</i>\nType-safe routing with\nauth guards and lazy loading"]
    end

    subgraph data_layer["Data Layer"]
      %% KIND: integration
      api_client["API Client\n<i>Auto-generated SDK</i>\nOpenAPI-generated service classes:\nItems, Login, Users, Utils"]
      %% KIND: service_layer
      query_client["React Query\n<i>TanStack Query</i>\nServer state management,\ncaching, and error handling"]
      %% KIND: service_layer
      auth_hook["Auth Hook\n<i>Custom Hook</i>\nLogin/logout mutations,\ntoken storage, current user query"]
    end

    subgraph features["Features"]
      %% KIND: boundary
      admin_feature["Admin Feature\n<i>React Components</i>\nUser management: add, edit,\ndelete users (superuser only)"]
      %% KIND: boundary
      items_feature["Items Feature\n<i>React Components</i>\nItem management: add, edit,\ndelete items with data table"]
      %% KIND: boundary
      settings_feature["Settings Feature\n<i>React Components</i>\nUser profile, change password,\ndelete account"]
    end

    subgraph shared["Shared"]
      %% KIND: boundary
      common_layout["Layout Components\n<i>React Components</i>\nSidebar, Footer, AuthLayout,\nErrorBoundary, NotFound"]
      %% KIND: boundary
      ui_lib["shadcn/ui Library\n<i>UI Primitives</i>\nRadix + Tailwind: buttons, dialogs,\nforms, tables, dropdowns"]
      %% KIND: service_layer
      theme_provider["Theme Provider\n<i>React Context</i>\nDark/light mode toggle\nvia next-themes"]
    end

  end

  fastapi_api["FastAPI Backend"]

  user -->|"Browses dashboard"| tanstack_router
  admin_user -->|"Manages users"| tanstack_router
  tanstack_router --> auth_hook
  tanstack_router --> admin_feature
  tanstack_router --> items_feature
  tanstack_router --> settings_feature
  auth_hook --> api_client
  auth_hook --> query_client
  admin_feature --> query_client
  admin_feature --> api_client
  admin_feature --> ui_lib
  items_feature --> query_client
  items_feature --> api_client
  items_feature --> ui_lib
  settings_feature --> query_client
  settings_feature --> api_client
  settings_feature --> ui_lib
  admin_feature --> common_layout
  items_feature --> common_layout
  settings_feature --> common_layout
  common_layout --> ui_lib
  common_layout --> theme_provider
  api_client -->|"REST API calls\n[JSON/HTTPS]"| fastapi_api
  query_client --> api_client
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [user, admin_user]
system_boundary  CONTAINS [traefik, spa, fastapi_api, pg]
external         CONTAINS [smtp, sentry]

%% ── L2 → L3 internal containment ────────────────────────────────
fastapi_api  CONTAINS [middleware, deps, routes, services, core]
  middleware   CONTAINS [cors_mw]
  deps         CONTAINS [auth_deps]
  routes       CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services     CONTAINS [crud_layer, email_utils]
  core         CONTAINS [security_mod, db_mod, config_mod, models_layer]

spa          CONTAINS [routing, data_layer, features, shared]
  routing      CONTAINS [tanstack_router]
  data_layer   CONTAINS [api_client, query_client, auth_hook]
  features     CONTAINS [admin_feature, items_feature, settings_feature]
  shared       CONTAINS [common_layout, ui_lib, theme_provider]
```
