# C4 Architecture Model – Full Stack FastAPI Project

---

## L1: System Context

```mermaid
graph TB
    %% SCOPE: urn:c4:system-context:full-stack-fastapi

    user["User<br/><i>Regular application user who<br/>manages items and profile</i>"]
    admin["Admin<br/><i>Superuser who manages<br/>users and system</i>"]

    subgraph system_boundary["Full Stack FastAPI Project"]
        system["Full Stack FastAPI Project<br/><i>Full-stack web application with<br/>authentication, user management,<br/>and items CRUD</i>"]
    end

    subgraph external["External Services"]
        smtp["SMTP Provider<br/><i>Email delivery service</i>"]
        sentry["Sentry<br/><i>Error tracking and<br/>performance monitoring</i>"]
    end

    user -- "Uses · HTTPS" --> system
    admin -- "Manages · HTTPS" --> system
    system -- "Sends emails · SMTP/TLS" --> smtp
    system -- "Reports errors · HTTPS" --> sentry
```

---

## L2: Container

```mermaid
graph TB
    %% SCOPE: urn:c4:system:full-stack-fastapi

    user["User<br/><i>Regular application user</i>"]
    admin["Admin<br/><i>Superuser</i>"]

    subgraph system_boundary["Full Stack FastAPI Project"]
        traefik["Traefik Proxy<br/><i>Traefik 3.6<br/>Reverse proxy · TLS termination</i>"]
        spa["React SPA<br/><i>React 19 · TypeScript · Vite<br/>TanStack Router · shadcn/ui</i>"]
        fastapi["FastAPI Backend<br/><i>Python · FastAPI · SQLModel<br/>REST API · JWT auth</i>"]
        pg[("PostgreSQL<br/><i>PostgreSQL 18<br/>Application database</i>")]
    end

    subgraph external["External Services"]
        smtp["SMTP Provider<br/><i>Email delivery service</i>"]
        sentry["Sentry<br/><i>Error tracking</i>"]
    end

    user -- "HTTPS" --> traefik
    admin -- "HTTPS" --> traefik
    traefik -- "Serves frontend · HTTP" --> spa
    traefik -- "Routes /api/* · HTTP" --> fastapi
    spa -- "API calls · JSON/HTTPS" --> fastapi
    fastapi -- "Reads/writes · SQL" --> pg
    fastapi -- "Sends emails · SMTP/TLS" --> smtp
    fastapi -- "Reports errors · HTTPS" --> sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TB
    %% SCOPE: urn:c4:container:fastapi

    spa["React SPA"]
    pg[("PostgreSQL")]
    smtp["SMTP Provider"]
    sentry["Sentry"]

    subgraph fastapi["FastAPI Backend"]

        %% KIND: boundary
        cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware<br/>Cross-origin request handling</i>"]

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes<br/><i>/login/*<br/>Token issue · password recovery/reset</i>"]
            %% KIND: router
            user_routes["User Routes<br/><i>/users/*<br/>User CRUD · profile · registration</i>"]
            %% KIND: router
            item_routes["Item Routes<br/><i>/items/*<br/>Item CRUD operations</i>"]
            %% KIND: router
            util_routes["Utility Routes<br/><i>/utils/*<br/>Health check · test email</i>"]
        end

        subgraph deps_layer["Request Dependencies"]
            %% KIND: boundary
            auth_deps["Auth Dependencies<br/><i>OAuth2PasswordBearer<br/>Token validation · current user extraction</i>"]
            %% KIND: data_access
            db_deps["DB Session<br/><i>SQLModel Session<br/>Per-request database session</i>"]
        end

        subgraph business_layer["Business Logic"]
            %% KIND: service_layer
            crud["CRUD Module<br/><i>crud.py<br/>create/read/update/delete · authenticate</i>"]
            %% KIND: integration
            email_utils["Email Utilities<br/><i>utils.py<br/>Render Jinja2 templates · send via SMTP</i>"]
        end

        subgraph core_layer["Core"]
            %% KIND: service_layer
            core_security["Security Module<br/><i>security.py<br/>JWT create/verify · Argon2/bcrypt hashing</i>"]
            %% KIND: boundary
            core_config["Configuration<br/><i>Pydantic Settings<br/>App settings from environment</i>"]
        end

        subgraph data_layer["Data Layer"]
            %% KIND: data_access
            models["SQLModel Models<br/><i>models.py<br/>User · Item tables and Pydantic schemas</i>"]
            %% KIND: storage
            core_db["Database Engine<br/><i>db.py<br/>SQLAlchemy engine · session factory · init_db</i>"]
            %% KIND: data_access
            alembic_mig["Alembic Migrations<br/><i>alembic/versions/<br/>Schema version control</i>"]
        end

    end

    spa -- "API requests" --> cors_mw
    cors_mw --> login_routes
    cors_mw --> user_routes
    cors_mw --> item_routes
    cors_mw --> util_routes

    login_routes --> auth_deps
    login_routes --> crud
    login_routes --> email_utils
    login_routes --> core_security
    user_routes --> auth_deps
    user_routes --> crud
    item_routes --> auth_deps
    item_routes --> crud
    util_routes --> email_utils

    auth_deps --> core_security
    auth_deps --> db_deps
    db_deps --> core_db

    crud --> models
    crud --> core_security
    models --> core_db
    core_db -- "SQL/TCP" --> pg
    alembic_mig -- "Migrates schema" --> pg

    core_db --> core_config
    core_security --> core_config
    email_utils --> core_config
    email_utils -- "SMTP/TLS" --> smtp
```

