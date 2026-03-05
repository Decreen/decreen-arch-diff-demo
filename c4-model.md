# C4 Architecture Model — Full Stack FastAPI Platform

> Auto-generated C4 model for the codebase.
> Diagrams use [Mermaid](https://mermaid.js.org/) syntax with **stable IDs** across all levels.

---

## L1: System Context

```mermaid
graph TD
    %% SCOPE: urn:c4:context:platform

    subgraph users["Users"]
        user["User<br/>[Person]<br/>Browses items, manages<br/>own profile and data"]
        admin_user["Superuser<br/>[Person]<br/>Full administrative access<br/>to the platform"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform<br/>[Software System]<br/>Web application for user<br/>and item management"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Service<br/>[External System]<br/>Transactional email delivery"]
        sentry["Sentry<br/>[External System]<br/>Error tracking &amp; monitoring"]
        letsencrypt["Let's Encrypt<br/>[External System]<br/>Automated TLS certificates"]
    end

    user -->|"Uses web application"| platform
    admin_user -->|"Manages users &amp; content"| platform
    platform -->|"Sends emails via SMTP"| smtp
    platform -->|"Reports errors"| sentry
    platform -->|"Obtains TLS certs<br/>via ACME"| letsencrypt

    classDef person fill:#08427b,color:#fff,stroke:#073b6f
    classDef system fill:#1168bd,color:#fff,stroke:#0e59a2
    classDef ext fill:#999999,color:#fff,stroke:#707070
    classDef bnd fill:none,stroke:#ccc,stroke-dasharray:5 5

    class user,admin_user person
    class platform system
    class smtp,sentry,letsencrypt ext
```

---

## L2: Container

```mermaid
graph TD
    %% SCOPE: urn:c4:system:platform

    subgraph users["Users"]
        user["User<br/>[Person]<br/>Browses items, manages<br/>own profile and data"]
        admin_user["Superuser<br/>[Person]<br/>Full administrative access"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik<br/>[Container: Traefik 3.6]<br/>Reverse proxy, TLS<br/>termination, routing"]
        spa["React SPA<br/>[Container: React 19 / Vite / Nginx]<br/>Single-page application<br/>served as static files"]
        fastapi_api["FastAPI Backend<br/>[Container: Python / FastAPI]<br/>REST API, business logic,<br/>authentication"]
        pg["PostgreSQL<br/>[Container: PostgreSQL 18]<br/>Relational data store for<br/>users and items"]
        adminer["Adminer<br/>[Container: Adminer]<br/>Database administration UI"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Service<br/>[External System]<br/>Transactional email delivery"]
        sentry["Sentry<br/>[External System]<br/>Error tracking &amp; monitoring"]
        letsencrypt["Let's Encrypt<br/>[External System]<br/>Automated TLS certificates"]
    end

    user -->|"Accesses via browser<br/>[HTTPS]"| traefik
    admin_user -->|"Accesses via browser<br/>[HTTPS]"| traefik
    admin_user -->|"Manages database<br/>[HTTPS]"| adminer

    traefik -->|"Routes to dashboard.*<br/>[HTTP]"| spa
    traefik -->|"Routes to api.*<br/>[HTTP]"| fastapi_api
    traefik -->|"Obtains TLS certs<br/>[ACME]"| letsencrypt

    spa -->|"Makes API calls<br/>[HTTPS / JSON]"| fastapi_api

    fastapi_api -->|"Reads &amp; writes<br/>[SQL / psycopg]"| pg
    fastapi_api -->|"Sends emails<br/>[SMTP]"| smtp
    fastapi_api -->|"Reports errors<br/>[HTTPS]"| sentry

    adminer -->|"Queries<br/>[SQL]"| pg

    classDef person fill:#08427b,color:#fff,stroke:#073b6f
    classDef container fill:#438dd5,color:#fff,stroke:#3c7fc0
    classDef db fill:#438dd5,color:#fff,stroke:#3c7fc0
    classDef ext fill:#999999,color:#fff,stroke:#707070
    classDef bnd fill:none,stroke:#ccc,stroke-dasharray:5 5

    class user,admin_user person
    class traefik,spa,fastapi_api,adminer container
    class pg db
    class smtp,sentry,letsencrypt ext
```

---

## L3: FastAPI Backend

```mermaid
graph TD
    %% SCOPE: urn:c4:container:fastapi_api

    spa["React SPA<br/>[Container]"]
    pg["PostgreSQL<br/>[Container]"]
    smtp["SMTP Service<br/>[External]"]
    sentry["Sentry<br/>[External]"]

    subgraph fastapi_api["FastAPI Backend"]

        %% KIND: router
        subgraph routes["API Routes (/api/v1)"]
            login_routes["Login Routes<br/>[Component]<br/>OAuth2 token, password<br/>recovery &amp; reset"]
            users_routes["Users Routes<br/>[Component]<br/>User CRUD, profile,<br/>signup, admin ops"]
            items_routes["Items Routes<br/>[Component]<br/>Item CRUD with<br/>ownership enforcement"]
            utils_routes["Utils Routes<br/>[Component]<br/>Health check,<br/>test email"]
        end

        %% KIND: boundary
        subgraph middleware["Middleware &amp; Dependencies"]
            cors_mw["CORS Middleware<br/>[Component]<br/>Cross-origin request<br/>handling via Starlette"]
            %% KIND: boundary
            auth_deps["Auth Dependencies<br/>[Component]<br/>OAuth2 bearer extraction,<br/>JWT decode, current user<br/>and superuser resolution"]
        end

        %% KIND: service_layer
        subgraph services["Business Logic"]
            %% KIND: service_layer
            crud_layer["CRUD Layer<br/>[Component]<br/>create/read/update/delete<br/>for User and Item models"]
            %% KIND: integration
            email_utils["Email Service<br/>[Component]<br/>Jinja2 template rendering,<br/>SMTP sending, token generation"]
        end

        %% KIND: data_access
        subgraph data_layer["Data &amp; Configuration"]
            %% KIND: data_access
            models_layer["SQLModel Models<br/>[Component]<br/>User, Item tables and<br/>Pydantic request/response schemas"]
            %% KIND: storage
            core_db["Database Engine<br/>[Component]<br/>SQLAlchemy engine,<br/>session management"]
            %% KIND: service_layer
            core_security["Security Module<br/>[Component]<br/>Argon2/bcrypt password hashing,<br/>JWT token creation"]
            %% KIND: service_layer
            core_config["App Configuration<br/>[Component]<br/>Pydantic settings from<br/>environment variables"]
        end

    end

    spa -->|"API requests<br/>[HTTPS / JSON]"| routes
    routes --> auth_deps
    auth_deps --> core_security
    auth_deps --> core_db

    login_routes --> crud_layer
    login_routes --> email_utils
    users_routes --> crud_layer
    users_routes --> email_utils
    items_routes --> crud_layer

    crud_layer --> models_layer
    crud_layer --> core_db
    crud_layer --> core_security
    email_utils -->|"Sends emails<br/>[SMTP]"| smtp
    core_db -->|"SQL queries<br/>[psycopg]"| pg
    core_config -.->|"Configures"| cors_mw
    core_config -.->|"Configures"| auth_deps
    core_config -.->|"Configures"| core_db

    fastapi_api -.->|"Reports errors"| sentry

    classDef component fill:#85bbf0,color:#000,stroke:#5a9bd5
    classDef ext fill:#999999,color:#fff,stroke:#707070
    classDef container fill:#438dd5,color:#fff,stroke:#3c7fc0
    classDef bnd fill:none,stroke:#ccc,stroke-dasharray:5 5

    class login_routes,users_routes,items_routes,utils_routes component
    class cors_mw,auth_deps component
    class crud_layer,email_utils component
    class models_layer,core_db,core_security,core_config component
    class spa,pg container
    class smtp,sentry ext
```

---

## L3: React SPA

```mermaid
graph TD
    %% SCOPE: urn:c4:container:spa

    fastapi_api["FastAPI Backend<br/>[Container]"]

    subgraph spa["React SPA"]

        %% KIND: router
        subgraph routing["Routing"]
            tanstack_router["TanStack Router<br/>[Component]<br/>File-based routing with<br/>generated route tree"]
            %% KIND: boundary
            route_guards["Route Guards<br/>[Component]<br/>Auth redirect &amp; superuser<br/>role checks via beforeLoad"]
        end

        %% KIND: boundary
        subgraph features["Feature Modules"]
            auth_pages["Auth Pages<br/>[Component]<br/>Login, Signup, Password<br/>Recovery, Password Reset"]
            dashboard_page["Dashboard<br/>[Component]<br/>Welcome view with<br/>current user info"]
            items_feature["Items Management<br/>[Component]<br/>Item CRUD with DataTable,<br/>inline actions menu"]
            admin_feature["Admin Panel<br/>[Component]<br/>User management DataTable<br/>(superuser only)"]
            settings_feature["User Settings<br/>[Component]<br/>Profile edit, password change,<br/>account deletion"]
        end

        %% KIND: data_access
        subgraph data_access_fe["Data &amp; API Layer"]
            %% KIND: integration
            api_client["OpenAPI Client<br/>[Component]<br/>Auto-generated Axios client<br/>from OpenAPI spec"]
            %% KIND: data_access
            query_layer["TanStack Query<br/>[Component]<br/>Server-state cache, queries,<br/>mutations, error handling"]
            %% KIND: service_layer
            auth_hooks["Auth Hooks<br/>[Component]<br/>useAuth for login, logout,<br/>signup, current user state"]
        end

        %% KIND: boundary
        subgraph shared["Shared UI"]
            common_comp["Common Components<br/>[Component]<br/>DataTable, Layout,<br/>ErrorComponent, NotFound"]
            sidebar_comp["Sidebar<br/>[Component]<br/>Navigation, user menu,<br/>collapsible layout"]
            ui_lib["UI Library<br/>[Component]<br/>shadcn/ui primitives<br/>(Button, Dialog, Card, etc.)"]
            theme["Theme Provider<br/>[Component]<br/>Dark/light mode toggle<br/>via React Context"]
        end

    end

    tanstack_router --> route_guards
    route_guards --> features

    auth_pages --> auth_hooks
    auth_pages --> api_client
    items_feature --> query_layer
    items_feature --> api_client
    admin_feature --> query_layer
    admin_feature --> api_client
    settings_feature --> query_layer
    settings_feature --> api_client
    dashboard_page --> query_layer

    features --> common_comp
    features --> ui_lib
    features --> sidebar_comp

    query_layer --> api_client
    auth_hooks --> api_client
    api_client -->|"REST API calls<br/>[HTTPS / JSON]"| fastapi_api

    classDef component fill:#85bbf0,color:#000,stroke:#5a9bd5
    classDef container fill:#438dd5,color:#fff,stroke:#3c7fc0
    classDef bnd fill:none,stroke:#ccc,stroke-dasharray:5 5

    class tanstack_router,route_guards component
    class auth_pages,dashboard_page,items_feature,admin_feature,settings_feature component
    class api_client,query_layer,auth_hooks component
    class common_comp,sidebar_comp,ui_lib,theme component
    class fastapi_api container
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% Every subgraph ↔ CONTAINS entry.  Indentation = nesting depth.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [spa, fastapi_api, pg, traefik, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2→L3 internal containment (FastAPI Backend) ──────────────────
fastapi_api  CONTAINS [routes, middleware, services, data_layer]
  routes       CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  middleware   CONTAINS [cors_mw, auth_deps]
  services     CONTAINS [crud_layer, email_utils]
  data_layer   CONTAINS [models_layer, core_db, core_security, core_config]

%% ── L2→L3 internal containment (React SPA) ────────────────────────
spa  CONTAINS [routing, features, data_access_fe, shared]
  routing        CONTAINS [tanstack_router, route_guards]
  features       CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
  data_access_fe CONTAINS [api_client, query_layer, auth_hooks]
  shared         CONTAINS [common_comp, sidebar_comp, ui_lib, theme]
```

---

## ID Cross-Reference

| Stable ID | L1 | L2 | L3 (fastapi_api) | L3 (spa) | Description |
|---|---|---|---|---|---|
| `user` | actor | actor | — | — | Regular user |
| `admin_user` | actor | actor | — | — | Superuser / admin |
| `platform` | system node | — | — | — | L1 system abstraction |
| `spa` | — | container | — | **scope boundary** | React SPA |
| `fastapi_api` | — | container | **scope boundary** | — | FastAPI Backend |
| `pg` | — | container | external ref | — | PostgreSQL database |
| `traefik` | — | container | — | — | Reverse proxy |
| `adminer` | — | container | — | — | DB admin UI |
| `smtp` | external | external | external ref | — | SMTP email service |
| `sentry` | external | external | external ref | — | Error tracking |
| `letsencrypt` | external | external | — | — | TLS certificate authority |
