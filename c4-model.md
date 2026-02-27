# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:platform
graph TB
    subgraph users_group["Users"]
        user["End User\n<i>Person</i>\nBrowses the dashboard,\nmanages items"]
        admin_user["Admin User\n<i>Person</i>\nManages users,\nconfigures system"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform\n<i>Software System</i>\nWeb application for managing\nusers and items"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Service\n<i>External System</i>\nSends transactional emails"]
        sentry["Sentry\n<i>External System</i>\nError tracking and monitoring"]
    end

    user -->|"Uses web application"| platform
    admin_user -->|"Administers users and data"| platform
    platform -->|"Sends emails via SMTP"| smtp
    platform -->|"Reports errors to"| sentry
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:platform
graph TB
    subgraph users_group["Users"]
        user["End User\n<i>Person</i>"]
        admin_user["Admin User\n<i>Person</i>"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik\n<i>Reverse Proxy: Traefik 3.x</i>\nRoutes HTTPS traffic,\nterminates TLS via Let's Encrypt"]
        spa["React SPA\n<i>Container: React 19 / TypeScript / Nginx</i>\nSingle-page application served\nby Nginx in production"]
        fastapi_backend["FastAPI Backend\n<i>Container: Python / FastAPI</i>\nREST API with JWT auth,\nserves /api/v1 endpoints"]
        pg["PostgreSQL\n<i>Database: PostgreSQL 18</i>\nStores users, items,\nand application data"]
        adminer["Adminer\n<i>Container: Adminer</i>\nWeb-based database\nadministration UI"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Service\n<i>External System</i>"]
        sentry["Sentry\n<i>External System</i>"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"dashboard.* → port 80"| spa
    traefik -->|"api.* → port 8000"| fastapi_backend
    traefik -->|"adminer.* → port 8080"| adminer
    spa -->|"REST API /api/v1/*"| fastapi_backend
    fastapi_backend -->|"SQL via psycopg"| pg
    adminer -->|"SQL"| pg
    fastapi_backend -->|"SMTP"| smtp
    fastapi_backend -->|"Sentry SDK"| sentry
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
graph TB
    spa["React SPA"]
    pg["PostgreSQL"]
    smtp["SMTP Service"]
    sentry["Sentry"]

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph middleware["Middleware & Dependencies"]
            %% KIND: boundary
            cors_mw["CORS Middleware\n<i>Starlette CORSMiddleware</i>\nEnforces allowed origins,\nmethods, headers"]
            %% KIND: service_layer
            auth_deps["Auth Dependencies\n<i>OAuth2 + JWT</i>\nExtracts bearer token,\nvalidates JWT, resolves user"]
            %% KIND: service_layer
            superuser_dep["Superuser Guard\n<i>Dependency</i>\nEnforces is_superuser flag"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes\n<i>/api/v1/login/*</i>\nOAuth2 password flow,\ntoken issuance, password recovery"]
            %% KIND: router
            user_routes["User Routes\n<i>/api/v1/users/*</i>\nUser CRUD, signup,\nprofile management"]
            %% KIND: router
            item_routes["Item Routes\n<i>/api/v1/items/*</i>\nItem CRUD, owner-scoped\nquery filtering"]
            %% KIND: router
            util_routes["Utility Routes\n<i>/api/v1/utils/*</i>\nHealth check,\ntest email endpoint"]
        end

        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Module\n<i>crud.py</i>\nCreate/read/update/delete\noperations for User & Item"]
            %% KIND: service_layer
            security["Security Service\n<i>core/security.py</i>\nJWT creation, Argon2/Bcrypt\npassword hashing & verification"]
            %% KIND: integration
            email_utils["Email Service\n<i>utils.py</i>\nSMTP sending, Jinja2 templates\nfor reset & welcome emails"]
            %% KIND: service_layer
            config["Configuration\n<i>core/config.py</i>\nPydantic Settings with\nenv-based configuration"]
        end

        subgraph data["Data Access Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models\n<i>models.py</i>\nUser & Item table models,\nPydantic schemas"]
            %% KIND: data_access
            db_session["DB Session\n<i>core/db.py</i>\nSQLModel engine creation,\nsession management"]
            %% KIND: storage
            alembic["Alembic Migrations\n<i>alembic/</i>\nSchema versioning,\nupgrade/downgrade scripts"]
        end

    end

    spa -->|"REST API calls"| cors_mw
    cors_mw --> routes
    routes --> auth_deps
    auth_deps --> superuser_dep
    login_routes --> security
    login_routes --> crud_layer
    login_routes --> email_utils
    user_routes --> crud_layer
    user_routes --> security
    item_routes --> crud_layer
    util_routes --> email_utils
    crud_layer --> models_layer
    crud_layer --> db_session
    security --> config
    email_utils --> config
    db_session -->|"SQL via psycopg"| pg
    alembic --> models_layer
    alembic -->|"DDL migrations"| pg
    email_utils -->|"SMTP"| smtp
    fastapi_backend -->|"Sentry SDK"| sentry
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB
    user["End User"]
    admin_user["Admin User"]
    fastapi_backend["FastAPI Backend"]

    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            tanstack_router["TanStack Router\n<i>File-based routing</i>\nAuto-generated route tree\nfrom filesystem convention"]
            %% KIND: boundary
            layout_guard["Layout Guard\n<i>beforeLoad hook</i>\nRedirects unauthenticated\nusers to /login"]
        end

        subgraph features["Features"]
            %% KIND: service_layer
            dashboard_feature["Dashboard\n<i>/_layout/index</i>\nWelcome page with\nuser greeting"]
            %% KIND: service_layer
            items_feature["Items Management\n<i>/_layout/items</i>\nCRUD table for items\nwith owner filtering"]
            %% KIND: service_layer
            admin_feature["Admin Panel\n<i>/_layout/admin</i>\nUser management,\nsuperuser-only access"]
            %% KIND: service_layer
            settings_feature["User Settings\n<i>/_layout/settings</i>\nProfile edit, password change,\naccount deletion"]
            %% KIND: service_layer
            auth_pages["Auth Pages\n<i>/login, /signup, /recover-password</i>\nLogin, registration,\npassword recovery flows"]
        end

        subgraph services_layer["Services"]
            %% KIND: integration
            api_client["API Client\n<i>@hey-api/openapi-ts</i>\nAuto-generated from OpenAPI,\nAxios-based HTTP calls"]
            %% KIND: service_layer
            auth_hook["Auth Hook\n<i>useAuth.ts</i>\nLogin/logout/signup mutations,\ncurrent user query"]
            %% KIND: service_layer
            query_client["Query Client\n<i>TanStack Query</i>\nServer-state cache,\nauto-refetch, error handling"]
        end

        subgraph ui_layer["UI Components"]
            %% KIND: service_layer
            ui_lib["Radix UI + shadcn\n<i>Component library</i>\nDialog, Dropdown, Sidebar,\nForm, DataTable primitives"]
            %% KIND: service_layer
            common_components["Common Components\n<i>components/Common/</i>\nLogo, Footer, ErrorComponent,\nNotFound, AuthLayout"]
        end

    end

    user -->|"Interacts via browser"| tanstack_router
    admin_user -->|"Interacts via browser"| tanstack_router
    tanstack_router --> layout_guard
    layout_guard --> features
    auth_pages --> auth_hook
    dashboard_feature --> query_client
    items_feature --> api_client
    items_feature --> query_client
    admin_feature --> api_client
    admin_feature --> query_client
    settings_feature --> api_client
    settings_feature --> query_client
    features --> ui_lib
    features --> common_components
    auth_hook --> api_client
    auth_hook --> query_client
    api_client -->|"REST API /api/v1/*"| fastapi_backend
```

---

## L3: Traefik

```mermaid
%% SCOPE: urn:c4:container:traefik
graph TB
    user["End User"]
    admin_user["Admin User"]
    spa["React SPA"]
    fastapi_backend["FastAPI Backend"]
    adminer["Adminer"]

    subgraph traefik["Traefik"]

        %% KIND: boundary
        entrypoints["Entrypoints\n<i>HTTP :80 / HTTPS :443</i>\nAccepts inbound traffic"]
        %% KIND: service_layer
        tls_termination["TLS Termination\n<i>Let's Encrypt certresolver</i>\nAutomatic certificate management"]
        %% KIND: router
        http_redirect["HTTP→HTTPS Redirect\n<i>Middleware</i>\nForces secure connections"]
        %% KIND: router
        routers["Domain Routers\n<i>Label-based config</i>\nHost rules: dashboard.*, api.*, adminer.*"]

    end

    user -->|"HTTPS"| entrypoints
    admin_user -->|"HTTPS"| entrypoints
    entrypoints --> tls_termination
    entrypoints --> http_redirect
    http_redirect --> tls_termination
    tls_termination --> routers
    routers -->|"dashboard.*"| spa
    routers -->|"api.*"| fastapi_backend
    routers -->|"adminer.*"| adminer
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% Every subgraph from every diagram appears as a CONTAINS entry.
%% Indentation reflects nesting depth.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users_group        CONTAINS [user, admin_user]
platform_boundary  CONTAINS [platform, traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2 container-level (same boundary, expanded) ────────────────
%%   platform_boundary already lists all containers above.
%%   L2 refines platform → {traefik, spa, fastapi_backend, pg, adminer}

%% ── L3: fastapi_backend ─────────────────────────────────────────
fastapi_backend    CONTAINS [middleware, routes, services, data]
  middleware       CONTAINS [cors_mw, auth_deps, superuser_dep]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services         CONTAINS [crud_layer, security, email_utils, config]
  data             CONTAINS [models_layer, db_session, alembic]

%% ── L3: spa ─────────────────────────────────────────────────────
spa                CONTAINS [routing, features, services_layer, ui_layer]
  routing          CONTAINS [tanstack_router, layout_guard]
  features         CONTAINS [dashboard_feature, items_feature, admin_feature, settings_feature, auth_pages]
  services_layer   CONTAINS [api_client, auth_hook, query_client]
  ui_layer         CONTAINS [ui_lib, common_components]

%% ── L3: traefik ─────────────────────────────────────────────────
traefik            CONTAINS [entrypoints, tls_termination, http_redirect, routers]
```
