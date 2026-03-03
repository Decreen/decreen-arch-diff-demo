# C4 Architecture Model — FastAPI Full-Stack Platform

> Auto-generated C4 model for the
> [full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template)
> codebase. Covers L1 (System Context), L2 (Container), and L3 (Component)
> diagrams with a complete containment map.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system-context:platform_boundary
graph TB

  subgraph users["Users"]
    user["User<br/><i>End user who manages<br/>items and profile</i>"]
    admin_user["Administrator<br/><i>Superuser who manages<br/>all users and content</i>"]
  end

  subgraph platform_boundary["FastAPI Full-Stack Platform"]
    platform_inner["Full-Stack Web Application<br/><i>Item management with user auth,<br/>CRUD operations, and admin panel</i><br/><i>[FastAPI + React + PostgreSQL]</i>"]
  end

  subgraph external["External Services"]
    smtp["SMTP Email Service<br/><i>Delivers transactional emails<br/>(password recovery, welcome)</i>"]
    sentry["Sentry<br/><i>Error monitoring and<br/>performance tracking</i>"]
  end

  user -->|"Uses via browser"| platform_inner
  admin_user -->|"Administers via browser"| platform_inner
  platform_inner -->|"Sends emails via SMTP"| smtp
  platform_inner -->|"Reports errors"| sentry
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:platform_boundary
graph TB

  subgraph users["Users"]
    user["User"]
    admin_user["Administrator"]
  end

  subgraph platform_boundary["FastAPI Full-Stack Platform"]
    traefik["Traefik<br/><i>Reverse proxy and TLS<br/>termination, routes traffic<br/>to frontend and backend</i><br/><i>[Go]</i>"]
    spa["React SPA<br/><i>Single-page application<br/>serving dashboard, item<br/>management, and admin UI</i><br/><i>[React 19, Vite, TypeScript]</i>"]
    fastapi["FastAPI Backend<br/><i>REST API providing auth,<br/>user and item CRUD,<br/>and email dispatch</i><br/><i>[Python, FastAPI, SQLModel]</i>"]
    pg[("PostgreSQL<br/><i>Primary relational store<br/>for users and items</i><br/><i>[PostgreSQL 18]</i>")]
  end

  subgraph external["External Services"]
    smtp["SMTP Email Service<br/><i>Transactional email delivery</i>"]
    sentry["Sentry<br/><i>Error monitoring</i>"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"Routes dashboard.*"| spa
  traefik -->|"Routes api.*"| fastapi
  spa -->|"REST /api/v1/*<br/>[JSON over HTTPS]"| fastapi
  fastapi -->|"SQL queries<br/>[psycopg3]"| pg
  fastapi -->|"Sends emails<br/>[SMTP/TLS]"| smtp
  fastapi -->|"Reports errors<br/>[HTTPS]"| sentry
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi
graph TB

  spa["React SPA"]
  pg[("PostgreSQL")]
  smtp["SMTP Email Service"]

  subgraph fastapi["FastAPI Backend"]

    %% KIND: router
    api_router["API Router<br/><i>Main FastAPI router<br/>mounts all /api/v1/* routes</i>"]

    subgraph routes["Route Handlers"]
      %% KIND: router
      login_routes["Login Routes<br/><i>OAuth2 token issuance,<br/>password recovery and reset</i>"]
      %% KIND: router
      user_routes["User Routes<br/><i>User CRUD, signup,<br/>profile management</i>"]
      %% KIND: router
      item_routes["Item Routes<br/><i>Item CRUD with<br/>ownership enforcement</i>"]
      %% KIND: router
      util_routes["Utils Routes<br/><i>Health check and<br/>test email endpoint</i>"]
    end

    subgraph middleware["Dependencies / Middleware"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies<br/><i>OAuth2 bearer extraction,<br/>JWT validation, current-user<br/>and superuser guards</i>"]
      %% KIND: data_access
      db_session["DB Session Provider<br/><i>Request-scoped SQLModel<br/>session via FastAPI Depends</i>"]
    end

    subgraph services["Service Layer"]
      %% KIND: data_access
      crud_layer["CRUD Operations<br/><i>create_user, update_user,<br/>authenticate, create_item</i>"]
      %% KIND: integration
      email_utils["Email Utilities<br/><i>Jinja2 template rendering<br/>and SMTP dispatch</i>"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      core_config["Configuration<br/><i>Pydantic BaseSettings<br/>loaded from .env</i>"]
      %% KIND: service_layer
      core_security["Security<br/><i>JWT encode/decode (PyJWT),<br/>Argon2 + Bcrypt hashing</i>"]
      %% KIND: storage
      core_db["Database Engine<br/><i>SQLAlchemy create_engine,<br/>session factory</i>"]
    end

    %% KIND: data_access
    models_layer["SQLModel Models<br/><i>User, Item ORM tables<br/>and Pydantic schemas</i>"]

  end

  spa -->|"REST API calls"| api_router
  api_router --> login_routes
  api_router --> user_routes
  api_router --> item_routes
  api_router --> util_routes

  login_routes --> auth_deps
  login_routes --> crud_layer
  user_routes --> auth_deps
  user_routes --> crud_layer
  item_routes --> auth_deps
  item_routes --> crud_layer
  util_routes --> email_utils

  auth_deps --> core_security
  auth_deps --> db_session
  crud_layer --> models_layer
  crud_layer --> core_security
  crud_layer --> db_session
  email_utils --> core_config

  db_session --> core_db
  core_db -->|"SQL"| pg
  email_utils -->|"SMTP"| smtp
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

  fastapi["FastAPI Backend"]

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router<br/><i>File-based routing with<br/>auth guards in beforeLoad,<br/>auto code-splitting</i>"]
      %% KIND: boundary
      layout["Layout Shell<br/><i>Sidebar layout wrapper,<br/>auth redirect, Outlet</i>"]
    end

    subgraph features["Feature Pages"]
      %% KIND: boundary
      auth_pages["Auth Pages<br/><i>Login, Signup,<br/>Recover / Reset Password</i>"]
      %% KIND: boundary
      dashboard_page["Dashboard<br/><i>Welcome page with<br/>user greeting</i>"]
      %% KIND: boundary
      items_feature["Items Management<br/><i>CRUD data table with<br/>add / edit / delete dialogs</i>"]
      %% KIND: boundary
      admin_feature["Admin Panel<br/><i>User management table<br/>(superuser only)</i>"]
      %% KIND: boundary
      settings_feature["User Settings<br/><i>Profile edit, password<br/>change, account deletion</i>"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>Auto-generated Axios SDK<br/>(@hey-api/openapi-ts)</i>"]
      %% KIND: service_layer
      auth_hook["useAuth Hook<br/><i>TanStack Query for current<br/>user, login/logout mutations</i>"]
    end

    subgraph ui_layer["UI Foundation"]
      %% KIND: boundary
      ui_components["UI Component Library<br/><i>shadcn/ui + Radix primitives,<br/>Tailwind CSS v4</i>"]
      %% KIND: service_layer
      theme_provider["Theme Provider<br/><i>Dark / light / system<br/>theme via React Context</i>"]
    end

  end

  tanstack_router --> layout
  layout --> dashboard_page
  layout --> items_feature
  layout --> admin_feature
  layout --> settings_feature
  tanstack_router --> auth_pages

  auth_pages --> auth_hook
  auth_pages --> api_client
  items_feature --> api_client
  admin_feature --> api_client
  settings_feature --> api_client
  dashboard_page --> auth_hook
  auth_hook --> api_client

  auth_pages --> ui_components
  items_feature --> ui_components
  admin_feature --> ui_components
  settings_feature --> ui_components
  dashboard_page --> ui_components

  api_client -->|"REST /api/v1/*<br/>[JSON over HTTPS]"| fastapi
```

---

## Containment Map

```text
%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi, pg]
external           CONTAINS [smtp, sentry]

%% ── L2 → L3: FastAPI Backend ───────────────────────────────────────
fastapi    CONTAINS [api_router, routes, middleware, services, core, models_layer]
  routes     CONTAINS [login_routes, user_routes, item_routes, util_routes]
  middleware CONTAINS [auth_deps, db_session]
  services   CONTAINS [crud_layer, email_utils]
  core       CONTAINS [core_config, core_security, core_db]

%% ── L2 → L3: React SPA ────────────────────────────────────────────
spa            CONTAINS [routing, features, services_layer, ui_layer]
  routing        CONTAINS [tanstack_router, layout]
  features       CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
  services_layer CONTAINS [api_client, auth_hook]
  ui_layer       CONTAINS [ui_components, theme_provider]
```
