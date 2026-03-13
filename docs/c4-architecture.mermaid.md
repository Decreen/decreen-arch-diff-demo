# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:platform
flowchart TD

    subgraph users["Users"]
        user(["User\n[Person]\nAuthenticated user who\nmanages personal items"])
        admin_user(["Admin\n[Person]\nSuperuser who manages\nusers and system"])
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform\n[Software System]\nWeb application providing\nuser authentication, item\nmanagement, and administration"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server\n[External System]\nEmail delivery service"]
        sentry["Sentry\n[External System]\nError monitoring\nand performance tracing"]
        letsencrypt["Let's Encrypt\n[External System]\nAutomated TLS\ncertificate authority"]
    end

    user -->|"Uses web application\n[HTTPS]"| platform
    admin_user -->|"Manages users and system\n[HTTPS]"| platform
    platform -->|"Sends emails\n[SMTP]"| smtp
    platform -->|"Reports errors\n[HTTPS]"| sentry
    platform -->|"Obtains TLS certificates\n[ACME]"| letsencrypt

    classDef person fill:#08427B,color:#fff,stroke:#052E56
    classDef system fill:#1168BD,color:#fff,stroke:#0B4884
    classDef ext fill:#999999,color:#fff,stroke:#6B6B6B
    class user,admin_user person
    class platform system
    class smtp,sentry,letsencrypt ext
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:platform
flowchart TD

    subgraph users["Users"]
        user(["User\n[Person]"])
        admin_user(["Admin\n[Person]"])
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik\n[Container: Traefik 3.6]\nReverse proxy, TLS termination,\nrequest routing"]
        spa["React SPA\n[Container: React 19 / Vite / Nginx]\nSingle-page application\nserving the user interface"]
        fastapi["FastAPI Backend\n[Container: Python / FastAPI]\nREST API providing JWT auth,\nbusiness logic, CRUD operations"]
        pg[("PostgreSQL\n[Container: PostgreSQL 18]\nPrimary relational data store\nfor users and items")]
        adminer["Adminer\n[Container: Adminer]\nDatabase administration\nweb interface"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server\n[External System]\nEmail delivery"]
        sentry["Sentry\n[External System]\nError monitoring"]
        letsencrypt["Let's Encrypt\n[External System]\nTLS certificates"]
    end

    user -->|"Browses application\n[HTTPS]"| traefik
    admin_user -->|"Browses application\n[HTTPS]"| traefik
    traefik -->|"Serves static assets\n[HTTP]"| spa
    traefik -->|"Proxies API requests\n[HTTP]"| fastapi
    traefik -->|"Proxies DB admin\n[HTTP]"| adminer
    spa -->|"Makes API calls\n[REST / JSON]"| fastapi
    fastapi -->|"Reads and writes data\n[SQL / psycopg]"| pg
    adminer -->|"Queries database\n[SQL]"| pg
    fastapi -->|"Sends emails\n[SMTP]"| smtp
    fastapi -->|"Reports errors\n[HTTPS]"| sentry
    traefik -->|"Obtains certificates\n[ACME]"| letsencrypt

    classDef person fill:#08427B,color:#fff,stroke:#052E56
    classDef container fill:#1168BD,color:#fff,stroke:#0B4884
    classDef database fill:#2A7B3F,color:#fff,stroke:#1A5C2C
    classDef ext fill:#999999,color:#fff,stroke:#6B6B6B
    classDef infra fill:#3B6EA5,color:#fff,stroke:#2A5080
    class user,admin_user person
    class spa,fastapi container
    class pg database
    class traefik,adminer infra
    class smtp,sentry,letsencrypt ext
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi
flowchart TD

    traefik["Traefik\n[Container]"]

    subgraph fastapi["FastAPI Backend"]

        %% KIND: boundary
        subgraph mw["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware\n[Component: Starlette]\nHandles cross-origin\nrequest policies"]
            %% KIND: boundary
            auth_deps["Auth Dependencies\n[Component: FastAPI Depends]\nOAuth2 bearer token\nvalidation, current user\nresolution"]
        end

        %% KIND: router
        subgraph routes["API Routes"]
            %% KIND: router
            api_router["API Router\n[Component: APIRouter]\nAggregates all route\nmodules under /api/v1"]
            %% KIND: router
            login_routes["Login Routes\n[Component: APIRouter]\nJWT issuance, token test,\npassword recovery and reset"]
            %% KIND: router
            user_routes["User Routes\n[Component: APIRouter]\nUser CRUD, signup,\nprofile management"]
            %% KIND: router
            item_routes["Item Routes\n[Component: APIRouter]\nItem CRUD operations\nwith ownership enforcement"]
            %% KIND: router
            util_routes["Utils Routes\n[Component: APIRouter]\nHealth check,\ntest email endpoint"]
            %% KIND: router
            private_routes["Private Routes\n[Component: APIRouter]\nDev-only endpoints\nfor local environment"]
        end

        %% KIND: service_layer
        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud["CRUD Module\n[Component: Python]\nCreate, read, update, delete\noperations for users and items"]
            %% KIND: service_layer
            security["Security Module\n[Component: PyJWT / pwdlib]\nJWT creation and verification,\nArgon2/bcrypt password hashing"]
            %% KIND: integration
            email_utils["Email Utilities\n[Component: emails / Jinja2]\nEmail rendering from\ntemplates and SMTP dispatch"]
        end

        %% KIND: data_access
        subgraph data_layer["Data Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models\n[Component: SQLModel]\nUser and Item ORM models\nwith Pydantic schemas"]
            %% KIND: data_access
            db_core["DB Session Manager\n[Component: SQLAlchemy]\nSession factory and\nconnection pool management"]
            %% KIND: data_access
            config["Settings\n[Component: Pydantic Settings]\nApplication configuration\nfrom environment variables"]
            %% KIND: data_access
            alembic_mig["Alembic Migrations\n[Component: Alembic]\nDatabase schema versioning\nand migration runner"]
        end

    end

    pg[("PostgreSQL\n[Container]")]
    smtp["SMTP Server\n[External System]"]
    sentry["Sentry\n[External System]"]

    traefik -->|"HTTP requests"| cors_mw
    cors_mw --> api_router
    api_router --> login_routes
    api_router --> user_routes
    api_router --> item_routes
    api_router --> util_routes
    api_router --> private_routes
    login_routes --> auth_deps
    user_routes --> auth_deps
    item_routes --> auth_deps
    util_routes --> auth_deps
    login_routes --> crud
    login_routes --> security
    login_routes --> email_utils
    user_routes --> crud
    user_routes --> security
    user_routes --> email_utils
    item_routes --> crud
    private_routes --> crud
    util_routes --> email_utils
    crud --> models_layer
    crud --> db_core
    crud --> security
    db_core --> config
    db_core -->|"SQL / psycopg"| pg
    alembic_mig -->|"DDL migrations"| pg
    email_utils -->|"SMTP"| smtp
    security --> config
    fastapi -.->|"HTTPS"| sentry

    classDef container fill:#1168BD,color:#fff,stroke:#0B4884
    classDef component fill:#4B8BBE,color:#fff,stroke:#306998
    classDef database fill:#2A7B3F,color:#fff,stroke:#1A5C2C
    classDef ext fill:#999999,color:#fff,stroke:#6B6B6B
    classDef infra fill:#3B6EA5,color:#fff,stroke:#2A5080
    class traefik infra
    class cors_mw,auth_deps,api_router,login_routes,user_routes,item_routes,util_routes,private_routes component
    class crud,security,email_utils component
    class models_layer,db_core,config,alembic_mig component
    class pg database
    class smtp,sentry ext
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TD

    user(["User\n[Person]"])
    admin_user(["Admin\n[Person]"])

    subgraph spa["React SPA"]

        %% KIND: router
        subgraph routing["Routing"]
            %% KIND: router
            router["TanStack Router\n[Component: @tanstack/router]\nFile-based routing with\nauth guards and code splitting"]
        end

        %% KIND: service_layer
        subgraph contexts["State Management"]
            %% KIND: service_layer
            query_client["TanStack Query\n[Component: @tanstack/react-query]\nServer state cache,\nquery invalidation,\nglobal error handling"]
            %% KIND: service_layer
            auth_hook["Auth Hook\n[Component: React Hook]\nLogin/logout mutations,\nJWT token management\nin localStorage"]
            %% KIND: service_layer
            theme_ctx["Theme Provider\n[Component: React Context]\nLight/dark/system theme\npersisted in localStorage"]
        end

        %% KIND: boundary
        subgraph features["Features"]
            %% KIND: boundary
            dashboard_feat["Dashboard\n[Component: React]\nWelcome page with\nuser greeting"]
            %% KIND: boundary
            items_feat["Items Management\n[Component: React]\nCRUD interface with\ndata table, dialogs"]
            %% KIND: boundary
            admin_feat["Admin Panel\n[Component: React]\nUser management\n(superuser only)"]
            %% KIND: boundary
            settings_feat["User Settings\n[Component: React]\nProfile edit, password\nchange, account deletion"]
            %% KIND: boundary
            auth_pages["Auth Pages\n[Component: React]\nLogin, signup, password\nrecovery and reset"]
        end

        %% KIND: integration
        subgraph services_layer["Services"]
            %% KIND: integration
            api_client["OpenAPI Client\n[Component: @hey-api/openapi-ts]\nGenerated typed API client\nusing Axios for HTTP"]
        end

        %% KIND: boundary
        subgraph ui["UI Layer"]
            %% KIND: boundary
            ui_lib["Component Library\n[Component: shadcn/ui + Radix]\nReusable UI primitives:\nbuttons, dialogs, tables,\nforms, cards, tooltips"]
            %% KIND: boundary
            app_sidebar["App Sidebar\n[Component: React]\nNavigation, theme toggle,\nuser menu"]
        end

    end

    fastapi["FastAPI Backend\n[Container]"]

    user -->|"Interacts with UI\n[HTTPS / Browser]"| router
    admin_user -->|"Interacts with UI\n[HTTPS / Browser]"| router
    router --> dashboard_feat
    router --> items_feat
    router --> admin_feat
    router --> settings_feat
    router --> auth_pages
    auth_pages --> auth_hook
    items_feat --> query_client
    admin_feat --> query_client
    settings_feat --> query_client
    dashboard_feat --> query_client
    items_feat --> ui_lib
    admin_feat --> ui_lib
    settings_feat --> ui_lib
    auth_pages --> ui_lib
    dashboard_feat --> app_sidebar
    items_feat --> app_sidebar
    admin_feat --> app_sidebar
    settings_feat --> app_sidebar
    query_client --> api_client
    auth_hook --> api_client
    api_client -->|"REST API calls\n[JSON / HTTPS]"| fastapi

    classDef person fill:#08427B,color:#fff,stroke:#052E56
    classDef container fill:#1168BD,color:#fff,stroke:#0B4884
    classDef component fill:#4B8BBE,color:#fff,stroke:#306998
    class user,admin_user person
    class fastapi container
    class router,query_client,auth_hook,theme_ctx component
    class dashboard_feat,items_feat,admin_feat,settings_feat,auth_pages component
    class api_client component
    class ui_lib,app_sidebar component
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
%% users              CONTAINS [user, admin_user]
%% platform_boundary  CONTAINS [traefik, spa, fastapi, pg, adminer]
%% external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 containers (within platform_boundary) ────────────────────
%% traefik            (leaf — no internal structure)
%% pg                 (leaf — no internal structure)
%% adminer            (leaf — no internal structure)

%% ── L3: FastAPI Backend ─────────────────────────────────────────
%% fastapi            CONTAINS [mw, routes, services, data_layer]
%%   mw               CONTAINS [cors_mw, auth_deps]
%%   routes            CONTAINS [api_router, login_routes, user_routes, item_routes, util_routes, private_routes]
%%   services          CONTAINS [crud, security, email_utils]
%%   data_layer        CONTAINS [models_layer, db_core, config, alembic_mig]

%% ── L3: React SPA ──────────────────────────────────────────────
%% spa                CONTAINS [routing, contexts, features, services_layer, ui]
%%   routing           CONTAINS [router]
%%   contexts          CONTAINS [query_client, auth_hook, theme_ctx]
%%   features          CONTAINS [dashboard_feat, items_feat, admin_feat, settings_feat, auth_pages]
%%   services_layer    CONTAINS [api_client]
%%   ui                CONTAINS [ui_lib, app_sidebar]
```
