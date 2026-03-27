# C4 Architecture – Full Stack FastAPI Project

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
flowchart TB
    subgraph users["Users"]
        user["End User<br/><i>Browser-based application user</i>"]
        admin_user["Administrator<br/><i>Superuser with elevated privileges</i>"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform<br/><i>Web application for managing items<br/>with user authentication &amp; admin</i>"]
    end

    subgraph external["External Services"]
        smtp_server["SMTP Email Service<br/><i>Sends transactional emails<br/>(password reset, notifications)</i>"]
        sentry["Sentry<br/><i>Error tracking &amp; performance monitoring</i>"]
    end

    user -->|"Uses web application"| platform
    admin_user -->|"Manages users &amp; system"| platform
    platform -->|"Sends emails via SMTP"| smtp_server
    platform -->|"Reports errors &amp; traces"| sentry
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
flowchart TB
    subgraph users["Users"]
        user["End User"]
        admin_user["Administrator"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik<br/><i>Reverse Proxy &amp; TLS termination<br/>(Let's Encrypt ACME)</i>"]
        spa["React SPA<br/><i>Vite + React 19 + TanStack Router<br/>Served by Nginx in production</i>"]
        fastapi_backend["FastAPI Backend<br/><i>Python REST API — Uvicorn<br/>OpenAPI / JSON over HTTP</i>"]
        pg["PostgreSQL 18<br/><i>Relational Database<br/>(Users, Items)</i>"]
        adminer["Adminer<br/><i>Database Admin UI</i>"]
    end

    subgraph external["External Services"]
        smtp_server["SMTP Email Service"]
        sentry["Sentry"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"dashboard.* → port 80"| spa
    traefik -->|"api.* → port 8000"| fastapi_backend
    traefik -->|"adminer.* → port 8080"| adminer
    spa -->|"REST / JSON<br/>/api/v1/*"| fastapi_backend
    fastapi_backend -->|"SQL via psycopg"| pg
    adminer -->|"SQL"| pg
    fastapi_backend -->|"SMTP"| smtp_server
    fastapi_backend -->|"Sentry SDK"| sentry
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TB
    subgraph fastapi_backend["FastAPI Backend"]

        subgraph api_layer["API Layer"]
            %% KIND: router
            api_router["API Router<br/><i>api/main.py — mounts all<br/>route modules under /api/v1</i>"]
            %% KIND: router
            auth_routes["Login Routes<br/><i>routes/login.py — token issuance,<br/>password recovery &amp; reset</i>"]
            %% KIND: router
            user_routes["User Routes<br/><i>routes/users.py — CRUD users,<br/>profile, signup, admin ops</i>"]
            %% KIND: router
            item_routes["Item Routes<br/><i>routes/items.py — CRUD items<br/>(owner-scoped)</i>"]
            %% KIND: router
            util_routes["Utility Routes<br/><i>routes/utils.py — health check,<br/>test email (superuser)</i>"]
        end

        subgraph middleware_layer["Request Pipeline"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware<br/>configured from BACKEND_CORS_ORIGINS</i>"]
            %% KIND: service_layer
            auth_deps["Auth Dependencies<br/><i>deps.py — OAuth2PasswordBearer,<br/>get_current_user, superuser guard,<br/>SessionDep</i>"]
        end

        subgraph service_layer["Business Logic"]
            %% KIND: data_access
            crud_layer["CRUD Layer<br/><i>crud.py — create_user, update_user,<br/>get_user_by_email, authenticate</i>"]
            %% KIND: service_layer
            email_utils["Email Utilities<br/><i>utils.py — send_email,<br/>Jinja2 HTML templates,<br/>password-reset JWT helpers</i>"]
            %% KIND: service_layer
            security_mod["Security Module<br/><i>core/security.py — JWT creation,<br/>Argon2/Bcrypt password hashing</i>"]
        end

        subgraph data_layer["Data Layer"]
            %% KIND: storage
            models_layer["Models &amp; Schemas<br/><i>models.py — User &amp; Item tables,<br/>Pydantic request/response types</i>"]
            %% KIND: data_access
            db_mod["Database Module<br/><i>core/db.py — SQLAlchemy engine,<br/>session factory, init_db seeder</i>"]
            %% KIND: storage
            config_mod["Configuration<br/><i>core/config.py — Pydantic Settings,<br/>env vars, SQLALCHEMY_DATABASE_URI</i>"]
            %% KIND: storage
            alembic_migrations["Alembic Migrations<br/><i>alembic/ — schema version control,<br/>run on prestart</i>"]
        end
    end

    pg["PostgreSQL 18"]
    smtp_server["SMTP Email Service"]
    sentry["Sentry"]

    api_router --> auth_routes
    api_router --> user_routes
    api_router --> item_routes
    api_router --> util_routes

    cors_mw -.->|"wraps"| api_router

    auth_routes --> auth_deps
    user_routes --> auth_deps
    item_routes --> auth_deps

    auth_routes --> security_mod
    auth_routes --> crud_layer
    auth_routes --> email_utils
    user_routes --> crud_layer
    user_routes --> security_mod
    item_routes --> crud_layer
    util_routes --> email_utils

    auth_deps --> security_mod
    auth_deps --> db_mod
    auth_deps --> models_layer

    crud_layer --> models_layer
    crud_layer --> db_mod
    email_utils --> config_mod

    db_mod --> config_mod
    db_mod --> pg
    alembic_migrations --> pg

    fastapi_backend -.->|"Sentry SDK init"| sentry
    email_utils -->|"SMTP"| smtp_server
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TB
    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            router["TanStack Router<br/><i>File-based routing,<br/>code-generated route tree</i>"]
            %% KIND: boundary
            layout_shell["Layout Shell<br/><i>_layout.tsx — sidebar chrome,<br/>auth-redirect beforeLoad guard</i>"]
        end

        subgraph features["Feature Pages"]
            %% KIND: boundary
            dashboard_page["Dashboard<br/><i>/ — authenticated landing page</i>"]
            %% KIND: boundary
            items_feature["Items Feature<br/><i>/items — data table with<br/>add, edit, delete dialogs</i>"]
            %% KIND: boundary
            admin_feature["Admin Feature<br/><i>/admin — user management<br/>table (superuser-only gate)</i>"]
            %% KIND: boundary
            settings_feature["User Settings<br/><i>/settings — profile info,<br/>password change, delete account</i>"]
            %% KIND: boundary
            auth_pages["Auth Pages<br/><i>/login, /signup,<br/>/recover-password, /reset-password</i>"]
        end

        subgraph state_layer["State &amp; Services"]
            %% KIND: service_layer
            auth_hook["useAuth Hook<br/><i>Login/logout mutations,<br/>current-user query,<br/>localStorage token management</i>"]
            %% KIND: integration
            api_client["OpenAPI Client<br/><i>Generated axios SDK via<br/>@hey-api/openapi-ts — typed services:<br/>LoginService, UsersService,<br/>ItemsService, UtilsService</i>"]
            %% KIND: service_layer
            query_client["TanStack Query<br/><i>Server state cache &amp; mutations,<br/>global 401/403 error handler</i>"]
        end

        subgraph ui_layer["UI Foundation"]
            %% KIND: boundary
            ui_components["UI Component Library<br/><i>shadcn/Radix primitives: Button,<br/>Dialog, Table, Form, Sidebar…</i>"]
            %% KIND: service_layer
            theme_provider["Theme Provider<br/><i>Light / dark mode React context</i>"]
        end
    end

    fastapi_backend["FastAPI Backend"]

    router --> layout_shell
    layout_shell --> dashboard_page
    layout_shell --> items_feature
    layout_shell --> admin_feature
    layout_shell --> settings_feature
    router --> auth_pages

    auth_pages --> auth_hook
    dashboard_page --> query_client
    items_feature --> query_client
    admin_feature --> query_client
    settings_feature --> query_client

    auth_hook --> api_client
    query_client --> api_client

    items_feature --> ui_components
    admin_feature --> ui_components
    settings_feature --> ui_components
    auth_pages --> ui_components
    dashboard_page --> ui_components

    api_client -->|"REST / JSON /api/v1/*"| fastapi_backend
```

---

## CONTAINMENT MAP

```
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers ALL levels (L1 → L2 → L3)
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp_server, sentry]

%% ── L2 → L3 internal containment: FastAPI Backend ───────────────
fastapi_backend    CONTAINS [api_layer, middleware_layer, service_layer, data_layer]
  api_layer        CONTAINS [api_router, auth_routes, user_routes, item_routes, util_routes]
  middleware_layer  CONTAINS [cors_mw, auth_deps]
  service_layer    CONTAINS [crud_layer, email_utils, security_mod]
  data_layer       CONTAINS [models_layer, db_mod, config_mod, alembic_migrations]

%% ── L2 → L3 internal containment: React SPA ────────────────────
spa                CONTAINS [routing, features, state_layer, ui_layer]
  routing          CONTAINS [router, layout_shell]
  features         CONTAINS [dashboard_page, items_feature, admin_feature, settings_feature, auth_pages]
  state_layer      CONTAINS [auth_hook, api_client, query_client]
  ui_layer         CONTAINS [ui_components, theme_provider]
```
