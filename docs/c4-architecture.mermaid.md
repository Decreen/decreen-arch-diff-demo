# C4 Architecture Model — Full Stack FastAPI Project

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI Project.

---

## L1: System Context

```mermaid
flowchart TD
    %% SCOPE: urn:c4:system:full-stack-fastapi-project

    subgraph users["Users"]
        anon_user["Anonymous User
        <i>[Person]</i>
        Unauthenticated visitor"]
        auth_user["Authenticated User
        <i>[Person]</i>
        Logged-in application user"]
        admin_user["Superuser / Admin
        <i>[Person]</i>
        Privileged administrator"]
    end

    subgraph system_boundary["Full Stack FastAPI Project"]
        system["Full Stack FastAPI Project
        <i>[Software System]</i>
        Full-stack web application with
        REST API, SPA dashboard, and
        PostgreSQL persistence"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server
        <i>[External System]</i>
        Transactional email delivery"]
        sentry["Sentry
        <i>[External System]</i>
        Error monitoring and tracing"]
    end

    anon_user -->|"Signs up, logs in,
    recovers password"| system
    auth_user -->|"Manages items,
    edits profile"| system
    admin_user -->|"Manages users,
    all items, test emails"| system
    system -->|"Sends emails via SMTP"| smtp
    system -->|"Reports errors and traces"| sentry
```

---

## L2: Container

```mermaid
flowchart TD
    %% SCOPE: urn:c4:system:full-stack-fastapi-project

    anon_user["Anonymous User
    <i>[Person]</i>"]
    auth_user["Authenticated User
    <i>[Person]</i>"]
    admin_user["Superuser / Admin
    <i>[Person]</i>"]

    subgraph system_boundary["Full Stack FastAPI Project"]
        traefik["Traefik
        <i>[Container: Reverse Proxy]</i>
        Routes requests, TLS termination,
        HTTP-to-HTTPS redirect"]

        spa["React SPA
        <i>[Container: React 19 / Vite / Nginx]</i>
        Dashboard, item management,
        admin panel, user settings"]

        fastapi_backend["FastAPI Backend
        <i>[Container: Python / FastAPI / Uvicorn]</i>
        REST API, JWT auth, business logic,
        email dispatch"]

        pg["PostgreSQL
        <i>[Container: PostgreSQL 18]</i>
        Stores users, items,
        and application data"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server
        <i>[External System]</i>
        Transactional email delivery"]
        sentry["Sentry
        <i>[External System]</i>
        Error monitoring and tracing"]
    end

    anon_user -->|"HTTPS"| traefik
    auth_user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"Routes dashboard.*
    requests to Nginx"| spa
    traefik -->|"Routes api.*
    requests to Uvicorn"| fastapi_backend
    spa -->|"JSON/HTTPS
    via generated API client"| fastapi_backend
    fastapi_backend -->|"SQL over TCP
    (psycopg driver)"| pg
    fastapi_backend -->|"SMTP
    (emails library)"| smtp
    fastapi_backend -->|"HTTPS
    (sentry-sdk)"| sentry
```

---

## L3: FastAPI Backend

```mermaid
flowchart TD
    %% SCOPE: urn:c4:container:fastapi_backend

    spa["React SPA
    <i>[Container]</i>"]
    pg["PostgreSQL
    <i>[Container]</i>"]
    smtp["SMTP Server
    <i>[External System]</i>"]
    sentry["Sentry
    <i>[External System]</i>"]

    subgraph fastapi_backend["FastAPI Backend"]

        %% KIND: boundary
        subgraph mw["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware
            <i>[Component: Starlette]</i>
            Enforces allowed origins,
            methods, and headers"]
        end

        %% KIND: router
        subgraph routes["API Routes (/api/v1)"]
            login_routes["Login Routes
            <i>[Component: /login/*]</i>
            OAuth2 token, password
            recovery and reset"]
            user_routes["User Routes
            <i>[Component: /users/*]</i>
            Registration, profile,
            user CRUD (admin)"]
            item_routes["Item Routes
            <i>[Component: /items/*]</i>
            Item CRUD, ownership
            scoped queries"]
            util_routes["Utils Routes
            <i>[Component: /utils/*]</i>
            Health check, test email"]
        end

        %% KIND: service_layer
        subgraph deps["Dependencies"]
            auth_dep["Auth Dependency
            <i>[Component: deps.py]</i>
            JWT validation, user
            extraction from token"]
            db_session_dep["DB Session
            <i>[Component: deps.py]</i>
            SQLModel session-per-request
            via FastAPI Depends"]
        end

        %% KIND: data_access
        crud_layer["CRUD Layer
        <i>[Component: crud.py]</i>
        User and item data access,
        authentication logic"]

        %% KIND: service_layer
        subgraph core["Core"]
            security_mod["Security
            <i>[Component: core/security.py]</i>
            JWT creation, Argon2/Bcrypt
            password hashing"]
            config_mod["Config
            <i>[Component: core/config.py]</i>
            Pydantic settings from
            environment variables"]
            db_engine["DB Engine
            <i>[Component: core/db.py]</i>
            SQLAlchemy engine,
            initial superuser seeding"]
        end

        %% KIND: integration
        email_svc["Email Service
        <i>[Component: utils.py]</i>
        Jinja2 template rendering,
        SMTP sending, token generation"]

        %% KIND: storage
        models_layer["Models
        <i>[Component: models.py]</i>
        SQLModel schemas for User,
        Item, Token, and DTOs"]
    end

    spa -->|"JSON/HTTPS"| cors_mw
    cors_mw --> routes
    login_routes --> auth_dep
    login_routes --> crud_layer
    login_routes --> email_svc
    user_routes --> auth_dep
    user_routes --> crud_layer
    user_routes --> email_svc
    item_routes --> auth_dep
    item_routes --> crud_layer
    util_routes --> email_svc
    auth_dep --> security_mod
    auth_dep --> db_session_dep
    auth_dep --> models_layer
    crud_layer --> models_layer
    crud_layer --> security_mod
    db_session_dep --> db_engine
    db_engine --> config_mod
    security_mod --> config_mod
    email_svc --> config_mod
    email_svc -->|"SMTP"| smtp
    db_engine -->|"SQL over TCP"| pg
    fastapi_backend -.->|"sentry-sdk"| sentry
```

