# C4 Architecture Model — Full Stack FastAPI Project

> Auto-generated C4 architecture expressed as Mermaid diagrams.
> Covers L1 (System Context), L2 (Container), and L3 (Component) views.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fastapi_project
flowchart TB

    subgraph users ["Users"]
        user["👤 User\n[Person]\nRegistered application user"]
        admin_user["👤 Admin\n[Person]\nSuperuser with elevated privileges"]
    end

    subgraph system_boundary ["FastAPI Project"]
        fastapi_project["📦 FastAPI Project\n[Software System]\nFull-stack web application\nwith REST API and SPA frontend"]
    end

    subgraph external ["External Systems"]
        smtp["✉️ SMTP Server\n[External System]\nSends transactional emails"]
        sentry["📊 Sentry\n[External System]\nError monitoring and tracing"]
    end

    user -- "Browses dashboard,\nmanages items" --> fastapi_project
    admin_user -- "Manages users,\nadministers platform" --> fastapi_project
    fastapi_project -- "Sends password-reset\nand account emails" --> smtp
    fastapi_project -- "Reports errors\nand traces" --> sentry
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fastapi_project
flowchart TB

    subgraph users ["Users"]
        user["👤 User\n[Person]"]
        admin_user["👤 Admin\n[Person]"]
    end

    subgraph system_boundary ["FastAPI Project"]
        traefik["🔀 Traefik\n[Container: Traefik v3]\nReverse proxy, TLS termination,\nroute-based load balancing"]

        spa["🌐 React SPA\n[Container: React 19 / Vite / Nginx]\nSingle-page application\nserving the user interface"]

        fastapi_api["⚙️ FastAPI Backend\n[Container: Python / FastAPI]\nREST API serving /api/v1\nwith JWT authentication"]

        pg["🗄️ PostgreSQL\n[Container: PostgreSQL 18]\nRelational database storing\nusers, items, and credentials"]
    end

    subgraph external ["External Systems"]
        smtp["✉️ SMTP Server\n[External System]"]
        sentry["📊 Sentry\n[External System]"]
    end

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik

    traefik -- "dashboard.*\nHTTP/80" --> spa
    traefik -- "api.*\nHTTP/8000" --> fastapi_api

    spa -- "REST /api/v1\nJSON over HTTPS" --> fastapi_api

    fastapi_api -- "SQL\npsycopg" --> pg
    fastapi_api -- "SMTP\nTLS" --> smtp
    fastapi_api -- "HTTPS\nDSN" --> sentry
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_api
flowchart TB

    spa["🌐 React SPA\n[Container]"]
    pg["🗄️ PostgreSQL\n[Container]"]
    smtp["✉️ SMTP Server\n[External System]"]
    sentry["📊 Sentry\n[External System]"]

    subgraph fastapi_api ["FastAPI Backend"]

        subgraph middleware ["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware\n[Component: Starlette]\nEnforces allowed origins,\nmethods, and headers"]
            %% KIND: integration
            sentry_mw["Sentry Integration\n[Component: sentry-sdk]\nCaptures exceptions\nand performance traces"]
        end

        subgraph routes ["API Routes"]
            %% KIND: router
            login_routes["Login Routes\n[Component: FastAPI Router]\n/login/* — OAuth2 token,\npassword recovery & reset"]
            %% KIND: router
            users_routes["Users Routes\n[Component: FastAPI Router]\n/users/* — CRUD, signup,\nprofile, password change"]
            %% KIND: router
            items_routes["Items Routes\n[Component: FastAPI Router]\n/items/* — CRUD operations\non user-owned items"]
            %% KIND: router
            utils_routes["Utils Routes\n[Component: FastAPI Router]\n/utils/* — health check,\ntest email"]
        end

        subgraph auth ["Auth & Security"]
            %% KIND: boundary
            auth_deps["Auth Dependencies\n[Component: FastAPI Depends]\nOAuth2 bearer extraction,\nJWT validation, user lookup"]
            %% KIND: service_layer
            security_mod["Security Module\n[Component: PyJWT / pwdlib]\nJWT creation, password\nhashing (Argon2/Bcrypt)"]
        end

        subgraph services ["Business Logic"]
            %% KIND: service_layer
            crud_layer["CRUD Layer\n[Component: SQLModel]\nData access functions for\nUser and Item entities"]
            %% KIND: service_layer
            email_utils["Email Utilities\n[Component: emails / Jinja2]\nTemplate rendering and\nSMTP email dispatch"]
        end

        subgraph data ["Data Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models\n[Component: SQLModel]\nUser and Item table models\nwith Pydantic schemas"]
            %% KIND: data_access
            db_engine["Database Engine\n[Component: SQLAlchemy]\nConnection pool and\nsession management"]
            %% KIND: data_access
            alembic["Alembic Migrations\n[Component: Alembic]\nSchema versioning and\ndatabase migrations"]
        end

        %% KIND: service_layer
        config_mod["Configuration\n[Component: Pydantic Settings]\nEnvironment-based settings\nfor all services"]

    end

    spa -- "REST /api/v1" --> cors_mw
    cors_mw --> routes
    sentry_mw -. "wraps all requests" .-> routes

    login_routes --> auth_deps
    login_routes --> security_mod
    login_routes --> crud_layer
    login_routes --> email_utils

    users_routes --> auth_deps
    users_routes --> crud_layer
    users_routes --> email_utils

    items_routes --> auth_deps
    items_routes --> crud_layer

    utils_routes --> email_utils

    auth_deps --> security_mod
    auth_deps --> db_engine

    crud_layer --> models_layer
    crud_layer --> db_engine
    crud_layer --> security_mod

    email_utils --> smtp

    db_engine --> pg
    alembic --> pg

    sentry_mw --> sentry
    config_mod -. "provides settings to all" .-> routes
    config_mod -. "provides settings to all" .-> auth
    config_mod -. "provides settings to all" .-> services
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TB

    fastapi_api["⚙️ FastAPI Backend\n[Container]"]

    subgraph spa ["React SPA"]

        subgraph routing ["Routing"]
            %% KIND: router
            router["TanStack Router\n[Component: @tanstack/react-router]\nFile-based routing with\nauto code-splitting"]
            %% KIND: boundary
            auth_guard["Auth Guard\n[Component: beforeLoad hook]\nRedirects unauthenticated\nusers to /login"]
            %% KIND: router
            layout["Layout Shell\n[Component: _layout.tsx]\nSidebar + content area\nfor authenticated pages"]
        end

        subgraph state ["State & Data"]
            %% KIND: service_layer
            query_client["React Query Client\n[Component: @tanstack/react-query]\nServer state management,\ncaching, and error handling"]
            %% KIND: integration
            api_client["OpenAPI Client SDK\n[Component: @hey-api/openapi-ts]\nGenerated typed HTTP client\nwith Axios transport"]
            %% KIND: service_layer
            auth_hook["useAuth Hook\n[Component: React Hook]\nLogin, signup, logout,\ncurrent user queries"]
        end

        subgraph features ["Feature Modules"]
            %% KIND: service_layer
            items_feature["Items Module\n[Component: React]\nAdd, edit, delete items\nwith data table"]
            %% KIND: service_layer
            admin_feature["Admin Module\n[Component: React]\nUser management — add, edit,\ndelete, role assignment"]
            %% KIND: service_layer
            settings_feature["User Settings Module\n[Component: React]\nProfile info, password change,\naccount deletion"]
            %% KIND: service_layer
            auth_pages["Auth Pages\n[Component: React]\nLogin, signup, password\nrecovery and reset forms"]
        end

        subgraph ui ["UI Foundation"]
            %% KIND: boundary
            theme_provider["Theme Provider\n[Component: next-themes]\nLight / dark / system\ntheme switching"]
            %% KIND: boundary
            ui_lib["UI Component Library\n[Component: shadcn/ui + Radix]\nButtons, forms, dialogs,\ntables, toasts"]
        end

    end

    router --> auth_guard
    auth_guard --> layout
    layout --> features

    auth_pages --> auth_hook
    items_feature --> query_client
    admin_feature --> query_client
    settings_feature --> query_client
    auth_hook --> query_client

    query_client --> api_client
    api_client -- "REST /api/v1\nJSON + JWT" --> fastapi_api

    features --> ui_lib
    auth_pages --> ui_lib
    layout --> ui_lib
    theme_provider -. "provides theme context" .-> ui_lib
