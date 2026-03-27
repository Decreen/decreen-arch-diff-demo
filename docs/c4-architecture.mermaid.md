# C4 Architecture Model — Full Stack FastAPI Platform

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI template.

---

## L1: System Context

```mermaid
---
title: "L1: System Context — Full Stack FastAPI Platform"
---
flowchart TD
    %% SCOPE: urn:c4:system:full-stack-fastapi

    user["End User
    [Person]
    Browses the dashboard,
    manages personal items"]

    admin_user["Administrator
    [Person]
    Manages users,
    system configuration"]

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI
        [Software System]
        Web application with user auth,
        item management, and admin tools"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server
        [External System]
        Delivers transactional email
        (password reset, welcome, test)"]

        sentry["Sentry
        [External System]
        Error tracking and
        performance monitoring"]
    end

    user -->|"Interacts via web browser"| platform
    admin_user -->|"Administers via web browser"| platform
    platform -->|"Sends emails via SMTP"| smtp
    platform -->|"Reports errors (staging/prod)"| sentry
```

---

## L2: Container Diagram

```mermaid
---
title: "L2: Container Diagram — Full Stack FastAPI Platform"
---
flowchart TD
    %% SCOPE: urn:c4:system:full-stack-fastapi

    user["End User
    [Person]"]

    admin_user["Administrator
    [Person]"]

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["traefik
        [Container: Traefik v3]
        Reverse proxy, TLS termination,
        routing to backend and frontend"]

        spa["spa
        [Container: React 19 / Vite / nginx]
        Single-page application served
        by nginx; TanStack Router + Query"]

        fastapi_backend["fastapi_backend
        [Container: Python / FastAPI]
        REST API with JWT auth,
        CRUD operations, email utils"]

        pg["pg
        [Container: PostgreSQL 18]
        Stores users, items,
        Alembic-managed schema"]

        adminer["adminer
        [Container: Adminer]
        Lightweight DB admin UI
        for ops/debugging"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server
        [External System]"]

        sentry["Sentry
        [External System]"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"dashboard.DOMAIN :80"| spa
    traefik -->|"api.DOMAIN :8000"| fastapi_backend
    traefik -->|"adminer.DOMAIN :8080"| adminer
    spa -->|"REST / JSON via
    OpenAPI Axios client"| fastapi_backend
    fastapi_backend -->|"SQL via SQLAlchemy
    (psycopg driver)"| pg
    fastapi_backend -->|"SMTP"| smtp
    fastapi_backend -->|"Sentry SDK"| sentry
    adminer -->|"PostgreSQL protocol"| pg
```

---

## L3: FastAPI Backend

```mermaid
---
title: "L3: FastAPI Backend — Component Diagram"
---
flowchart TD
    %% SCOPE: urn:c4:container:fastapi-backend

    spa["spa
    [Container: React SPA]"]

    subgraph fastapi_backend["FastAPI Backend"]

        %% KIND: boundary
        cors_mw["cors_mw
        [Component: CORSMiddleware]
        Starlette CORS middleware;
        allows configured origins"]

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["login_routes
            [Component: Login Router]
            OAuth2 password flow, JWT issuance,
            password recovery/reset"]

            %% KIND: router
            users_routes["users_routes
            [Component: Users Router]
            CRUD for user accounts,
            signup, profile, admin ops"]

            %% KIND: router
            items_routes["items_routes
            [Component: Items Router]
            CRUD for items owned by
            authenticated users"]

            %% KIND: router
            utils_routes["utils_routes
            [Component: Utils Router]
            Health check endpoint,
            test-email (superuser)"]
        end

        %% KIND: service_layer
        deps["deps
        [Component: API Dependencies]
        OAuth2 bearer extraction,
        get_current_user, DB session provider"]

        %% KIND: data_access
        crud_layer["crud_layer
        [Component: CRUD Operations]
        create/update/get user, authenticate,
        create item — DB session helpers"]

        %% KIND: data_access
        models_layer["models_layer
        [Component: SQLModel Domain Models]
        User, Item tables + Pydantic
        request/response schemas"]

        subgraph core["Core"]
            %% KIND: service_layer
            core_security["core_security
            [Component: Security]
            JWT encode/decode, Argon2+Bcrypt
            password hashing (pwdlib)"]

            %% KIND: service_layer
            core_config["core_config
            [Component: Configuration]
            pydantic-settings: DB, SMTP,
            CORS, Sentry, secrets"]

            %% KIND: storage
            core_db["core_db
            [Component: DB Engine]
            SQLAlchemy engine creation,
            init_db (first superuser seed)"]
        end

        %% KIND: integration
        email_utils["email_utils
        [Component: Email Utilities]
        Jinja2 + MJML templates;
        password reset, welcome, test emails"]

    end

    pg["pg
    [Container: PostgreSQL 18]"]

    smtp["SMTP Server
    [External System]"]

    sentry["Sentry
    [External System]"]

    spa -->|"REST requests"| cors_mw
    cors_mw --> routes
    login_routes --> deps
    users_routes --> deps
    items_routes --> deps
    utils_routes --> deps
    deps -->|"Validates JWT"| core_security
    deps -->|"Opens DB session"| core_db
    login_routes --> crud_layer
    users_routes --> crud_layer
    items_routes --> crud_layer
    login_routes --> email_utils
    crud_layer --> models_layer
    crud_layer --> core_security
    models_layer --> core_db
    core_db -->|"SQL via psycopg"| pg
    core_config -->|"Provides DSN"| core_db
    email_utils -->|"SMTP send"| smtp
    fastapi_backend -->|"Sentry SDK init"| sentry
```

---

## L3: React SPA