---

## L3: React SPA

```mermaid
flowchart TD
    %% SCOPE: urn:c4:container:spa

    fastapi_backend["FastAPI Backend
    <i>[Container]</i>"]
    traefik["Traefik
    <i>[Container]</i>"]

    subgraph spa["React SPA"]

        %% KIND: router
        subgraph routing["Routing (TanStack Router)"]
            root_layout["Root Layout
            <i>[Component: __root.tsx]</i>
            Error boundary,
            not-found fallback"]
            protected_layout["Protected Layout
            <i>[Component: _layout.tsx]</i>
            Auth guard, sidebar,
            header, footer"]
            auth_pages["Auth Pages
            <i>[Component: login, signup,
            recover-password, reset-password]</i>
            Public authentication flows"]
        end

        %% KIND: service_layer
        subgraph features["Features"]
            dashboard_page["Dashboard
            <i>[Component: _layout/index.tsx]</i>
            Welcome greeting,
            user overview"]
            items_feature["Items Management
            <i>[Component: Items/*]</i>
            CRUD, data table,
            owner-scoped listing"]
            admin_feature["Admin Panel
            <i>[Component: Admin/*]</i>
            User management
            (superuser only)"]
            settings_feature["User Settings
            <i>[Component: UserSettings/*]</i>
            Profile edit, password
            change, account deletion"]
        end

        %% KIND: integration
        api_client["API Client
        <i>[Component: openapi-ts / Axios]</i>
        Auto-generated TypeScript SDK
        from OpenAPI schema"]

        %% KIND: service_layer
        subgraph hooks["Hooks"]
            auth_hook["useAuth
            <i>[Component: hooks/useAuth.ts]</i>
            Token management,
            login/logout helpers"]
            toast_hook["useCustomToast
            <i>[Component: hooks/useCustomToast.ts]</i>
            Notification helpers"]
        end

        %% KIND: boundary
        subgraph ui_lib["UI Library"]
            shadcn_components["shadcn/ui Primitives
            <i>[Component: components/ui/*]</i>
            Radix UI, Tailwind CSS,
            forms, dialogs, tables"]
            sidebar_component["Sidebar
            <i>[Component: Sidebar/*]</i>
            App navigation,
            user menu"]
            data_table["DataTable
            <i>[Component: Common/DataTable.tsx]</i>
            TanStack Table wrapper
            with pagination"]
        end

        %% KIND: service_layer
        theme_provider["Theme Provider
        <i>[Component: theme-provider.tsx]</i>
        Dark/light mode via next-themes"]
    end

    traefik -->|"Serves static assets"| spa
    root_layout --> protected_layout
    root_layout --> auth_pages
    protected_layout --> features
    auth_pages --> api_client
    auth_pages --> auth_hook
    auth_pages --> ui_lib
    features --> api_client
    features --> auth_hook
    features --> toast_hook
    items_feature --> data_table
    admin_feature --> data_table
    features --> shadcn_components
    api_client -->|"JSON/HTTPS"| fastapi_backend
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Project
%% Every subgraph and its direct children across all diagrams.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [anon_user, auth_user, admin_user]
system_boundary  CONTAINS [system]
external         CONTAINS [smtp, sentry]

%% ── L2 system boundary (containers) ─────────────────────────────
system_boundary  CONTAINS [traefik, spa, fastapi_backend, pg]

%% ── L3: FastAPI Backend ─────────────────────────────────────────
fastapi_backend  CONTAINS [mw, routes, deps, crud_layer, core, email_svc, models_layer]
  mw             CONTAINS [cors_mw]
  routes         CONTAINS [login_routes, user_routes, item_routes, util_routes]
  deps           CONTAINS [auth_dep, db_session_dep]
  core           CONTAINS [security_mod, config_mod, db_engine]

%% ── L3: React SPA ──────────────────────────────────────────────
spa              CONTAINS [routing, features, api_client, hooks, ui_lib, theme_provider]
  routing        CONTAINS [root_layout, protected_layout, auth_pages]
  features       CONTAINS [dashboard_page, items_feature, admin_feature, settings_feature]
  hooks          CONTAINS [auth_hook, toast_hook]
  ui_lib         CONTAINS [shadcn_components, sidebar_component, data_table]
```