```

---

## L3: Traefik Proxy

```mermaid
%% SCOPE: urn:c4:container:traefik
flowchart TB

    user["👤 User\n[Person]"]
    admin_user["👤 Admin\n[Person]"]
    spa["🌐 React SPA\n[Container]"]
    fastapi_api["⚙️ FastAPI Backend\n[Container]"]

    subgraph traefik ["Traefik Proxy"]

        %% KIND: router
        entrypoints["Entrypoints\n[Component: Traefik]\nHTTP (:80) and HTTPS (:443)\nlisteners"]

        %% KIND: boundary
        https_redirect["HTTPS Redirect\n[Component: Traefik Middleware]\nRedirects HTTP to HTTPS"]

        %% KIND: integration
        tls_resolver["TLS Resolver\n[Component: Let's Encrypt]\nAutomatic certificate\nprovisioning via ACME"]

        %% KIND: router
        frontend_router["Frontend Router\n[Component: Traefik Router]\nHost: dashboard.* →\nfrontend:80"]

        %% KIND: router
        api_router["API Router\n[Component: Traefik Router]\nHost: api.* →\nbackend:8000"]

        %% KIND: router
        adminer_router["Adminer Router\n[Component: Traefik Router]\nHost: adminer.* →\nadminer:8080"]

    end

    user -- "HTTPS" --> entrypoints
    admin_user -- "HTTPS" --> entrypoints

    entrypoints --> https_redirect
    https_redirect --> tls_resolver

    tls_resolver --> frontend_router
    tls_resolver --> api_router
    tls_resolver --> adminer_router

    frontend_router --> spa
    api_router --> fastapi_api