---

## L3: React SPA

```mermaid
graph TB
    %% SCOPE: urn:c4:container:spa

    fastapi["FastAPI Backend"]

    subgraph spa["React SPA"]

        %% KIND: router
        tanstack_router["TanStack Router<br/><i>@tanstack/react-router<br/>File-based routing · auth guards</i>"]

        subgraph pages["Pages"]
            %% KIND: boundary
            auth_pages["Auth Pages<br/><i>Login · Signup<br/>Recover Password · Reset Password</i>"]
            %% KIND: boundary
            dashboard_page["Dashboard<br/><i>User greeting<br/>and overview</i>"]
            %% KIND: boundary
            items_page["Items Page<br/><i>Items CRUD<br/>with DataTable</i>"]
            %% KIND: boundary
            admin_page["Admin Page<br/><i>User management<br/>(superuser only)</i>"]
            %% KIND: boundary
            settings_page["Settings Page<br/><i>Profile · Password<br/>Account deletion</i>"]
        end

        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            query_client["Query Client<br/><i>TanStack Query v5<br/>Server state · caching · error handling</i>"]
            %% KIND: service_layer
            auth_hook["Auth Hook<br/><i>useAuth<br/>Token management · current user</i>"]
            %% KIND: boundary
            theme_provider["Theme Provider<br/><i>React Context<br/>Dark / light / system theme</i>"]
        end

        subgraph services_layer["Services"]
            %% KIND: integration
            api_client["API Client<br/><i>@hey-api/openapi-ts<br/>Generated type-safe HTTP client</i>"]
        end

        subgraph ui_layer["UI Layer"]
            %% KIND: boundary
            layout_cmp["Layout Components<br/><i>AppSidebar · Footer<br/>Protected layout wrapper</i>"]
            %% KIND: boundary
            ui_lib["UI Library<br/><i>shadcn/ui · Radix · Tailwind<br/>Buttons · dialogs · forms · tables</i>"]
        end

    end

    tanstack_router --> auth_pages
    tanstack_router --> dashboard_page
    tanstack_router --> items_page
    tanstack_router --> admin_page
    tanstack_router --> settings_page
    tanstack_router --> layout_cmp

    auth_pages --> api_client
    auth_pages --> auth_hook
    auth_pages --> ui_lib
    items_page --> api_client
    items_page --> query_client
    items_page --> ui_lib
    admin_page --> api_client
    admin_page --> query_client
    admin_page --> ui_lib
    settings_page --> api_client
    settings_page --> auth_hook
    settings_page --> ui_lib
    dashboard_page --> auth_hook

    auth_hook --> api_client
    query_client --> api_client
    layout_cmp --> ui_lib

    api_client -- "JSON/HTTPS" --> fastapi
```

---

## Containment Map

```text
%% ── CONTAINMENT MAP ──────────────────────────────────────────────
%%
%% Every subgraph from every diagram is listed.
%% Nesting is expressed by indentation.
%%
%% ── L1 top-level groups ─────────────────────────────────────────
system_boundary  CONTAINS [system]
external         CONTAINS [smtp, sentry]

%% ── L2 system internals (system_boundary zoomed in) ─────────────
system_boundary  CONTAINS [traefik, spa, fastapi, pg]
external         CONTAINS [smtp, sentry]

%% ── L3: FastAPI Backend ──────────────────────────────────────────
fastapi          CONTAINS [cors_mw, routes, deps_layer, business_layer, core_layer, data_layer]
  routes         CONTAINS [login_routes, user_routes, item_routes, util_routes]
  deps_layer     CONTAINS [auth_deps, db_deps]
  business_layer CONTAINS [crud, email_utils]
  core_layer     CONTAINS [core_security, core_config]
  data_layer     CONTAINS [models, core_db, alembic_mig]

%% ── L3: React SPA ───────────────────────────────────────────────
spa              CONTAINS [tanstack_router, pages, state_mgmt, services_layer, ui_layer]
  pages          CONTAINS [auth_pages, dashboard_page, items_page, admin_page, settings_page]
  state_mgmt     CONTAINS [query_client, auth_hook, theme_provider]
  services_layer CONTAINS [api_client]
  ui_layer       CONTAINS [layout_cmp, ui_lib]
```

---

## ID Cross-Reference

Stable IDs that appear across multiple diagrams:

| ID | L1 | L2 | L3: FastAPI | L3: SPA | Role |
|----|----|----|-------------|---------|------|
| `user` | Actor | Actor | — | — | Regular user |
| `admin` | Actor | Actor | — | — | Superuser |
| `system_boundary` | Subgraph | Subgraph | — | — | System boundary |
| `system` | Node | — | — | — | L1 system black box |
| `traefik` | — | Container | — | — | Reverse proxy |
| `spa` | — | Container | — | Scope boundary | React SPA |
| `fastapi` | — | Container | Scope boundary | External ref | FastAPI Backend |
| `pg` | — | Container DB | External ref | — | PostgreSQL |
| `smtp` | External | External | External ref | — | SMTP Provider |
| `sentry` | External | External | External ref | — | Error tracking |
| `external` | Subgraph | Subgraph | — | — | External services group |
