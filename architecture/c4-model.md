# C4 Architecture Model — Full Stack FastAPI Template

## L1: System Context

```mermaid
---
title: "L1: System Context — Full Stack FastAPI Template"
---
flowchart TD
    %% SCOPE: urn:c4:system-landscape

    subgraph users["Users"]
        user["User
        Regular application user who
        manages personal items"]
        admin_user["Admin
        Superuser who manages
        all users and system"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Template"]
        fullstack_app["Full Stack FastAPI App
        Web application providing user
        registration, auth, and item management"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server
        Email delivery for password
        recovery and notifications"]
        sentry["Sentry
        Error tracking
        and performance monitoring"]
    end

    user -- "Manages personal items
    HTTPS" --> fullstack_app
    admin_user -- "Manages users and items
    HTTPS" --> fullstack_app
    fullstack_app -- "Sends transactional emails
    SMTP / TLS" --> smtp
    fullstack_app -- "Reports errors
    HTTPS" --> sentry
```

---

## L2: Container Diagram

```mermaid
---
title: "L2: Container Diagram — Full Stack FastAPI Template"
---
flowchart TD
    %% SCOPE: urn:c4:system:fullstack_app

    user["User
    Regular application user"]
    admin_user["Admin
    Superuser / administrator"]

    subgraph fullstack_boundary["Full Stack FastAPI Template"]
        traefik["Traefik
        Reverse proxy and load balancer
        TLS termination, Let's Encrypt
        Ports 80 / 443"]

        spa["React SPA
        Vite + React 19 + TanStack Router
        shadcn/ui + Tailwind CSS
        Served by Nginx on port 80"]

        fastapi_backend["FastAPI Backend
        Python REST API
        SQLModel ORM, JWT auth
        Port 8000"]

        pg["PostgreSQL 18
        Relational database
        Users, Items tables
        Port 5432"]
    end

    smtp["SMTP Server
    Email delivery service"]
    sentry["Sentry
    Error tracking"]

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik
    traefik -- "dashboard.* routes
    HTTP" --> spa
    traefik -- "api.* routes
    HTTP" --> fastapi_backend
    spa -. "API calls from browser
    JSON / HTTPS via Traefik" .-> fastapi_backend
    fastapi_backend -- "SQL queries
    psycopg (async)" --> pg
    fastapi_backend -- "Sends emails
    SMTP / TLS" --> smtp
    fastapi_backend -- "Reports errors
    HTTPS" --> sentry
```

---

## L3: FastAPI Backend — Component Diagram

```mermaid
---
title: "L3: FastAPI Backend — Components"
---
flowchart TD
    %% SCOPE: urn:c4:container:fastapi_backend

    spa["React SPA"]
    pg["PostgreSQL 18"]
    smtp["SMTP Server"]
    sentry["Sentry"]

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph mw["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware
            Allows configured origins
            Credentials, methods, headers"]
        end

        subgraph routes["API Routes — /api/v1"]
            %% KIND: router
            login_routes["Login Routes
            POST /login/access-token
            POST /password-recovery
            POST /reset-password"]

            %% KIND: router
            users_routes["Users Routes
            CRUD /users, /users/me
            POST /users/signup
            PATCH /users/me/password"]

            %% KIND: router
            items_routes["Items Routes
            CRUD /items
            Owner-scoped access control"]

            %% KIND: router
            utils_routes["Utils Routes
            POST /utils/test-email
            GET /utils/health-check"]
        end

        subgraph services["Business Logic"]
            %% KIND: boundary
            auth_deps["Auth Dependencies
            OAuth2 bearer token extraction
            JWT decode and user resolution
            Superuser permission guard"]

            %% KIND: data_access
            crud_layer["CRUD Layer
            create/update/get user
            authenticate user
            create item"]

            %% KIND: integration
            email_utils["Email Utilities
            Jinja2 template rendering
            Password reset token generation
            SMTP email dispatch"]
        end

        subgraph data["Data Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models
            User, Item tables
            Pydantic schemas for
            request/response validation"]

            %% KIND: storage
            db_engine["Database Engine
            SQLAlchemy engine
            Session management
            Connection to PostgreSQL"]
        end

        subgraph core["Core Configuration"]
            %% KIND: service_layer
            security_module["Security Module
            JWT token creation (HS256)
            Argon2 + Bcrypt password hashing
            Password verification"]

            %% KIND: service_layer
            config_settings["Settings
            Pydantic BaseSettings
            Environment-driven config
            DB, SMTP, CORS, secrets"]
        end
    end

    spa -. "HTTP requests
    JSON" .-> cors_mw
    cors_mw --> routes
    routes --> auth_deps
    auth_deps --> security_module
    auth_deps --> db_engine
    login_routes --> crud_layer
    users_routes --> crud_layer
    items_routes --> crud_layer
    crud_layer --> models_layer
    models_layer --> db_engine
    db_engine --> pg
    email_utils --> smtp
    login_routes --> email_utils
    users_routes --> email_utils
    config_settings -. "provides config to" .-> security_module
    config_settings -. "provides config to" .-> db_engine
    fastapi_backend -. "Sentry SDK
    auto-instrumentation" .-> sentry
```

