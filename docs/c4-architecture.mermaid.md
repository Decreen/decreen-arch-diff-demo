# C4 Architecture Model — Full Stack FastAPI Platform

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI Platform.

---

## L1: System Context

```mermaid
graph TD
    %% SCOPE: urn:c4:system:platform

    subgraph users["Users"]
        user["<b>User</b><br/>Regular user who manages<br/>items and account settings"]
        admin_user["<b>Admin</b><br/>Superuser who manages<br/>users and the platform"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["<b>Full Stack FastAPI Platform</b><br/>Web application for item management<br/>with authentication and administration"]
    end

    subgraph external["External Systems"]
        smtp["<b>SMTP Server</b><br/>Email delivery service"]
        sentry["<b>Sentry</b><br/>Error monitoring and tracking"]
        letsencrypt["<b>Let's Encrypt</b><br/>TLS certificate authority"]
    end

    user -->|"Browses UI, manages items"| platform
    admin_user -->|"Manages users and platform"| platform
    platform -->|"Sends emails via SMTP"| smtp
    platform -->|"Reports errors via SDK"| sentry
    platform -->|"Obtains TLS certificates"| letsencrypt

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff

    class user,admin_user person
    class platform system
    class smtp,sentry,letsencrypt ext
```

---

## L2: Container

```mermaid
graph TD
    %% SCOPE: urn:c4:system:platform

    user["<b>User</b>"]
    admin_user["<b>Admin</b>"]

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["<b>Traefik</b><br/>Reverse proxy with TLS<br/>termination and routing<br/><i>Traefik 3.6</i>"]
        spa["<b>React SPA</b><br/>Single-page application<br/>serving the web UI<br/><i>React 19 · Vite · TypeScript</i>"]
        fastapi["<b>FastAPI Backend</b><br/>REST API server handling<br/>business logic and data access<br/><i>Python · FastAPI</i>"]
        pg["<b>PostgreSQL</b><br/>Primary relational database<br/>storing users and items<br/><i>PostgreSQL 18</i>"]
        adminer["<b>Adminer</b><br/>Database administration UI<br/><i>Adminer</i>"]
    end

    subgraph external["External Systems"]
        smtp["<b>SMTP Server</b><br/>Email delivery"]
        sentry["<b>Sentry</b><br/>Error monitoring"]
        letsencrypt["<b>Let's Encrypt</b><br/>TLS certificates"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"Serves static assets"| spa
    traefik -->|"Proxies /api requests"| fastapi
    traefik -->|"Proxies DB admin UI"| adminer
    spa -->|"REST / JSON over HTTP"| fastapi
    fastapi -->|"SQL over TCP"| pg
    adminer -->|"SQL over TCP"| pg
    fastapi -->|"SMTP"| smtp
    fastapi -->|"Sentry SDK"| sentry
    traefik -->|"ACME protocol"| letsencrypt

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
    classDef db fill:#438dd5,stroke:#2e6295,color:#fff

    class user,admin_user person
    class traefik,spa,fastapi,adminer container
    class pg db
    class smtp,sentry,letsencrypt ext
```

---

## L3: FastAPI Backend

```mermaid
graph TD
    %% SCOPE: urn:c4:container:fastapi

    spa["<b>React SPA</b>"]
    pg["<b>PostgreSQL</b>"]
    smtp["<b>SMTP Server</b>"]

    subgraph fastapi["FastAPI Backend"]

        %% KIND: boundary
        subgraph mw["Middleware and Dependencies"]
            %% KIND: router
            cors_mw["<b>CORS Middleware</b><br/>Cross-origin request handling<br/><i>Starlette CORSMiddleware</i>"]
            %% KIND: router
            auth_deps["<b>Auth Dependencies</b><br/>JWT validation · user loading<br/>superuser authorization<br/><i>OAuth2 Bearer Token</i>"]
        end

        %% KIND: boundary
        subgraph routes["API Routes · /api/v1"]
            %% KIND: router
            login_routes["<b>Login Routes</b><br/>Authentication · token issuance<br/>password recovery and reset<br/><i>POST /login/*</i>"]
            %% KIND: router
            users_routes["<b>Users Routes</b><br/>User CRUD · profile management<br/>signup<br/><i>/users/*</i>"]
            %% KIND: router
            items_routes["<b>Items Routes</b><br/>Item CRUD operations<br/><i>/items/*</i>"]
            %% KIND: router
            utils_routes["<b>Utils Routes</b><br/>Health check · test email<br/><i>/utils/*</i>"]
            %% KIND: router
            private_routes["<b>Private Routes</b><br/>Internal user creation<br/>(local environment only)<br/><i>/private/*</i>"]
        end

        %% KIND: data_access
        crud_layer["<b>CRUD Layer</b><br/>Reusable database operations<br/>for users and items<br/><i>crud.py</i>"]

        %% KIND: data_access
        models_layer["<b>SQLModel Models</b><br/>User and Item table definitions<br/>with Pydantic request/response schemas<br/><i>models.py</i>"]

        %% KIND: boundary
        subgraph core["Core Services"]
            %% KIND: service_layer
            core_config["<b>Configuration</b><br/>Pydantic Settings for all<br/>environment variables<br/><i>core/config.py</i>"]
            %% KIND: service_layer
            core_security["<b>Security</b><br/>JWT creation · password hashing<br/>(Argon2 / Bcrypt)<br/><i>core/security.py</i>"]
            %% KIND: data_access
            core_db["<b>Database Engine</b><br/>SQLAlchemy engine and<br/>session management<br/><i>core/db.py</i>"]
        end

        %% KIND: integration
        email_utils["<b>Email Utils</b><br/>Email composition (Jinja2 templates)<br/>and SMTP delivery<br/><i>utils.py</i>"]

    end

    spa -->|"REST / JSON"| cors_mw

    auth_deps --> core_security
    auth_deps --> core_db

    login_routes --> crud_layer
    login_routes --> core_security
    login_routes --> email_utils
    users_routes --> crud_layer
    users_routes --> core_security
    items_routes --> crud_layer
    utils_routes --> email_utils
    private_routes --> crud_layer

    crud_layer --> models_layer
    models_layer --> core_db
    core_db -->|"SQL"| pg
    email_utils -->|"SMTP"| smtp

    core_config -.-> core_security
    core_config -.-> core_db
    core_config -.-> email_utils

    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000

    class spa,pg,smtp ext
    class cors_mw,auth_deps component
    class login_routes,users_routes,items_routes,utils_routes,private_routes component
    class crud_layer,models_layer component
    class core_config,core_security,core_db component
    class email_utils component
```

