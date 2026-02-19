# C4 Architecture Model — Full Stack FastAPI Platform

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:platform
flowchart TD

    subgraph users["Users"]
        user["👤 User\n&lt;i&gt;Manages items and account settings&lt;/i&gt;"]
        admin_user["👤 Admin\n&lt;i&gt;Manages users and system config&lt;/i&gt;"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["🖥️ Full Stack FastAPI Platform\n&lt;i&gt;Web application for user and item management\nFastAPI · React · PostgreSQL&lt;/i&gt;"]
    end

    subgraph external["External Systems"]
        smtp["📧 SMTP Server\n&lt;i&gt;Transactional email delivery&lt;/i&gt;"]
        sentry["📊 Sentry\n&lt;i&gt;Error monitoring and tracing&lt;/i&gt;"]
    end

    user -->|"Uses\nHTTPS"| platform
    admin_user -->|"Administers\nHTTPS"| platform
    platform -->|"Sends emails\nSMTP/TLS"| smtp
    platform -->|"Reports errors\nHTTPS"| sentry
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:platform
flowchart TD

    subgraph users["Users"]
        user["👤 User"]
        admin_user["👤 Admin"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["🔀 Traefik\n&lt;i&gt;Reverse Proxy / Load Balancer&lt;/i&gt;\n[Docker: traefik]"]
        spa["⚛️ React SPA\n&lt;i&gt;Single Page Application&lt;/i&gt;\n[Docker: Nginx · React/Vite/TypeScript]"]
        fastapi_backend["🐍 FastAPI Backend\n&lt;i&gt;REST API Server&lt;/i&gt;\n[Docker: Python · FastAPI · Uvicorn]"]
        pg["🐘 PostgreSQL\n&lt;i&gt;Relational Database&lt;/i&gt;\n[Docker: postgres:18]"]
        adminer["🔧 Adminer\n&lt;i&gt;Database Admin UI&lt;/i&gt;\n[Docker: adminer]"]
    end

    subgraph external["External Systems"]
        smtp["📧 SMTP Server"]
        sentry["📊 Sentry"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"HTTP :80"| spa
    traefik -->|"HTTP :8000"| fastapi_backend
    traefik -->|"HTTP :8080"| adminer
    spa -->|"REST API\n/api/v1/*"| fastapi_backend
    fastapi_backend -->|"SQL\npsycopg"| pg
    adminer -->|"SQL"| pg
    fastapi_backend -->|"SMTP/TLS"| smtp
    fastapi_backend -->|"HTTPS"| sentry
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TD

    spa["⚛️ React SPA"]
    pg["🐘 PostgreSQL"]
    smtp["📧 SMTP Server"]
    sentry["📊 Sentry"]

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph middleware["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware\n&lt;i&gt;Cross-origin request handling&lt;/i&gt;"]
            %% KIND: boundary
            auth_deps["Auth Dependencies\n&lt;i&gt;JWT validation · OAuth2 · Session DI&lt;/i&gt;"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes\n&lt;i&gt;/login/* · /password-recovery/*\n/reset-password/&lt;/i&gt;"]
            %% KIND: router
            user_routes["User Routes\n&lt;i&gt;/users/*&lt;/i&gt;"]
            %% KIND: router
            item_routes["Item Routes\n&lt;i&gt;/items/*&lt;/i&gt;"]
            %% KIND: router
            utils_routes["Utils Routes\n&lt;i&gt;/utils/* · health-check&lt;/i&gt;"]
        end

        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Operations\n&lt;i&gt;User & Item data operations&lt;/i&gt;"]
            %% KIND: integration
            email_utils["Email Utilities\n&lt;i&gt;Jinja2 templates · SMTP sending&lt;/i&gt;"]
        end

        subgraph core["Core"]
            %% KIND: service_layer
            security_mod["Security Module\n&lt;i&gt;JWT tokens · Argon2/Bcrypt hashing&lt;/i&gt;"]
            %% KIND: service_layer
            config_mod["Configuration\n&lt;i&gt;Pydantic Settings · env vars&lt;/i&gt;"]
        end

        subgraph data_layer["Data Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models\n&lt;i&gt;User · Item · Token schemas&lt;/i&gt;"]
            %% KIND: storage
            db_engine["DB Engine\n&lt;i&gt;SQLAlchemy engine & session mgmt&lt;/i&gt;"]
            %% KIND: data_access
            alembic["Alembic Migrations\n&lt;i&gt;Schema version control&lt;/i&gt;"]
        end

    end

    spa -->|"REST /api/v1/*"| cors_mw
    cors_mw --> auth_deps
    auth_deps --> login_routes
    auth_deps --> user_routes
    auth_deps --> item_routes
    auth_deps --> utils_routes

    login_routes --> crud_layer
    login_routes --> security_mod
    login_routes --> email_utils
    user_routes --> crud_layer
    user_routes --> security_mod
    item_routes --> crud_layer
    utils_routes --> email_utils

    crud_layer --> models_layer
    crud_layer --> db_engine
    email_utils --> smtp

    security_mod --> config_mod
    db_engine --> pg
    alembic --> pg

    fastapi_backend -.->|"Reports errors"| sentry
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TD

    user["👤 User"]
    admin_user["👤 Admin"]
    fastapi_backend["🐍 FastAPI Backend"]

    subgraph spa["React SPA"]

        subgraph routing["Routing Layer"]
            %% KIND: router
            router["TanStack Router\n&lt;i&gt;File-based routing · route guards&lt;/i&gt;"]
            %% KIND: service_layer
            auth_hooks["Auth Hooks\n&lt;i&gt;useAuth · login/logout/signup&lt;/i&gt;"]
        end

        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            query_client["TanStack Query\n&lt;i&gt;Server state caching & sync&lt;/i&gt;"]
        end

        subgraph api_layer["API Layer"]
            %% KIND: integration
            api_client["OpenAPI Client SDK\n&lt;i&gt;Auto-generated from backend schema\nItemsService · UsersService · LoginService&lt;/i&gt;"]
        end

        subgraph features["Feature Modules"]
            %% KIND: boundary
            admin_views["Admin Views\n&lt;i&gt;User management CRUD&lt;/i&gt;"]
            %% KIND: boundary
            item_views["Item Views\n&lt;i&gt;Item management CRUD&lt;/i&gt;"]
            %% KIND: boundary
            settings_views["User Settings\n&lt;i&gt;Profile · Password · Account&lt;/i&gt;"]
        end

        subgraph ui_framework["UI Framework"]
            %% KIND: boundary
            layout_components["Layout Components\n&lt;i&gt;Sidebar · Footer · AuthLayout&lt;/i&gt;"]
            %% KIND: boundary
            ui_lib["shadcn/ui Components\n&lt;i&gt;Buttons · Dialogs · Tables · Forms&lt;/i&gt;"]
            %% KIND: service_layer
            theme_provider["Theme Provider\n&lt;i&gt;Dark / Light mode toggle&lt;/i&gt;"]
        end

    end

    user -->|"HTTPS"| router
    admin_user -->|"HTTPS"| router

    router --> auth_hooks
    router --> admin_views
    router --> item_views
    router --> settings_views

    auth_hooks --> query_client
    admin_views --> query_client
    item_views --> query_client
    settings_views --> query_client

    admin_views --> layout_components
    item_views --> layout_components
    settings_views --> layout_components
    admin_views --> ui_lib
    item_views --> ui_lib
    settings_views --> ui_lib

    query_client --> api_client
    api_client -->|"REST /api/v1/*"| fastapi_backend

    layout_components --> theme_provider
```

---

## Containment Map

```
%% ═══════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ═══════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3: FastAPI Backend ──────────────────────────────────────
fastapi_backend    CONTAINS [middleware, routes, services, core, data_layer]
  middleware       CONTAINS [cors_mw, auth_deps]
  routes           CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services         CONTAINS [crud_layer, email_utils]
  core             CONTAINS [security_mod, config_mod]
  data_layer       CONTAINS [models_layer, db_engine, alembic]

%% ── L2→L3: React SPA ───────────────────────────────────────────
spa                CONTAINS [routing, state_mgmt, api_layer, features, ui_framework]
  routing          CONTAINS [router, auth_hooks]
  state_mgmt       CONTAINS [query_client]
  api_layer        CONTAINS [api_client]
  features         CONTAINS [admin_views, item_views, settings_views]
  ui_framework     CONTAINS [layout_components, ui_lib, theme_provider]
```