---

## L3: React SPA — Component Diagram

```mermaid
---
title: "L3: React SPA — Components"
---
flowchart TD
    %% SCOPE: urn:c4:container:spa

    user["User"]
    admin_user["Admin"]
    fastapi_backend["FastAPI Backend"]

    subgraph spa["React SPA"]

        subgraph routing["Routing Layer"]
            %% KIND: router
            tanstack_router["TanStack Router
            File-based route tree
            Type-safe navigation
            Auth-guarded layout"]
        end

        subgraph data_layer["Data & State Management"]
            %% KIND: service_layer
            tanstack_query["TanStack Query
            Server state cache
            Query/mutation management
            Auto error handling (401/403)"]

            %% KIND: integration
            api_client["OpenAPI Client
            Auto-generated from backend schema
            Typed SDK for all endpoints
            Axios-based HTTP transport"]
        end

        subgraph pages["Page Routes"]
            %% KIND: router
            login_page["Login Page
            OAuth2 password form
            Token storage in localStorage"]

            %% KIND: router
            signup_page["Signup Page
            Self-service registration"]

            %% KIND: router
            dashboard_page["Dashboard / Items Page
            Item CRUD with data table
            Owner-scoped item listing"]

            %% KIND: router
            admin_page["Admin Page
            User management table
            Add / edit / delete users"]

            %% KIND: router
            settings_page["Settings Page
            User profile editing
            Password change
            Account deletion"]

            %% KIND: router
            password_pages["Password Recovery Pages
            Recover password flow
            Reset password with token"]
        end

        subgraph features["Feature Components"]
            %% KIND: service_layer
            admin_features["Admin Components
            AddUser, EditUser, DeleteUser
            User columns, actions menu"]

            %% KIND: service_layer
            item_features["Item Components
            AddItem, EditItem, DeleteItem
            Item columns, actions menu"]

            %% KIND: service_layer
            settings_features["User Settings Components
            UserInformation, ChangePassword
            DeleteAccount, DeleteConfirmation"]
        end

        subgraph common["Shared / Common"]
            %% KIND: boundary
            layout_components["Layout & Sidebar
            AppSidebar, AuthLayout
            DataTable, Footer, Logo
            Error & NotFound components"]

            %% KIND: boundary
            ui_library["UI Component Library
            shadcn/ui + Radix primitives
            Button, Dialog, Form, Table
            Input, Select, Tabs, etc."]

            %% KIND: service_layer
            theme_provider["Theme Provider
            Dark / light mode toggle
            next-themes integration"]
        end

        subgraph hooks["Custom Hooks"]
            %% KIND: service_layer
            auth_hook["useAuth Hook
            Login / logout / signup mutations
            Current user query
            Token management"]
        end
    end

    user -- "Browses dashboard" --> tanstack_router
    admin_user -- "Manages users" --> tanstack_router
    tanstack_router --> pages
    pages --> features
    pages --> common
    features --> ui_library
    pages --> tanstack_query
    auth_hook --> tanstack_query
    tanstack_query --> api_client
    api_client -- "REST API calls
    JSON / HTTPS" --> fastapi_backend
    pages --> auth_hook
    features --> auth_hook
    layout_components --> auth_hook
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Template
%% Covers L1 → L2 → L3. Every subgraph and node listed.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin_user]
fullstack_boundary CONTAINS [traefik, spa, fastapi_backend, pg]
external           CONTAINS [smtp, sentry]

%% ── L2 containers (inside fullstack_boundary) ──────────────────────
%%   traefik        — no internal structure (config-driven proxy)
%%   pg             — no internal structure (managed database)

%% ── L3: fastapi_backend components ─────────────────────────────────
fastapi_backend CONTAINS [mw, routes, services, data, core]
  mw            CONTAINS [cors_mw]
  routes        CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services      CONTAINS [auth_deps, crud_layer, email_utils]
  data          CONTAINS [models_layer, db_engine]
  core          CONTAINS [security_module, config_settings]

%% ── L3: spa components ─────────────────────────────────────────────
spa             CONTAINS [routing, data_layer, pages, features, common, hooks]
  routing       CONTAINS [tanstack_router]
  data_layer    CONTAINS [tanstack_query, api_client]
  pages         CONTAINS [login_page, signup_page, dashboard_page, admin_page, settings_page, password_pages]
  features      CONTAINS [admin_features, item_features, settings_features]
  common        CONTAINS [layout_components, ui_library, theme_provider]
  hooks         CONTAINS [auth_hook]
```
