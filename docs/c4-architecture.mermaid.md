# C4 Architecture Model — Full Stack FastAPI Platform

> C4 model: L1 System Context · L2 Container · L3 Component diagrams.
> Generated from source analysis of the full-stack-fastapi-template codebase.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fastapi_platform
flowchart TD

    subgraph users["Users"]
        user["User<br/>‹Person›<br/>Manages personal items<br/>and account settings"]
        admin_user["Administrator<br/>‹Person›<br/>Manages users and<br/>system configuration"]
    end

    subgraph fastapi_platform_boundary["Full Stack FastAPI Platform"]
        fastapi_platform["Full Stack FastAPI Platform<br/>‹Software System›<br/>Web application for item<br/>management, user admin,<br/>and JWT authentication"]
    end

    subgraph external["External Systems"]
        smtp_service["SMTP Provider<br/>‹External System›<br/>Delivers transactional emails<br/>(password reset, new account)"]
        sentry["Sentry<br/>‹External System›<br/>Error tracking and<br/>performance monitoring"]
    end

    user -- "Uses web application [HTTPS]" --> fastapi_platform
    admin_user -- "Administers platform [HTTPS]" --> fastapi_platform
    fastapi_platform -- "Sends emails [SMTP]" --> smtp_service
    fastapi_platform -- "Reports errors [HTTPS]" --> sentry
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fastapi_platform
flowchart TD

    subgraph users["Users"]
        user["User<br/>‹Person›"]
        admin_user["Administrator<br/>‹Person›"]
    end

    subgraph fastapi_platform["Full Stack FastAPI Platform"]
        traefik["Traefik<br/>‹Container: Reverse Proxy›<br/>TLS termination via Let's Encrypt<br/>and host-based request routing"]
        spa["React SPA<br/>‹Container: React 19 / Vite / Nginx›<br/>Single-page application serving<br/>dashboard, items, admin, settings"]
        fastapi_backend["FastAPI Backend<br/>‹Container: Python 3.10 / FastAPI›<br/>REST API providing auth,<br/>user management, and item CRUD"]
        pg[("PostgreSQL<br/>‹Container: PostgreSQL 18›<br/>Stores users, items,<br/>and application data")]
        adminer["Adminer<br/>‹Container: PHP›<br/>Database administration UI"]
    end

    subgraph external["External Systems"]
        smtp_service["SMTP Provider<br/>‹External System›"]
        sentry["Sentry<br/>‹External System›"]
    end

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik
    traefik -- "dashboard.* → static assets" --> spa
    traefik -- "api.* → /api/v1/*" --> fastapi_backend
    traefik -- "adminer.*" --> adminer
    spa -. "REST API /api/v1/* [JSON/HTTPS]<br/>(browser → backend)" .-> fastapi_backend
    fastapi_backend -- "SQL [psycopg / SQLModel]" --> pg
    fastapi_backend -- "Sends emails [SMTP]" --> smtp_service
    fastapi_backend -- "Reports errors [HTTPS SDK]" --> sentry
    adminer -- "SQL" --> pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TD

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph mw["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/>‹Component›<br/>Cross-origin request<br/>header management"]
        end

        subgraph routes["API Routes /api/v1"]
            %% KIND: router
            login_routes["Login Routes<br/>‹Component: /login›<br/>OAuth2 token exchange,<br/>password recovery & reset"]
            %% KIND: router
            users_routes["Users Routes<br/>‹Component: /users›<br/>User CRUD, signup,<br/>profile management"]
            %% KIND: router
            items_routes["Items Routes<br/>‹Component: /items›<br/>Item CRUD operations<br/>(owner-scoped)"]
            %% KIND: router
            utils_routes["Utils Routes<br/>‹Component: /utils›<br/>Health check,<br/>test email endpoint"]
            %% KIND: router
            private_routes["Private Routes<br/>‹Component: /private›<br/>Local-env admin<br/>user creation"]
        end

        subgraph deps["Request Dependencies"]
            %% KIND: boundary
            auth_dep["Auth Dependency<br/>‹Component›<br/>JWT validation and<br/>current-user resolution"]
            %% KIND: data_access
            session_dep["DB Session Provider<br/>‹Component›<br/>Yields SQLModel Session<br/>per request via get_db"]
        end

        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Functions<br/>‹Component›<br/>create/read/update/delete<br/>for Users and Items"]
            %% KIND: integration
            email_utils["Email Utilities<br/>‹Component›<br/>Jinja2 template rendering<br/>and SMTP dispatch"]
        end

        subgraph data["Data Access Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models<br/>‹Component›<br/>User, Item table models<br/>and Pydantic DTOs"]
            %% KIND: storage
            db_engine["Database Engine<br/>‹Component›<br/>SQLAlchemy engine,<br/>session factory, init_db"]
            %% KIND: data_access
            alembic_migrations["Alembic Migrations<br/>‹Component›<br/>Schema versioning and<br/>migration runner"]
        end

        subgraph core["Core"]
            %% KIND: boundary
            app_config["Settings<br/>‹Component›<br/>Pydantic Settings with<br/>env-var binding"]
            %% KIND: service_layer
            jwt_security["Security<br/>‹Component›<br/>JWT HS256 token creation,<br/>Argon2 / bcrypt hashing"]
        end

    end

    spa["React SPA"]
    pg[("PostgreSQL")]
    smtp_service["SMTP Provider"]
    sentry["Sentry"]

    spa --> cors_mw
    cors_mw --> login_routes
    cors_mw --> users_routes
    cors_mw --> items_routes
    cors_mw --> utils_routes
    cors_mw --> private_routes

    users_routes --> auth_dep
    items_routes --> auth_dep
    utils_routes --> auth_dep
    auth_dep --> jwt_security
    auth_dep --> session_dep
    session_dep --> db_engine

    login_routes --> crud_layer
    login_routes --> jwt_security
    login_routes --> email_utils
    users_routes --> crud_layer
    items_routes --> crud_layer
    utils_routes --> email_utils
    private_routes --> crud_layer

    crud_layer --> models_layer
    crud_layer --> session_dep
    models_layer --> db_engine
    db_engine --> pg

    email_utils --> smtp_service
    alembic_migrations --> db_engine
    app_config -. "configures" .-> jwt_security
    app_config -. "configures" .-> db_engine
    fastapi_backend -. "reports errors" .-> sentry
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TD

    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            ts_router["TanStack Router<br/>‹Component›<br/>File-based routing with<br/>auto code splitting"]
            %% KIND: boundary
            route_guards["Route Guards<br/>‹Component›<br/>beforeLoad auth checks<br/>and role verification"]
        end

        subgraph state["State Management"]
            %% KIND: service_layer
            auth_hook["Auth Hook — useAuth<br/>‹Component›<br/>Login, signup, current-user<br/>state via TanStack Query"]
            %% KIND: service_layer
            theme_ctx["Theme Provider<br/>‹Component›<br/>Light / dark / system theme<br/>with localStorage persistence"]
            %% KIND: data_access
            query_client["TanStack Query Client<br/>‹Component›<br/>Server-state caching,<br/>refetching & invalidation"]
        end

        subgraph api_layer["API Integration"]
            %% KIND: integration
            api_client["Generated API Client<br/>‹Component›<br/>OpenAPI-generated SDK with<br/>Axios request layer"]
            %% KIND: integration
            openapi_config["OpenAPI Config<br/>‹Component›<br/>Base URL and JWT token<br/>injection from localStorage"]
        end

        subgraph features["Feature Modules"]
            %% KIND: service_layer
            dashboard_feat["Dashboard<br/>‹Component›<br/>Overview page with<br/>current-user info"]
            %% KIND: service_layer
            items_feat["Items Management<br/>‹Component›<br/>CRUD UI for items<br/>with DataTable"]
            %% KIND: service_layer
            admin_feat["Admin Panel<br/>‹Component›<br/>User management UI<br/>(superuser only)"]
            %% KIND: service_layer
            settings_feat["User Settings<br/>‹Component›<br/>Profile, password change,<br/>account deletion"]
            %% KIND: service_layer
            auth_feat["Auth Pages<br/>‹Component›<br/>Login, signup, password<br/>recovery & reset forms"]
        end

        subgraph ui_layer["UI Layer"]
            %% KIND: boundary
            common_ui["Common Components<br/>‹Component›<br/>Sidebar, layout, logo,<br/>footer, error pages"]
            %% KIND: boundary
            radix_primitives["Radix UI Primitives<br/>‹Component›<br/>Accessible component<br/>library via shadcn/ui"]
            %% KIND: boundary
            data_table["DataTable<br/>‹Component›<br/>TanStack Table with<br/>pagination & sorting"]
        end

    end

    fastapi_backend["FastAPI Backend"]
    user["User"]
    admin_user["Administrator"]

    user --> ts_router
    admin_user --> ts_router

    ts_router --> route_guards
    route_guards --> auth_hook
    route_guards --> dashboard_feat
    route_guards --> items_feat
    route_guards --> admin_feat
    route_guards --> settings_feat
    ts_router --> auth_feat

    auth_feat --> auth_hook
    dashboard_feat --> query_client
    items_feat --> query_client
    admin_feat --> query_client
    settings_feat --> query_client
    auth_hook --> api_client
    query_client --> api_client
    api_client --> openapi_config
    api_client --> fastapi_backend

    items_feat --> data_table
    admin_feat --> data_table
    items_feat --> common_ui
    admin_feat --> common_ui
    settings_feat --> common_ui
    auth_feat --> common_ui
    dashboard_feat --> common_ui
    data_table --> radix_primitives
    common_ui --> radix_primitives
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% Covers L1, L2, and all L3 diagrams.
%% Every subgraph from every diagram appears as a CONTAINS entry.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users                     CONTAINS [user, admin_user]
fastapi_platform_boundary CONTAINS [fastapi_platform]
external                  CONTAINS [smtp_service, sentry]

%% ── L2 internal containment ─────────────────────────────────────
fastapi_platform CONTAINS [traefik, spa, fastapi_backend, pg, adminer]

%% ── L3: FastAPI Backend ─────────────────────────────────────────
fastapi_backend CONTAINS [mw, routes, deps, services, data, core]
  mw            CONTAINS [cors_mw]
  routes        CONTAINS [login_routes, users_routes, items_routes, utils_routes, private_routes]
  deps          CONTAINS [auth_dep, session_dep]
  services      CONTAINS [crud_layer, email_utils]
  data          CONTAINS [models_layer, db_engine, alembic_migrations]
  core          CONTAINS [app_config, jwt_security]

%% ── L3: React SPA ──────────────────────────────────────────────
spa CONTAINS [routing, state, api_layer, features, ui_layer]
  routing       CONTAINS [ts_router, route_guards]
  state         CONTAINS [auth_hook, theme_ctx, query_client]
  api_layer     CONTAINS [api_client, openapi_config]
  features      CONTAINS [dashboard_feat, items_feat, admin_feat, settings_feat, auth_feat]
  ui_layer      CONTAINS [common_ui, radix_primitives, data_table]
```
