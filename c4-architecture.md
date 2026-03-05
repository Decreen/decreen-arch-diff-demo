# C4 Architecture Model – Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
graph TD
    %% L1: System Context – Full Stack FastAPI Platform

    subgraph users["Users"]
        user["User<br/><i>Registered user who<br/>manages items</i>"]
        admin_user["Administrator<br/><i>Superuser who manages<br/>users and the system</i>"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform<br/><i>Web application for item management<br/>with user authentication</i>"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Provider<br/><i>Email delivery service<br/>(Mailgun, SendGrid, etc.)</i>"]
        sentry["Sentry<br/><i>Error and performance<br/>monitoring</i>"]
    end

    user -- "Uses | HTTPS" --> platform
    admin_user -- "Administers | HTTPS" --> platform
    platform -- "Sends emails | SMTP/TLS" --> smtp
    platform -- "Reports errors | HTTPS" --> sentry
```

---

## L2: Container

```mermaid
graph TD
    %% L2: Container – Full Stack FastAPI Platform

    subgraph users["Users"]
        user["User"]
        admin_user["Administrator"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik<br/><i>Reverse proxy, TLS termination<br/>and routing · Traefik 3.6</i>"]
        spa["React Frontend<br/><i>Single-page application<br/>React 19 · Vite 7 · Nginx</i>"]
        fastapi_api["FastAPI Backend<br/><i>REST API server<br/>Python · FastAPI · 4 workers</i>"]
        pg["PostgreSQL<br/><i>Relational database<br/>PostgreSQL 18</i>"]
        adminer["Adminer<br/><i>Database administration UI</i>"]
        prestart["Prestart<br/><i>One-shot migration runner<br/>Alembic · seed data</i>"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Provider"]
        sentry["Sentry"]
    end

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik
    traefik -- "dashboard.*" --> spa
    traefik -- "api.*" --> fastapi_api
    traefik -- "adminer.*" --> adminer
    spa -- "API calls | JSON / HTTPS" --> fastapi_api
    fastapi_api -- "SQL | psycopg" --> pg
    adminer -- "SQL" --> pg
    prestart -- "Alembic migrate + seed" --> pg
    fastapi_api -- "SMTP/TLS" --> smtp
    fastapi_api -- "HTTPS" --> sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TD
    %% SCOPE: urn:c4:container:fastapi_api
    %% L3: Component – FastAPI Backend

    subgraph fastapi_api["FastAPI Backend"]

        subgraph middleware["Middleware & Dependencies"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/><i>Cross-origin request handling<br/>CORSMiddleware</i>"]
            %% KIND: boundary
            auth_deps["Auth Dependencies<br/><i>OAuth2PasswordBearer<br/>JWT validation · CurrentUser</i>"]
        end

        subgraph routes["API Routes  /api/v1"]
            %% KIND: router
            login_routes["Login Routes<br/><i>/login/access-token<br/>/password-recovery · /reset-password</i>"]
            %% KIND: router
            users_routes["Users Routes<br/><i>/users CRUD · /users/me<br/>/users/signup</i>"]
            %% KIND: router
            items_routes["Items Routes<br/><i>/items CRUD<br/>owner-scoped access</i>"]
            %% KIND: router
            utils_routes["Utils Routes<br/><i>/utils/health-check<br/>/utils/test-email</i>"]
        end

        subgraph services["Core Services"]
            %% KIND: service_layer
            security_mod["Security Module<br/><i>JWT token creation (HS256)<br/>Argon2 + Bcrypt hashing</i>"]
            %% KIND: integration
            email_utils["Email Utilities<br/><i>Jinja2 templates · SMTP send<br/>password-reset · welcome emails</i>"]
            %% KIND: service_layer
            config_mod["Configuration<br/><i>Pydantic Settings<br/>env-based config</i>"]
        end

        subgraph data["Data Access"]
            %% KIND: data_access
            crud_layer["CRUD Layer<br/><i>create/read/update/delete<br/>User &amp; Item operations</i>"]
            %% KIND: data_access
            models_layer["SQLModel Models<br/><i>User · Item entities<br/>Pydantic schemas</i>"]
            %% KIND: data_access
            alembic_mig["Alembic Migrations<br/><i>Schema versioning<br/>5 migration revisions</i>"]
        end

    end

    pg["PostgreSQL"]
    smtp["SMTP Provider"]
    sentry["Sentry"]

    cors_mw --> auth_deps
    auth_deps --> login_routes
    auth_deps --> users_routes
    auth_deps --> items_routes
    auth_deps --> utils_routes

    login_routes --> security_mod
    login_routes --> crud_layer
    login_routes --> email_utils
    users_routes --> crud_layer
    users_routes --> security_mod
    items_routes --> crud_layer
    utils_routes --> email_utils

    security_mod --> config_mod
    crud_layer --> models_layer
    models_layer -- "SQL | psycopg" --> pg
    alembic_mig -- "DDL migrations" --> pg
    email_utils -- "SMTP/TLS" --> smtp

    fastapi_api -. "error tracking | HTTPS" .-> sentry
    config_mod -. "configures all modules" .-> fastapi_api
```

---

## L3: React Frontend

```mermaid
graph TD
    %% SCOPE: urn:c4:container:spa
    %% L3: Component – React Frontend

    subgraph spa["React Frontend"]

        subgraph routing["Routing"]
            %% KIND: router
            router["TanStack Router<br/><i>File-based routing<br/>auto code-splitting</i>"]
        end

        subgraph state["State Management"]
            %% KIND: service_layer
            query_client["TanStack Query<br/><i>Server-state cache<br/>queries &amp; mutations</i>"]
            %% KIND: service_layer
            theme_ctx["Theme Provider<br/><i>Light / Dark / System<br/>localStorage persistence</i>"]
            %% KIND: service_layer
            auth_hook["useAuth Hook<br/><i>Token management<br/>login / logout state</i>"]
        end

        subgraph features["Feature Modules"]
            %% KIND: boundary
            auth_pages["Auth Pages<br/><i>Login · Signup<br/>Password Recovery · Reset</i>"]
            %% KIND: boundary
            dashboard_feat["Dashboard<br/><i>User greeting<br/>overview page</i>"]
            %% KIND: boundary
            items_feat["Items Management<br/><i>CRUD with DataTable<br/>TanStack Table</i>"]
            %% KIND: boundary
            admin_feat["Admin Panel<br/><i>User management<br/>superuser-only</i>"]
            %% KIND: boundary
            settings_feat["User Settings<br/><i>Profile · Password<br/>Account deletion</i>"]
        end

        subgraph integration_layer["Integration"]
            %% KIND: integration
            api_client["OpenAPI Client<br/><i>Auto-generated SDK<br/>Axios · Bearer auth</i>"]
        end

        subgraph ui["UI Layer"]
            %% KIND: boundary
            ui_kit["UI Components<br/><i>Radix UI · shadcn/ui<br/>Tailwind CSS 4</i>"]
        end

    end

    fastapi_api["FastAPI Backend"]

    router --> auth_pages
    router --> dashboard_feat
    router --> items_feat
    router --> admin_feat
    router --> settings_feat

    auth_pages --> auth_hook
    auth_pages --> api_client
    auth_pages --> ui_kit
    dashboard_feat --> query_client
    dashboard_feat --> auth_hook
    dashboard_feat --> ui_kit
    items_feat --> query_client
    items_feat --> api_client
    items_feat --> ui_kit
    admin_feat --> query_client
    admin_feat --> api_client
    admin_feat --> ui_kit
    settings_feat --> query_client
    settings_feat --> api_client
    settings_feat --> ui_kit

    auth_hook --> api_client
    query_client --> api_client
    api_client -- "JSON / HTTPS" --> fastapi_api
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP – Full Stack FastAPI Platform
%% Every subgraph from every diagram appears as a CONTAINS entry.
%% Indentation denotes nesting depth.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [platform]
external           CONTAINS [smtp, sentry]

%% ── L2 platform internals ──────────────────────────────────────────
platform_boundary  CONTAINS [traefik, spa, fastapi_api, pg, adminer, prestart]

%% ── L3: FastAPI Backend (fastapi_api) ──────────────────────────────
fastapi_api        CONTAINS [middleware, routes, services, data]
  middleware       CONTAINS [cors_mw, auth_deps]
  routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services         CONTAINS [security_mod, email_utils, config_mod]
  data             CONTAINS [crud_layer, models_layer, alembic_mig]

%% ── L3: React Frontend (spa) ──────────────────────────────────────
spa                CONTAINS [routing, state, features, integration_layer, ui]
  routing          CONTAINS [router]
  state            CONTAINS [query_client, theme_ctx, auth_hook]
  features         CONTAINS [auth_pages, dashboard_feat, items_feat, admin_feat, settings_feat]
  integration_layer CONTAINS [api_client]
  ui               CONTAINS [ui_kit]
```

---

## Stable ID Cross-Reference

| Stable ID        | L1          | L2               | L3 Scope Boundary |
|------------------|-------------|------------------|--------------------|
| `user`           | actor node  | actor node       | —                  |
| `admin_user`     | actor node  | actor node       | —                  |
| `platform`       | system node | _(boundary)_     | —                  |
| `spa`            | —           | container node   | scope boundary (L3)|
| `fastapi_api`    | —           | container node   | scope boundary (L3)|
| `pg`             | —           | container node   | external on L3     |
| `smtp`           | external    | external         | external on L3     |
| `sentry`         | external    | external         | external on L3     |
| `traefik`        | —           | container node   | —                  |
| `adminer`        | —           | container node   | —                  |
| `prestart`       | —           | container node   | —                  |
