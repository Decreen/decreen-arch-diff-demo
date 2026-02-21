# C4 Architecture Model — Full Stack FastAPI Project

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fastapi_platform
flowchart TD
    subgraph users [" Users "]
        end_user["<b>End User</b><br/>Manages personal items"]
        admin_user["<b>Admin User</b><br/>Manages users &amp; system"]
    end

    subgraph fastapi_platform [" Full Stack FastAPI Project "]
        fastapi_platform_desc["Item management web application<br/><i>FastAPI · React · PostgreSQL</i>"]
    end

    subgraph external [" External Systems "]
        smtp["<b>SMTP Server</b><br/>Email delivery"]
        sentry["<b>Sentry</b><br/>Error tracking &amp; monitoring"]
    end

    end_user -->|"Uses [HTTPS]"| fastapi_platform
    admin_user -->|"Manages [HTTPS]"| fastapi_platform
    fastapi_platform -->|"Sends emails [SMTP/TLS]"| smtp
    fastapi_platform -.->|"Reports errors [HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fastapi_platform
flowchart TD
    subgraph users [" Users "]
        end_user["<b>End User</b><br/>Manages personal items"]
        admin_user["<b>Admin User</b><br/>Manages users &amp; system"]
    end

    subgraph fastapi_platform [" Full Stack FastAPI Project "]
        traefik["<b>Traefik</b><br/><i>Reverse proxy / load balancer</i><br/>[Docker · traefik:3.6]"]
        react_spa["<b>React SPA</b><br/><i>Single-page application</i><br/>[React · TypeScript · Vite · Tailwind · shadcn/ui]"]
        fastapi_backend["<b>FastAPI Backend</b><br/><i>REST API server</i><br/>[Python · FastAPI · SQLModel]"]
        pg[("<b>PostgreSQL</b><br/><i>Relational database</i><br/>[postgres:18]")]
        adminer["<b>Adminer</b><br/><i>Database admin UI</i>"]
    end

    subgraph external [" External Systems "]
        smtp["<b>SMTP Server</b><br/>Email delivery"]
        sentry["<b>Sentry</b><br/>Error tracking &amp; monitoring"]
    end

    end_user -->|"Browses [HTTPS]"| traefik
    admin_user -->|"Manages [HTTPS]"| traefik
    traefik -->|"Serves static files [HTTP]"| react_spa
    traefik -->|"Proxies API calls [HTTP]"| fastapi_backend
    traefik -->|"Proxies DB admin [HTTP]"| adminer
    react_spa -.->|"API calls [JSON/HTTPS via Traefik]"| fastapi_backend
    fastapi_backend -->|"Reads &amp; writes data [SQL/TCP]"| pg
    adminer -->|"Queries [SQL/TCP]"| pg
    fastapi_backend -->|"Sends emails [SMTP/TLS]"| smtp
    fastapi_backend -.->|"Reports errors [HTTPS]"| sentry
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TD
    react_spa["React SPA<br/><i>[External container]</i>"]

    subgraph fastapi_backend [" FastAPI Backend "]

        %% KIND: boundary
        subgraph middleware [" Middleware "]
            %% KIND: boundary
            cors_mw["<b>CORS Middleware</b><br/><i>Allows cross-origin requests</i><br/>[starlette.middleware.cors]"]
        end

        %% KIND: boundary
        subgraph deps [" Dependencies / Guards "]
            %% KIND: boundary
            auth_deps["<b>Auth Dependencies</b><br/><i>OAuth2 bearer, JWT validation,<br/>current-user &amp; superuser guards</i><br/>[app.api.deps]"]
        end

        %% KIND: router
        subgraph routes [" API Routes · /api/v1 "]
            %% KIND: router
            login_routes["<b>Login Routes</b><br/><i>Token login, password recovery,<br/>password reset</i><br/>[app.api.routes.login]"]
            %% KIND: router
            user_routes["<b>User Routes</b><br/><i>CRUD users, signup,<br/>profile &amp; password update</i><br/>[app.api.routes.users]"]
            %% KIND: router
            item_routes["<b>Item Routes</b><br/><i>CRUD items with<br/>ownership enforcement</i><br/>[app.api.routes.items]"]
            %% KIND: router
            utils_routes["<b>Utils Routes</b><br/><i>Health check, test email</i><br/>[app.api.routes.utils]"]
        end

        %% KIND: service_layer
        crud_layer["<b>CRUD Layer</b><br/><i>Create / read / update / delete<br/>operations for User &amp; Item</i><br/>[app.crud]"]

        %% KIND: data_access
        models_layer["<b>SQLModel Models</b><br/><i>User, Item, Token, and<br/>Pydantic schemas</i><br/>[app.models]"]

        %% KIND: service_layer
        subgraph core [" Core "]
            %% KIND: service_layer
            core_config["<b>Settings &amp; Config</b><br/><i>Pydantic settings from .env,<br/>DB DSN, CORS origins, SMTP</i><br/>[app.core.config]"]
            %% KIND: service_layer
            core_security["<b>Security</b><br/><i>JWT creation &amp; verification,<br/>Argon2/Bcrypt password hashing</i><br/>[app.core.security]"]
            %% KIND: storage
            core_db["<b>Database Engine</b><br/><i>SQLAlchemy engine &amp; session factory</i><br/>[app.core.db]"]
        end

        %% KIND: integration
        email_utils["<b>Email Utilities</b><br/><i>Render Jinja2 templates,<br/>send via SMTP, token generation</i><br/>[app.utils]"]

    end

    pg[("<b>PostgreSQL</b><br/><i>[External container]</i>")]
    smtp["<b>SMTP Server</b><br/><i>[External system]</i>"]
    sentry["<b>Sentry</b><br/><i>[External system]</i>"]

    react_spa -->|"HTTP requests"| cors_mw
    cors_mw --> auth_deps
    auth_deps --> routes

    login_routes --> crud_layer
    user_routes --> crud_layer
    item_routes --> crud_layer

    login_routes --> email_utils
    user_routes --> email_utils
    utils_routes --> email_utils

    crud_layer --> models_layer
    crud_layer --> core_db
    auth_deps --> core_security
    core_security --> core_config
    core_db --> core_config

    core_db -->|"SQL [psycopg]"| pg
    email_utils -->|"SMTP/TLS"| smtp
    fastapi_backend -.->|"HTTPS"| sentry
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:react_spa
flowchart TD
    fastapi_backend["FastAPI Backend<br/><i>[External container]</i>"]

    subgraph react_spa [" React SPA "]

        %% KIND: router
        tanstack_router["<b>TanStack Router</b><br/><i>File-based routing,<br/>auth guards, layout nesting</i><br/>[src/routeTree.gen.ts]"]

        %% KIND: boundary
        subgraph pages [" Pages / Views "]
            %% KIND: boundary
            auth_views["<b>Auth Pages</b><br/><i>Login, Signup,<br/>Password Recovery &amp; Reset</i><br/>[src/routes/login · signup · …]"]
            %% KIND: boundary
            admin_views["<b>Admin Pages</b><br/><i>User management (superuser)</i><br/>[src/routes/_layout/admin]"]
            %% KIND: boundary
            items_views["<b>Items Pages</b><br/><i>Item CRUD with data table</i><br/>[src/routes/_layout/items]"]
            %% KIND: boundary
            settings_views["<b>Settings Pages</b><br/><i>Profile, password, appearance,<br/>account deletion</i><br/>[src/routes/_layout/settings]"]
        end

        %% KIND: service_layer
        subgraph services_layer [" Services "]
            %% KIND: integration
            api_client["<b>API Client</b><br/><i>Auto-generated from OpenAPI spec<br/>(ItemsService, UsersService,<br/>LoginService, UtilsService)</i><br/>[src/client]"]
            %% KIND: service_layer
            auth_hook["<b>useAuth Hook</b><br/><i>Login, logout, signup mutations,<br/>current-user query, token storage</i><br/>[src/hooks/useAuth]"]
        end

        %% KIND: boundary
        subgraph ui_layer [" UI Layer "]
            %% KIND: boundary
            sidebar["<b>Sidebar Navigation</b><br/><i>App sidebar, user menu,<br/>main nav links</i><br/>[src/components/Sidebar]"]
            %% KIND: boundary
            ui_components["<b>UI Component Library</b><br/><i>shadcn/ui primitives: Button, Dialog,<br/>Table, Form, Card, Tabs, …</i><br/>[src/components/ui]"]
            %% KIND: service_layer
            theme_provider["<b>Theme Provider</b><br/><i>Dark/light mode toggle,<br/>persisted in localStorage</i><br/>[src/components/theme-provider]"]
        end

    end

    tanstack_router --> auth_views
    tanstack_router --> admin_views
    tanstack_router --> items_views
    tanstack_router --> settings_views

    auth_views --> auth_hook
    admin_views --> api_client
    items_views --> api_client
    settings_views --> api_client
    auth_hook --> api_client

    auth_views --> ui_components
    admin_views --> ui_components
    items_views --> ui_components
    settings_views --> ui_components

    admin_views --> sidebar
    items_views --> sidebar
    settings_views --> sidebar

    api_client -->|"JSON/HTTPS"| fastapi_backend
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Project
%% Covers L1, L2, L3-Backend, L3-SPA
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [end_user, admin_user]
fastapi_platform   CONTAINS [traefik, react_spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2 → L3 internal containment ───────────────────────────────

%% FastAPI Backend (L3)
fastapi_backend    CONTAINS [middleware, deps, routes, crud_layer, models_layer, core, email_utils]
  middleware       CONTAINS [cors_mw]
  deps             CONTAINS [auth_deps]
  routes           CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  core             CONTAINS [core_config, core_security, core_db]

%% React SPA (L3)
react_spa          CONTAINS [tanstack_router, pages, services_layer, ui_layer]
  pages            CONTAINS [auth_views, admin_views, items_views, settings_views]
  services_layer   CONTAINS [api_client, auth_hook]
  ui_layer         CONTAINS [sidebar, ui_components, theme_provider]
```