```mermaid
---
title: "L3: React SPA — Component Diagram"
---
flowchart TD
    %% SCOPE: urn:c4:container:spa

    user["End User
    [Person]"]

    admin_user["Administrator
    [Person]"]

    subgraph spa["React SPA"]

        %% KIND: router
        router["router
        [Component: TanStack Router]
        File-based routing with auth
        guards (beforeLoad redirect)"]

        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            auth_hook["auth_hook
            [Component: useAuth Hook]
            Login/signup mutations,
            current-user query, logout"]

            %% KIND: data_access
            query_client["query_client
            [Component: TanStack Query Client]
            Server-state cache, automatic
            401/403 error handler"]

            %% KIND: integration
            openapi_client["openapi_client
            [Component: OpenAPI Axios Client]
            Auto-generated from backend
            OpenAPI schema; typed SDK"]
        end

        subgraph features["Feature Modules"]
            %% KIND: service_layer
            items_feature["items_feature
            [Component: Items Management]
            Add/Edit/Delete items,
            data table with columns"]

            %% KIND: service_layer
            admin_feature["admin_feature
            [Component: Admin Panel]
            User CRUD for superusers,
            data table with actions"]

            %% KIND: service_layer
            settings_feature["settings_feature
            [Component: User Settings]
            Profile edit, change password,
            delete account, appearance"]
        end

        subgraph ui_layer["UI Layer"]
            %% KIND: boundary
            ui_components["ui_components
            [Component: Shadcn/Radix Primitives]
            Button, Dialog, Form, Table,
            Select, Tabs, Tooltip, etc."]

            %% KIND: boundary
            sidebar_comp["sidebar_comp
            [Component: Sidebar Navigation]
            App navigation, user menu,
            theme toggle"]

            %% KIND: service_layer
            theme_provider["theme_provider
            [Component: Theme Provider]
            Dark/light mode via
            next-themes library"]
        end

    end

    fastapi_backend["fastapi_backend
    [Container: FastAPI Backend]"]

    user -->|"Interacts via browser"| router
    admin_user -->|"Interacts via browser"| router
    router --> features
    router --> sidebar_comp
    items_feature --> auth_hook
    admin_feature --> auth_hook
    settings_feature --> auth_hook
    auth_hook --> query_client
    items_feature --> query_client
    admin_feature --> query_client
    settings_feature --> query_client
    query_client --> openapi_client
    openapi_client -->|"REST / JSON calls
    to /api/v1/*"| fastapi_backend
    features --> ui_components
    sidebar_comp --> ui_components
    sidebar_comp --> theme_provider
```

---

## L3: Traefik Reverse Proxy

```mermaid
---
title: "L3: Traefik Reverse Proxy — Component Diagram"
---
flowchart TD
    %% SCOPE: urn:c4:container:traefik

    user["End User
    [Person]"]

    subgraph traefik["Traefik Reverse Proxy"]

        %% KIND: boundary
        entrypoints["entrypoints
        [Component: Entrypoints]
        HTTP (:80) and HTTPS (:443)
        listeners"]

        %% KIND: boundary
        https_redirect["https_redirect
        [Component: HTTPS Redirect Middleware]
        Redirects all HTTP traffic
        to HTTPS"]

        %% KIND: router
        frontend_router["frontend_router
        [Component: Frontend Router]
        Host rule: dashboard.DOMAIN
        routes to SPA container"]

        %% KIND: router
        backend_router["backend_router
        [Component: Backend Router]
        Host rule: api.DOMAIN
        routes to FastAPI container"]

        %% KIND: router
        adminer_router["adminer_router
        [Component: Adminer Router]
        Host rule: adminer.DOMAIN
        routes to Adminer container"]

        %% KIND: integration
        tls_resolver["tls_resolver
        [Component: Let's Encrypt Resolver]
        Automatic TLS certificate
        provisioning via ACME"]

    end

    spa["spa
    [Container: React SPA]"]

    fastapi_backend["fastapi_backend
    [Container: FastAPI Backend]"]

    adminer["adminer
    [Container: Adminer]"]

    user -->|"HTTPS"| entrypoints
    entrypoints --> https_redirect
    https_redirect --> frontend_router
    https_redirect --> backend_router
    https_redirect --> adminer_router
    entrypoints --> tls_resolver
    frontend_router -->|":80"| spa
    backend_router -->|":8000"| fastapi_backend
    adminer_router -->|":8080"| adminer
```

---

<!-- ============================================================
     CONTAINMENT MAP
     Every subgraph and node→parent relationship from all diagrams.
     The parser uses this to build the full parent→child hierarchy.
     ============================================================ -->

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
user             CONTAINS []
admin_user       CONTAINS []
platform_boundary CONTAINS [platform]
external         CONTAINS [smtp, sentry]

%% ── L2 container-level ──────────────────────────────────────────
platform_boundary CONTAINS [traefik, spa, fastapi_backend, pg, adminer]

%% ── L3: fastapi_backend internals ───────────────────────────────
fastapi_backend  CONTAINS [cors_mw, routes, deps, crud_layer, models_layer, core, email_utils]
  routes         CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  core           CONTAINS [core_security, core_config, core_db]

%% ── L3: spa internals ──────────────────────────────────────────
spa              CONTAINS [router, state_mgmt, features, ui_layer]
  state_mgmt     CONTAINS [auth_hook, query_client, openapi_client]
  features       CONTAINS [items_feature, admin_feature, settings_feature]
  ui_layer       CONTAINS [ui_components, sidebar_comp, theme_provider]

%% ── L3: traefik internals ──────────────────────────────────────
traefik          CONTAINS [entrypoints, https_redirect, frontend_router, backend_router, adminer_router, tls_resolver]
```
