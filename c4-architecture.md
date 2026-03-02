# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
flowchart TB
    %% SCOPE: urn:c4:system:fullstack_platform

    subgraph users["Users"]
        user["User
        [Person]
        Browses the web dashboard,
        manages own items and profile"]

        admin_user["Administrator
        [Person]
        Manages users and
        system configuration"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        react_spa["React SPA
        [Container: React 19 / Nginx]"]
        fastapi_api["FastAPI Backend
        [Container: Python / FastAPI]"]
        pg["PostgreSQL
        [Container: PostgreSQL 18]"]
        traefik_proxy["Traefik
        [Container: Reverse Proxy]"]
        adminer["Adminer
        [Container: PHP]"]
    end

    subgraph external["External Systems"]
        smtp_server["SMTP Server
        [External System]
        Delivers transactional emails"]

        sentry["Sentry
        [External System]
        Error monitoring & tracing"]
    end

    user -->|"Uses [HTTPS]"| platform_boundary
    admin_user -->|"Administers [HTTPS]"| platform_boundary
    platform_boundary -->|"Sends emails [SMTP/TLS]"| smtp_server
    platform_boundary -->|"Reports errors [HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
flowchart TB
    %% SCOPE: urn:c4:system:fullstack_platform

    user["User
    [Person]"]

    admin_user["Administrator
    [Person]"]

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik_proxy["Traefik
        [Container: Traefik 3.x]
        Reverse proxy, TLS termination,
        subdomain routing"]

        react_spa["React SPA
        [Container: React 19 / Vite / Nginx]
        Single-page application served
        as static files by Nginx"]

        fastapi_api["FastAPI Backend
        [Container: Python 3.10 / FastAPI]
        RESTful API, business logic,
        auth, email"]

        pg["PostgreSQL
        [Container: PostgreSQL 18]
        Stores users, items,
        and application data"]

        adminer["Adminer
        [Container: PHP]
        Database administration UI"]
    end

    subgraph external["External Systems"]
        smtp_server["SMTP Server
        [External System]
        Delivers transactional emails
        (Mailcatcher in dev)"]

        sentry["Sentry
        [External System]
        Error monitoring & tracing"]
    end

    user -->|"Browses
    [HTTPS]"| traefik_proxy

    admin_user -->|"Manages
    [HTTPS]"| traefik_proxy

    traefik_proxy -->|"dashboard.DOMAIN
    [HTTP]"| react_spa

    traefik_proxy -->|"api.DOMAIN
    [HTTP]"| fastapi_api

    traefik_proxy -->|"adminer.DOMAIN
    [HTTP]"| adminer

    react_spa -->|"API calls
    [HTTPS / JSON]"| fastapi_api

    fastapi_api -->|"Reads / writes
    [SQL / psycopg]"| pg

    adminer -->|"Admin queries
    [SQL]"| pg

    fastapi_api -->|"Sends emails
    [SMTP]"| smtp_server

    fastapi_api -->|"Reports errors
    [HTTPS]"| sentry
```

---

## L3: FastAPI Backend

```mermaid
flowchart TB
    %% SCOPE: urn:c4:container:fastapi_api

    react_spa["React SPA
    [Container]"]

    pg["PostgreSQL
    [Container]"]

    smtp_server["SMTP Server
    [External]"]

    sentry["Sentry
    [External]"]

    subgraph fastapi_api["FastAPI Backend"]

        %% KIND: boundary
        subgraph middleware["Middleware"]
            cors_mw["CORS Middleware
            [Component: Starlette]
            Handles cross-origin requests
            from frontend origins"]
        end

        %% KIND: router
        subgraph api_routes["API Routes"]
            login_routes["Login Routes
            [Component: /login/*]
            access-token, test-token,
            password recovery & reset"]

            users_routes["Users Routes
            [Component: /users/*]
            CRUD, signup, profile,
            password change"]

            items_routes["Items Routes
            [Component: /items/*]
            Item CRUD operations"]

            utils_routes["Utils Routes
            [Component: /utils/*]
            Health check, test email"]
        end

        %% KIND: service_layer
        subgraph dependencies["FastAPI Dependencies"]
            auth_deps["Auth Dependencies
            [Component: deps.py]
            OAuth2 bearer token validation,
            current-user resolution,
            superuser guard"]

            db_session_dep["DB Session Provider
            [Component: deps.py]
            Yields SQLModel Session
            per request"]
        end

        %% KIND: service_layer
        subgraph services["Service Layer"]
            crud_layer["CRUD Layer
            [Component: crud.py]
            User & Item create, read,
            update, delete operations"]

            security["Security
            [Component: core/security.py]
            JWT token creation & verification,
            password hashing (Argon2/Bcrypt)"]

            email_utils["Email Utils
            [Component: utils.py]
            Jinja2 email templates,
            password-reset & new-account emails"]
        end

        %% KIND: data_access
        subgraph data_layer["Data Layer"]
            models_layer["SQLModel Models
            [Component: models.py]
            User, Item — ORM models
            and Pydantic schemas"]

            db_engine["DB Engine
            [Component: core/db.py]
            SQLAlchemy engine,
            session factory, init_db"]

            alembic_migrations["Alembic Migrations
            [Component: alembic/]
            Schema versioning
            and upgrade scripts"]
        end

        %% KIND: boundary
        app_config["App Config
        [Component: core/config.py]
        Pydantic Settings —
        env-based configuration"]
    end

    react_spa -->|"HTTP requests"| cors_mw
    cors_mw --> api_routes
    api_routes --> auth_deps
    api_routes --> db_session_dep
    auth_deps --> security
    auth_deps --> db_session_dep
    login_routes --> crud_layer
    login_routes --> email_utils
    users_routes --> crud_layer
    users_routes --> email_utils
    items_routes --> crud_layer
    crud_layer --> models_layer
    crud_layer --> db_session_dep
    db_session_dep --> db_engine
    db_engine -->|"SQL"| pg
    email_utils -->|"SMTP"| smtp_server
    alembic_migrations --> db_engine
    app_config -.->|"configures"| db_engine
    app_config -.->|"configures"| security
    fastapi_api -.->|"reports errors"| sentry
```

---

## L3: React SPA

```mermaid
flowchart TB
    %% SCOPE: urn:c4:container:react_spa

    fastapi_api["FastAPI Backend
    [Container]"]

    subgraph react_spa["React SPA"]

        %% KIND: router
        subgraph routing["Routing"]
            tanstack_router["TanStack Router
            [Component: @tanstack/react-router]
            File-based routing,
            route guards, layout nesting"]
        end

        %% KIND: boundary
        subgraph pages["Pages"]
            login_page["Login Page
            [Component: routes/login.tsx]
            Email & password login form"]

            signup_page["Signup Page
            [Component: routes/signup.tsx]
            New user registration"]

            dashboard_page["Dashboard
            [Component: routes/_layout/index.tsx]
            Main landing page"]

            items_page["Items Page
            [Component: routes/_layout/items.tsx]
            Item listing and CRUD"]

            admin_page["Admin Page
            [Component: routes/_layout/admin.tsx]
            User management (superuser)"]

            settings_page["Settings Page
            [Component: routes/_layout/settings.tsx]
            Profile & password settings"]

            password_pages["Password Recovery / Reset
            [Component: routes/recover-password.tsx,
            routes/reset-password.tsx]"]
        end

        %% KIND: service_layer
        subgraph state_mgmt["State Management"]
            query_client["React Query Client
            [Component: @tanstack/react-query]
            Server-state caching,
            background refetching"]

            auth_hook["useAuth Hook
            [Component: hooks/useAuth.ts]
            Login/signup mutations,
            current-user query,
            token management"]

            theme_context["Theme & Sidebar Contexts
            [Component: Providers]
            Dark/light theme toggle,
            sidebar collapse state"]
        end

        %% KIND: boundary
        subgraph ui_layer["UI Layer"]
            ui_components["UI Primitives
            [Component: Radix UI / shadcn]
            Button, Dialog, Form, Table,
            Select, Tabs, Tooltip, etc."]

            feature_components["Feature Components
            [Component: components/]
            Admin panel, Items table,
            User settings forms,
            Sidebar navigation"]
        end

        %% KIND: integration
        subgraph client_layer["API Client"]
            api_client["OpenAPI Client
            [Component: @hey-api/openapi-ts]
            Auto-generated SDK from
            backend OpenAPI spec —
            ItemsService, UsersService,
            LoginService"]
        end
    end

    tanstack_router --> pages
    pages --> feature_components
    feature_components --> ui_components
    pages --> auth_hook
    pages --> query_client
    auth_hook --> api_client
    query_client --> api_client
    feature_components --> query_client
    api_client -->|"HTTP / JSON"| fastapi_api
    theme_context -.->|"provides context"| pages
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% Every subgraph from every diagram appears as a CONTAINS entry.
%% Indentation shows nesting depth.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik_proxy, react_spa, fastapi_api, pg, adminer]
external           CONTAINS [smtp_server, sentry]

%% ── L2 → L3 internal containment ──────────────────────────────────

%% ── FastAPI Backend (fastapi_api) ──────────────────────────────────
fastapi_api        CONTAINS [middleware, api_routes, dependencies, services, data_layer, app_config]
  middleware       CONTAINS [cors_mw]
  api_routes       CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  dependencies     CONTAINS [auth_deps, db_session_dep]
  services         CONTAINS [crud_layer, security, email_utils]
  data_layer       CONTAINS [models_layer, db_engine, alembic_migrations]

%% ── React SPA (react_spa) ──────────────────────────────────────────
react_spa          CONTAINS [routing, pages, state_mgmt, ui_layer, client_layer]
  routing          CONTAINS [tanstack_router]
  pages            CONTAINS [login_page, signup_page, dashboard_page, items_page, admin_page, settings_page, password_pages]
  state_mgmt       CONTAINS [query_client, auth_hook, theme_context]
  ui_layer         CONTAINS [ui_components, feature_components]
  client_layer     CONTAINS [api_client]
```
