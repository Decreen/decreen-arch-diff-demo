# C4 Architecture — Full Stack FastAPI Project

> Auto-generated C4 architecture model.
> Diagrams follow the C4 hierarchy: L1 System Context → L2 Container → L3 Component.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack_platform
graph TB

    subgraph users["Users"]
        user["User
        <i>Application end-user who manages
        personal items and profile</i>"]
        admin_user["Administrator
        <i>Superuser with elevated privileges
        for user and system management</i>"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Platform"]
        fullstack_system["Full Stack FastAPI Platform
        <i>Web application providing user management,
        item CRUD, and admin capabilities</i>"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server
        <i>Transactional email delivery
        (password reset, notifications)</i>"]
        sentry["Sentry
        <i>Error monitoring and
        performance tracing</i>"]
    end

    user -- "Uses web interface" --> fullstack_system
    admin_user -- "Manages users and configuration" --> fullstack_system
    fullstack_system -- "Sends transactional email via SMTP" --> smtp
    fullstack_system -- "Reports errors and traces" --> sentry
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fullstack_platform
graph TB

    user["User"]
    admin_user["Administrator"]

    subgraph fullstack_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik
        <i>Reverse proxy, TLS termination,
        host-based routing (v3)</i>"]
        spa["React SPA
        <i>React 19 + Vite + TanStack Router
        Served via Nginx container</i>"]
        fastapi_api["FastAPI Backend
        <i>Python REST API at /api/v1
        Auth, users, items, utilities</i>"]
        pg[("PostgreSQL
        <i>Primary relational data store
        (v18, psycopg driver)</i>")]
        adminer["Adminer
        <i>Database administration UI</i>"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server"]
        sentry["Sentry"]
    end

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik
    traefik -- "dashboard.* host" --> spa
    traefik -- "api.* host" --> fastapi_api
    traefik -- "adminer.* host" --> adminer
    spa -. "REST / JSON via Axios
    (OpenAPI-generated client)" .-> fastapi_api
    fastapi_api -- "SQL via SQLModel
    (postgresql+psycopg)" --> pg
    fastapi_api -- "SMTP" --> smtp
    fastapi_api -. "Sentry SDK" .-> sentry
    adminer -- "SQL" --> pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_api
graph TB

    spa["React SPA"]
    pg[("PostgreSQL")]
    smtp["SMTP Server"]
    sentry["Sentry"]

    subgraph fastapi_api["FastAPI Backend"]

        %% KIND: boundary
        cors_mw["CORS Middleware
        <i>Starlette CORSMiddleware
        Origin validation, credential handling</i>"]

        %% KIND: router
        subgraph routes["API Routes (/api/v1)"]
            %% KIND: router
            login_routes["Login Routes
            <i>POST /login/access-token
            Password recovery and reset</i>"]
            %% KIND: router
            users_routes["Users Routes
            <i>User CRUD, POST /users/signup
            GET-PATCH-DELETE /users/me</i>"]
            %% KIND: router
            items_routes["Items Routes
            <i>Item CRUD with owner-scoped
            access control</i>"]
            %% KIND: router
            utils_routes["Utils Routes
            <i>GET /utils/health-check
            POST /utils/test-email</i>"]
        end

        %% KIND: boundary
        auth_deps["Auth Dependencies
        <i>OAuth2PasswordBearer, JWT decode,
        get_current_user, superuser guard</i>"]

        %% KIND: service_layer
        subgraph services["Business Logic"]
            %% KIND: data_access
            crud_layer["CRUD Layer
            <i>create_user, authenticate,
            create_item, update_user</i>"]
            %% KIND: service_layer
            security_mod["Security Module
            <i>JWT creation (HS256),
            Argon2 + Bcrypt hashing (pwdlib)</i>"]
            %% KIND: integration
            email_utils["Email Utilities
            <i>Jinja2 HTML templates,
            SMTP transport via emails lib</i>"]
        end

        %% KIND: data_access
        subgraph data["Data Access Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models
            <i>User, Item entities with UUID PKs,
            relationships, and cascades</i>"]
            %% KIND: storage
            db_engine["Database Engine
            <i>SQLAlchemy create_engine,
            per-request Session via get_db()</i>"]
        end

        %% KIND: boundary
        config["Configuration
        <i>Pydantic BaseSettings
        .env loading, computed fields</i>"]

    end

    spa -. "HTTP requests" .-> cors_mw
    cors_mw --> routes
    login_routes --> auth_deps
    login_routes --> crud_layer
    login_routes --> security_mod
    users_routes --> auth_deps
    users_routes --> crud_layer
    items_routes --> auth_deps
    items_routes --> crud_layer
    utils_routes --> email_utils
    auth_deps --> security_mod
    auth_deps --> db_engine
    crud_layer --> models_layer
    crud_layer --> security_mod
    models_layer --> db_engine
    db_engine -- "SQL" --> pg
    email_utils -- "SMTP" --> smtp
    fastapi_api -. "error reports" .-> sentry
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

    user["User"]
    fastapi_api["FastAPI Backend"]

    subgraph spa["React SPA"]

        %% KIND: router
        tanstack_router["TanStack Router
        <i>File-based routing, auth guards
        in beforeLoad, code splitting</i>"]

        %% KIND: service_layer
        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            auth_hook["useAuth Hook
            <i>Login / logout / signup,
            JWT in localStorage</i>"]
            %% KIND: data_access
            query_layer["TanStack Query
            <i>Server state caching,
            background refetch, mutations</i>"]
        end

        %% KIND: integration
        api_client["API Client
        <i>@hey-api/openapi-ts generated SDK,
        Axios HTTP transport, Bearer token</i>"]

        %% KIND: boundary
        subgraph features["Feature Pages"]
            %% KIND: boundary
            dashboard_feature["Dashboard
            <i>Home / overview index page</i>"]
            %% KIND: boundary
            admin_feature["Admin Panel
            <i>User management table
            (superuser only)</i>"]
            %% KIND: boundary
            items_feature["Items Manager
            <i>Item CRUD with DataTable,
            owner-scoped views</i>"]
            %% KIND: boundary
            settings_feature["Settings
            <i>Profile info, change password,
            delete account</i>"]
            %% KIND: boundary
            auth_pages["Auth Pages
            <i>Login, signup, recover-password,
            reset-password</i>"]
        end

        %% KIND: boundary
        subgraph ui_framework["UI Framework"]
            %% KIND: boundary
            ui_lib["UI Components
            <i>shadcn/ui primitives (Radix),
            Tailwind CSS 4</i>"]
            %% KIND: service_layer
            theme_provider["Theme Provider
            <i>next-themes: light / dark /
            system mode</i>"]
            %% KIND: boundary
            sidebar_nav["Sidebar Navigation
            <i>AppSidebar with route links,
            collapsible SidebarProvider</i>"]
        end

    end

    user -- "Interacts via browser" --> tanstack_router
    tanstack_router --> auth_hook
    tanstack_router --> features
    auth_pages --> auth_hook
    features --> query_layer
    features --> ui_lib
    features --> sidebar_nav
    query_layer --> api_client
    auth_hook --> api_client
    api_client -. "REST / JSON" .-> fastapi_api
    theme_provider -. "provides theme" .-> ui_lib
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% ═══════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
fullstack_boundary CONTAINS [fullstack_system, traefik, spa, fastapi_api, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment: FastAPI Backend ─────────────────
fastapi_api CONTAINS [cors_mw, routes, auth_deps, services, data, config]
  routes    CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services  CONTAINS [crud_layer, security_mod, email_utils]
  data      CONTAINS [models_layer, db_engine]

%% ── L2→L3 internal containment: React SPA ──────────────────────
spa CONTAINS [tanstack_router, state_mgmt, api_client, features, ui_framework]
  state_mgmt   CONTAINS [auth_hook, query_layer]
  features     CONTAINS [dashboard_feature, admin_feature, items_feature, settings_feature, auth_pages]
  ui_framework CONTAINS [ui_lib, theme_provider, sidebar_nav]
```