```

---

## L3: PostgreSQL

```mermaid
%% SCOPE: urn:c4:container:pg
flowchart TB

    fastapi_api["⚙️ FastAPI Backend\n[Container]"]

    subgraph pg ["PostgreSQL"]

        %% KIND: storage
        user_table["User Table\n[Component: SQL Table]\nid (UUID), email, hashed_password,\nis_active, is_superuser, full_name, created_at"]

        %% KIND: storage
        item_table["Item Table\n[Component: SQL Table]\nid (UUID), title, description,\nowner_id (FK → user), created_at"]

        %% KIND: data_access
        pgdata_volume["Data Volume\n[Component: Docker Volume]\nPersistent storage at\n/var/lib/postgresql/data/pgdata"]

    end

    fastapi_api -- "SQL queries\npsycopg" --> user_table
    fastapi_api -- "SQL queries\npsycopg" --> item_table
    user_table -. "one-to-many\ncascade delete" .-> item_table
    user_table --> pgdata_volume
    item_table --> pgdata_volume
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers every subgraph and node across all levels
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ───────────────────────────────────────────
users            CONTAINS [user, admin_user]
system_boundary  CONTAINS [fastapi_project]
external         CONTAINS [smtp, sentry]

%% ── L2 system internals ──────────────────────────────────────────
system_boundary  CONTAINS [traefik, spa, fastapi_api, pg]

%% ── L3: FastAPI Backend (fastapi_api) ────────────────────────────
fastapi_api      CONTAINS [middleware, routes, auth, services, data, config_mod]
  middleware     CONTAINS [cors_mw, sentry_mw]
  routes         CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  auth           CONTAINS [auth_deps, security_mod]
  services       CONTAINS [crud_layer, email_utils]
  data           CONTAINS [models_layer, db_engine, alembic]

%% ── L3: React SPA (spa) ─────────────────────────────────────────
spa              CONTAINS [routing, state, features, ui]
  routing        CONTAINS [router, auth_guard, layout]
  state          CONTAINS [query_client, api_client, auth_hook]
  features       CONTAINS [items_feature, admin_feature, settings_feature, auth_pages]
  ui             CONTAINS [theme_provider, ui_lib]

%% ── L3: Traefik Proxy (traefik) ─────────────────────────────────
traefik          CONTAINS [entrypoints, https_redirect, tls_resolver, frontend_router, api_router, adminer_router]

%% ── L3: PostgreSQL (pg) ─────────────────────────────────────────
pg               CONTAINS [user_table, item_table, pgdata_volume]
```