---

## L3: React SPA

```mermaid
graph TD
    %% SCOPE: urn:c4:container:spa

    fastapi["<b>FastAPI Backend</b>"]

    subgraph spa["React SPA"]

        %% KIND: router
        router["<b>TanStack Router</b><br/>File-based routing with<br/>auth guards and lazy loading<br/><i>@tanstack/react-router</i>"]

        %% KIND: boundary
        subgraph pages["Pages"]
            %% KIND: boundary
            auth_pages["<b>Auth Pages</b><br/>Login · Signup · Recover<br/>and Reset Password"]
            %% KIND: boundary
            dashboard_page["<b>Dashboard</b><br/>Main landing page<br/>after authentication"]
            %% KIND: boundary
            items_feature["<b>Items Feature</b><br/>Item CRUD with data table<br/>add/edit/delete dialogs"]
            %% KIND: boundary
            admin_feature["<b>Admin Feature</b><br/>User management panel<br/>(superuser only)"]
            %% KIND: boundary
            settings_feature["<b>Settings Feature</b><br/>Profile · password · appearance<br/>and account deletion"]
        end

        %% KIND: boundary
        subgraph shared["Shared"]
            %% KIND: service_layer
            common_components["<b>Common Components</b><br/>AuthLayout · DataTable · Footer<br/>Logo · NotFound · ErrorComponent"]
            %% KIND: boundary
            ui_primitives["<b>UI Primitives</b><br/>shadcn/Radix components: Button<br/>Dialog · Form · Table · Sidebar"]
            %% KIND: service_layer
            hooks_layer["<b>Custom Hooks</b><br/>useAuth · useCustomToast<br/>useMobile · useCopyToClipboard"]
        end

        %% KIND: integration
        api_client["<b>OpenAPI Client</b><br/>Auto-generated TypeScript SDK<br/>from FastAPI OpenAPI schema<br/><i>@hey-api/openapi-ts · Axios</i>"]

        %% KIND: service_layer
        theme_provider["<b>Theme Provider</b><br/>Dark / light / system theme<br/>via React Context<br/><i>next-themes</i>"]

    end

    router --> auth_pages
    router --> dashboard_page
    router --> items_feature
    router --> admin_feature
    router --> settings_feature

    auth_pages --> hooks_layer
    auth_pages --> common_components
    items_feature --> hooks_layer
    items_feature --> common_components
    admin_feature --> hooks_layer
    admin_feature --> common_components
    settings_feature --> hooks_layer
    settings_feature --> common_components
    dashboard_page --> hooks_layer

    common_components --> ui_primitives
    hooks_layer --> api_client

    api_client -->|"REST / JSON"| fastapi

    theme_provider -.-> pages
    theme_provider -.-> shared

    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000

    class fastapi ext
    class router component
    class auth_pages,dashboard_page,items_feature,admin_feature,settings_feature component
    class common_components,ui_primitives,hooks_layer component
    class api_client,theme_provider component
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
%% users              CONTAINS [user, admin_user]
%% platform_boundary  CONTAINS [platform]
%% external           CONTAINS [smtp, sentry, letsencrypt]
%%
%% ── L2 top-level groups ─────────────────────────────────────────
%% platform_boundary  CONTAINS [traefik, spa, fastapi, pg, adminer]
%% external           CONTAINS [smtp, sentry, letsencrypt]
%%
%% ── L3: FastAPI Backend (urn:c4:container:fastapi) ──────────────
%% fastapi            CONTAINS [mw, routes, crud_layer, models_layer, core, email_utils]
%%   mw               CONTAINS [cors_mw, auth_deps]
%%   routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes, private_routes]
%%   core             CONTAINS [core_config, core_security, core_db]
%%
%% ── L3: React SPA (urn:c4:container:spa) ────────────────────────
%% spa                CONTAINS [router, pages, shared, api_client, theme_provider]
%%   pages            CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
%%   shared           CONTAINS [common_components, ui_primitives, hooks_layer]
```
