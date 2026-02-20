# C4 Architecture Model — Full Stack FastAPI Template

This document describes the architecture of the Full Stack FastAPI Template
using the [C4 model](https://c4model.com/) expressed as Mermaid diagrams.

> **Stable IDs** are used across all levels so that the same logical element
> keeps the same Mermaid node ID everywhere it appears.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fastapi_fullstack
flowchart TB

  %% ── Actors ──────────────────────────────────────────────────────
  subgraph users ["Users"]
    end_user["End User\n(manages items & profile)"]
    admin_user["Admin User\n(manages all users & items)"]
  end

  %% ── System ──────────────────────────────────────────────────────
  subgraph system_boundary ["Full Stack FastAPI Template"]
    fastapi_fullstack["Full Stack FastAPI Template\n[Software System]\nWeb application for managing\nitems with user authentication"]
  end

  %% ── External Systems ───────────────────────────────────────────
  subgraph external ["External Systems"]
    smtp_server["SMTP Server\n[External System]\nSends transactional emails"]
    sentry["Sentry\n[External System]\nError tracking & monitoring"]
  end

  %% ── Relationships ──────────────────────────────────────────────
  end_user -->|"uses\n[HTTPS]"| fastapi_fullstack
  admin_user -->|"administers\n[HTTPS]"| fastapi_fullstack
  fastapi_fullstack -->|"sends emails via\n[SMTP/TLS]"| smtp_server
  fastapi_fullstack -->|"reports errors to\n[HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fastapi_fullstack
flowchart TB

  %% ── Actors ──────────────────────────────────────────────────────
  subgraph users ["Users"]
    end_user["End User"]
    admin_user["Admin User"]
  end

  %% ── System boundary ────────────────────────────────────────────
  subgraph system_boundary ["Full Stack FastAPI Template"]
    traefik["Traefik\n[Container: Reverse Proxy]\nTLS termination, routing,\nload balancing"]
    react_spa["React SPA\n[Container: TypeScript / React]\nSingle-page app served by Nginx\nTailwind CSS + shadcn/ui"]
    fastapi_backend["FastAPI Backend\n[Container: Python / FastAPI]\nREST API, business logic,\nJWT authentication"]
    pg["PostgreSQL\n[Container: Database]\nStores users, items,\nand application data"]
  end

  %% ── External Systems ───────────────────────────────────────────
  subgraph external ["External Systems"]
    smtp_server["SMTP Server\n[External System]"]
    sentry["Sentry\n[External System]"]
  end

  %% ── Relationships ──────────────────────────────────────────────
  end_user -->|"uses\n[HTTPS]"| traefik
  admin_user -->|"administers\n[HTTPS]"| traefik
  traefik -->|"proxies to\n[HTTP :80]"| react_spa
  traefik -->|"proxies to\n[HTTP :8000]"| fastapi_backend
  react_spa -->|"calls API\n[HTTP/JSON]"| fastapi_backend
  fastapi_backend -->|"reads/writes\n[SQL via psycopg]"| pg
  fastapi_backend -->|"sends emails\n[SMTP/TLS]"| smtp_server
  fastapi_backend -->|"reports errors\n[HTTPS]"| sentry
```

---

## L3: FastAPI Backend — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TB

  %% ── Neighbouring elements (not inside this container) ──────────
  react_spa["React SPA\n[Container]"]
  pg["PostgreSQL\n[Container]"]
  smtp_server["SMTP Server\n[External]"]
  sentry["Sentry\n[External]"]

  %% ── Container boundary ─────────────────────────────────────────
  subgraph fastapi_backend ["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware ["Middleware"]
      %% KIND: router
      cors_mw["CORS Middleware\n[Component]\nAllows cross-origin requests\nfrom the frontend"]
    end

    %% KIND: router
    subgraph routes ["API Routes"]
      %% KIND: router
      api_router["API Router\n[Component: APIRouter]\nMounts all route modules\nunder /api/v1"]
      %% KIND: router
      login_routes["Login Routes\n[Component]\nOAuth2 token login,\npassword recovery & reset"]
      %% KIND: router
      users_routes["Users Routes\n[Component]\nUser CRUD, registration,\nprofile management"]
      %% KIND: router
      items_routes["Items Routes\n[Component]\nItem CRUD with\nownership enforcement"]
      %% KIND: router
      utils_routes["Utils Routes\n[Component]\nHealth check,\ntest email endpoint"]
    end

    %% KIND: service_layer
    subgraph services ["Service Layer"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies\n[Component: deps.py]\nJWT validation, session injection,\ncurrent-user resolution"]
      %% KIND: service_layer
      crud_layer["CRUD Layer\n[Component: crud.py]\nCreate/read/update/delete\noperations for User & Item"]
      %% KIND: service_layer
      email_utils["Email Utilities\n[Component: utils.py]\nRender Jinja2 templates,\nsend transactional emails"]
    end

    %% KIND: data_access
    subgraph data_access ["Data Access"]
      %% KIND: data_access
      models_layer["SQLModel Models\n[Component: models.py]\nUser, Item, Token, and\nPydantic schemas"]
      %% KIND: data_access
      db_engine["DB Engine\n[Component: core/db.py]\nSQLAlchemy engine,\nsession factory, init_db"]
      %% KIND: data_access
      alembic_mig["Alembic Migrations\n[Component]\nSchema versioning &\ndatabase migrations"]
    end

    %% KIND: service_layer
    subgraph core ["Core"]
      %% KIND: service_layer
      security_mod["Security Module\n[Component: core/security.py]\nJWT creation, Argon2/bcrypt\npassword hashing"]
      %% KIND: service_layer
      config_mod["Config Module\n[Component: core/config.py]\nPydantic Settings,\nenvironment variable loading"]
    end

  end

  %% ── Relationships ──────────────────────────────────────────────
  react_spa -->|"HTTP/JSON"| cors_mw
  cors_mw --> api_router
  api_router --> login_routes
  api_router --> users_routes
  api_router --> items_routes
  api_router --> utils_routes

  login_routes --> auth_deps
  login_routes --> crud_layer
  login_routes --> email_utils
  login_routes --> security_mod
  users_routes --> auth_deps
  users_routes --> crud_layer
  users_routes --> email_utils
  items_routes --> auth_deps
  items_routes --> models_layer
  utils_routes --> email_utils

  auth_deps --> security_mod
  auth_deps --> db_engine
  auth_deps --> models_layer
  crud_layer --> models_layer
  crud_layer --> security_mod
  crud_layer --> db_engine
  email_utils --> config_mod
  email_utils --> security_mod

  db_engine --> config_mod
  db_engine -->|"SQL via psycopg"| pg
  alembic_mig -->|"applies migrations"| pg
  email_utils -->|"SMTP/TLS"| smtp_server
  config_mod -.->|"DSN config"| sentry
```

---

## L3: React SPA — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:react_spa
flowchart TB

  %% ── Neighbouring elements ──────────────────────────────────────
  fastapi_backend["FastAPI Backend\n[Container]"]
  end_user["End User"]
  admin_user["Admin User"]

  %% ── Container boundary ─────────────────────────────────────────
  subgraph react_spa ["React SPA"]

    %% KIND: router
    subgraph routing ["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router\n[Component]\nFile-based routing,\nroute guards, code splitting"]
    end

    %% KIND: service_layer
    subgraph state_mgmt ["State & Data"]
      %% KIND: service_layer
      query_client["React Query Client\n[Component]\nServer state management,\ncaching, error handling"]
      %% KIND: integration
      api_client["OpenAPI Client\n[Component: sdk.gen.ts]\nAuto-generated typed HTTP\nclient for backend API"]
      %% KIND: service_layer
      auth_hook["useAuth Hook\n[Component: useAuth.ts]\nLogin, logout, signup,\ncurrent user state"]
    end

    %% KIND: boundary
    subgraph pages ["Pages"]
      %% KIND: boundary
      auth_pages["Auth Pages\n[Component]\nLogin, Signup,\nRecover & Reset Password"]
      %% KIND: boundary
      dashboard_page["Dashboard Page\n[Component]\nWelcome screen for\nauthenticated users"]
      %% KIND: boundary
      items_page["Items Page\n[Component]\nItem list, create,\nedit, delete"]
      %% KIND: boundary
      admin_page["Admin Page\n[Component]\nUser management\n(superuser only)"]
      %% KIND: boundary
      settings_page["Settings Page\n[Component]\nProfile editing,\npassword change, account deletion"]
    end

    %% KIND: boundary
    subgraph ui_layer ["UI Library"]
      %% KIND: boundary
      ui_lib["shadcn/ui Components\n[Component]\nTailwind CSS + Radix UI\nprimitives"]
      %% KIND: service_layer
      theme_prov["Theme Provider\n[Component]\nDark/light mode\nvia next-themes"]
    end

  end

  %% ── Relationships ──────────────────────────────────────────────
  end_user -->|"interacts"| tanstack_router
  admin_user -->|"interacts"| tanstack_router

  tanstack_router --> auth_pages
  tanstack_router --> dashboard_page
  tanstack_router --> items_page
  tanstack_router --> admin_page
  tanstack_router --> settings_page

  auth_pages --> auth_hook
  dashboard_page --> auth_hook
  items_page --> query_client
  admin_page --> query_client
  settings_page --> query_client
  settings_page --> auth_hook

  auth_hook --> api_client
  query_client --> api_client
  api_client -->|"HTTP/JSON\n/api/v1/*"| fastapi_backend

  auth_pages --> ui_lib
  dashboard_page --> ui_lib
  items_page --> ui_lib
  admin_page --> ui_lib
  settings_page --> ui_lib
  ui_lib --> theme_prov
```

---

## L3: Traefik — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:traefik
flowchart TB

  %% ── Neighbouring elements ──────────────────────────────────────
  end_user["End User"]
  admin_user["Admin User"]
  react_spa["React SPA\n[Container]"]
  fastapi_backend["FastAPI Backend\n[Container]"]

  %% ── Container boundary ─────────────────────────────────────────
  subgraph traefik ["Traefik Reverse Proxy"]

    %% KIND: router
    http_entrypoint["HTTP Entrypoint\n[Component: :80]\nRedirects to HTTPS"]
    %% KIND: router
    https_entrypoint["HTTPS Entrypoint\n[Component: :443]\nTLS termination via\nLet's Encrypt"]
    %% KIND: router
    https_redirect["HTTPS Redirect Middleware\n[Component]\nForces HTTP → HTTPS"]
    %% KIND: router
    frontend_router["Frontend Router\n[Component]\nHost: dashboard.DOMAIN"]
    %% KIND: router
    backend_router["Backend Router\n[Component]\nHost: api.DOMAIN"]
    %% KIND: integration
    le_resolver["Let's Encrypt Resolver\n[Component]\nACME TLS challenge\ncertificate management"]
  end

  %% ── Relationships ──────────────────────────────────────────────
  end_user -->|"HTTPS"| https_entrypoint
  admin_user -->|"HTTPS"| https_entrypoint
  end_user -->|"HTTP"| http_entrypoint
  admin_user -->|"HTTP"| http_entrypoint
  http_entrypoint --> https_redirect
  https_redirect --> https_entrypoint
  https_entrypoint --> frontend_router
  https_entrypoint --> backend_router
  frontend_router -->|"proxy :80"| react_spa
  backend_router -->|"proxy :8000"| fastapi_backend
  https_entrypoint --> le_resolver
```

---

## L3: PostgreSQL — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:pg
flowchart TB

  %% ── Neighbouring elements ──────────────────────────────────────
  fastapi_backend["FastAPI Backend\n[Container]"]

  %% ── Container boundary ─────────────────────────────────────────
  subgraph pg ["PostgreSQL"]

    %% KIND: storage
    user_table["user Table\n[Component: Table]\nid, email, full_name,\nhashed_password, is_active,\nis_superuser, created_at"]
    %% KIND: storage
    item_table["item Table\n[Component: Table]\nid, title, description,\nowner_id (FK → user),\ncreated_at"]
    %% KIND: storage
    alembic_version["alembic_version Table\n[Component: Table]\nSchema migration\nversion tracking"]
  end

  %% ── Relationships ──────────────────────────────────────────────
  fastapi_backend -->|"reads/writes\n[SQL]"| user_table
  fastapi_backend -->|"reads/writes\n[SQL]"| item_table
  fastapi_backend -->|"checks version\n[SQL]"| alembic_version
  item_table -.->|"FK owner_id\nCASCADE DELETE"| user_table
```

---

## Containment Map

```text
%% ════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Template
%% Every subgraph and node from every diagram is listed here.
%% Indentation expresses nesting: child is indented under parent.
%% ════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [end_user, admin_user]
system_boundary  CONTAINS [fastapi_fullstack, traefik, react_spa, fastapi_backend, pg]
external         CONTAINS [smtp_server, sentry]

%% ── L2 system_boundary internals ────────────────────────────────
system_boundary  CONTAINS [traefik, react_spa, fastapi_backend, pg]

%% ── L3: fastapi_backend ────────────────────────────────────────
fastapi_backend  CONTAINS [middleware, routes, services, data_access, core]
  middleware     CONTAINS [cors_mw]
  routes         CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes]
  services       CONTAINS [auth_deps, crud_layer, email_utils]
  data_access    CONTAINS [models_layer, db_engine, alembic_mig]
  core           CONTAINS [security_mod, config_mod]

%% ── L3: react_spa ──────────────────────────────────────────────
react_spa        CONTAINS [routing, state_mgmt, pages, ui_layer]
  routing        CONTAINS [tanstack_router]
  state_mgmt     CONTAINS [query_client, api_client, auth_hook]
  pages          CONTAINS [auth_pages, dashboard_page, items_page, admin_page, settings_page]
  ui_layer       CONTAINS [ui_lib, theme_prov]

%% ── L3: traefik ────────────────────────────────────────────────
traefik          CONTAINS [http_entrypoint, https_entrypoint, https_redirect, frontend_router, backend_router, le_resolver]

%% ── L3: pg ─────────────────────────────────────────────────────
pg               CONTAINS [user_table, item_table, alembic_version]
```
