# C4 Architecture Model – Full Stack FastAPI Platform

> Auto-generated C4 model for the Full Stack FastAPI Template codebase.

---

## L1: System Context

```mermaid
graph TD
    %% SCOPE: urn:c4:landscape:platform

    subgraph users["Users"]
        user(["End User<br/>[Person]<br/>Uses the web application<br/>via browser"])
        admin_user(["Admin<br/>[Person]<br/>Superuser with elevated<br/>privileges"])
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform<br/>[Software System]<br/>Web application for user<br/>and item management"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server<br/>[External System]<br/>Email delivery service"]
        sentry["Sentry<br/>[External System]<br/>Error monitoring and tracking"]
    end

    user -->|"Uses<br/>[HTTPS]"| platform
    admin_user -->|"Manages users and items<br/>[HTTPS]"| platform
    platform -->|"Sends emails<br/>[SMTP]"| smtp
    platform -->|"Reports errors<br/>[HTTPS]"| sentry

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef system fill:#1168bd,color:#fff,stroke:#0b4884
    classDef ext fill:#999999,color:#fff,stroke:#6b6b6b

    class user,admin_user person
    class platform system
    class smtp,sentry ext
```

---

## L2: Container

```mermaid
graph TD
    %% SCOPE: urn:c4:system:platform

    user(["End User<br/>[Person]"])
    admin_user(["Admin<br/>[Person]"])

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik_proxy["Traefik<br/>[Container: Reverse Proxy]<br/>TLS termination,<br/>domain-based routing"]
        spa["React SPA<br/>[Container: TypeScript / React 19]<br/>Single-page application<br/>served by Nginx"]
        fastapi_api["FastAPI Backend<br/>[Container: Python / FastAPI]<br/>REST API, JWT auth,<br/>business logic"]
        pg[("PostgreSQL 18<br/>[Container: Database]<br/>Stores users, items,<br/>and application data")]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server<br/>[External System]<br/>Email delivery"]
        sentry["Sentry<br/>[External System]<br/>Error monitoring"]
    end

    user -->|"Accesses<br/>[HTTPS]"| traefik_proxy
    admin_user -->|"Accesses<br/>[HTTPS]"| traefik_proxy
    traefik_proxy -->|"Serves frontend<br/>[HTTP]"| spa
    traefik_proxy -->|"Routes API requests<br/>[HTTP]"| fastapi_api
    spa -->|"Makes API calls<br/>[HTTPS / JSON]"| fastapi_api
    fastapi_api -->|"Reads and writes data<br/>[psycopg / SQL]"| pg
    fastapi_api -->|"Sends emails<br/>[SMTP]"| smtp
    fastapi_api -->|"Reports errors<br/>[HTTPS]"| sentry

    classDef person fill:#08427b,color:#fff,stroke:#052e56
    classDef container fill:#438dd5,color:#fff,stroke:#3079b6
    classDef ext fill:#999999,color:#fff,stroke:#6b6b6b
    classDef db fill:#438dd5,color:#fff,stroke:#3079b6

    class user,admin_user person
    class traefik_proxy,spa,fastapi_api container
    class pg db
    class smtp,sentry ext
```

---

## L3: FastAPI Backend

```mermaid
graph TD
    %% SCOPE: urn:c4:container:fastapi_api

    spa["React SPA<br/>[Container]"]

    subgraph fastapi_api["FastAPI Backend"]

        subgraph middleware["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/>[Component]<br/>Cross-origin request<br/>handling"]
            %% KIND: boundary
            auth_dep["Auth Dependencies<br/>[Component]<br/>JWT validation,<br/>OAuth2 password bearer"]
            %% KIND: data_access
            session_dep["DB Session Dependency<br/>[Component]<br/>Per-request database<br/>session via get_db"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes<br/>[Component]<br/>/login – access tokens,<br/>password recovery and reset"]
            %% KIND: router
            users_routes["Users Routes<br/>[Component]<br/>/users – CRUD, signup,<br/>profile, admin ops"]
            %% KIND: router
            items_routes["Items Routes<br/>[Component]<br/>/items – item CRUD<br/>per-user ownership"]
            %% KIND: router
            utils_routes["Utils Routes<br/>[Component]<br/>/utils – health check,<br/>test email"]
        end

        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Operations<br/>[Component]<br/>create_user, update_user,<br/>authenticate, create_item"]
            %% KIND: integration
            email_utils["Email Utilities<br/>[Component]<br/>SMTP sending, Jinja2<br/>template rendering"]
            %% KIND: service_layer
            security_mod["Security Module<br/>[Component]<br/>JWT creation, password<br/>hashing via Argon2 / Bcrypt"]
        end

        subgraph core["Core"]
            %% KIND: service_layer
            config_mod["Configuration<br/>[Component]<br/>Pydantic Settings,<br/>environment variables"]
            %% KIND: data_access
            db_engine["Database Engine<br/>[Component]<br/>SQLAlchemy engine and<br/>connection pool"]
            %% KIND: data_access
            models_layer["SQLModel Models<br/>[Component]<br/>User, Item table models<br/>and Pydantic schemas"]
        end

    end

    pg[("PostgreSQL<br/>[Database]")]
    smtp["SMTP Server<br/>[External]"]
    sentry["Sentry<br/>[External]"]

    spa -->|"API requests<br/>[HTTPS / JSON]"| cors_mw
    cors_mw --> auth_dep
    auth_dep --> session_dep
    session_dep --> login_routes
    session_dep --> users_routes
    session_dep --> items_routes
    session_dep --> utils_routes

    login_routes --> security_mod
    login_routes --> crud_layer
    users_routes --> crud_layer
    users_routes --> email_utils
    items_routes --> crud_layer
    utils_routes --> email_utils

    crud_layer --> models_layer
    models_layer --> db_engine
    security_mod --> config_mod
    db_engine --> config_mod
    db_engine -->|"SQL queries<br/>[psycopg]"| pg
    email_utils -->|"Sends emails<br/>[SMTP]"| smtp

    classDef component fill:#85bbf0,color:#000,stroke:#5a9bd5
    classDef ext fill:#999999,color:#fff,stroke:#6b6b6b
    classDef db fill:#438dd5,color:#fff,stroke:#3079b6
    classDef container fill:#438dd5,color:#fff,stroke:#3079b6

    class cors_mw,auth_dep,session_dep component
    class login_routes,users_routes,items_routes,utils_routes component
    class crud_layer,email_utils,security_mod component
    class config_mod,db_engine,models_layer component
    class pg db
    class smtp,sentry ext
    class spa container
```

---

## L3: React SPA

```mermaid
graph TD
    %% SCOPE: urn:c4:container:spa

    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            tanstack_router["TanStack Router<br/>[Component]<br/>File-based routing,<br/>auto code-splitting"]
            %% KIND: boundary
            auth_pages["Auth Pages<br/>[Component]<br/>Login, Signup, Recover<br/>Password, Reset Password"]
            %% KIND: boundary
            dashboard_page["Dashboard Page<br/>[Component]<br/>Welcome view with<br/>user information"]
            %% KIND: boundary
            items_page["Items Management<br/>[Component]<br/>Item CRUD with<br/>DataTable"]
            %% KIND: boundary
            admin_page["Admin Panel<br/>[Component]<br/>User management<br/>for superusers"]
            %% KIND: boundary
            settings_page["User Settings<br/>[Component]<br/>Profile, password,<br/>account deletion"]
        end

        subgraph presentation["Presentation"]
            %% KIND: boundary
            ui_components["UI Components<br/>[Component]<br/>shadcn / Radix primitives,<br/>Tailwind CSS 4"]
            %% KIND: boundary
            sidebar_component["Sidebar Navigation<br/>[Component]<br/>App navigation<br/>and layout"]
            %% KIND: service_layer
            theme_provider["Theme Provider<br/>[Component]<br/>Light and dark mode<br/>via next-themes"]
        end

        subgraph data_layer["Data Layer"]
            %% KIND: service_layer
            auth_hook["useAuth Hook<br/>[Component]<br/>Auth state, login,<br/>logout, signup"]
            %% KIND: integration
            api_client["OpenAPI Client<br/>[Component]<br/>Generated TypeScript SDK,<br/>Axios-based HTTP"]
            %% KIND: service_layer
            query_client["TanStack Query<br/>[Component]<br/>Server state caching,<br/>401 / 403 handling"]
        end

    end

    fastapi_api["FastAPI Backend<br/>[Container]"]

    tanstack_router --> auth_pages
    tanstack_router --> dashboard_page
    tanstack_router --> items_page
    tanstack_router --> admin_page
    tanstack_router --> settings_page

    auth_pages --> ui_components
    dashboard_page --> ui_components
    items_page --> ui_components
    admin_page --> ui_components
    settings_page --> ui_components

    items_page --> sidebar_component
    admin_page --> sidebar_component
    settings_page --> sidebar_component

    auth_pages --> auth_hook
    dashboard_page --> auth_hook
    settings_page --> auth_hook

    items_page --> query_client
    admin_page --> query_client

    auth_hook --> api_client
    query_client --> api_client

    api_client -->|"REST API calls<br/>[HTTPS / JSON]"| fastapi_api

    classDef component fill:#85bbf0,color:#000,stroke:#5a9bd5
    classDef container fill:#438dd5,color:#fff,stroke:#3079b6

    class tanstack_router,auth_pages,dashboard_page,items_page,admin_page,settings_page component
    class ui_components,sidebar_component,theme_provider component
    class auth_hook,api_client,query_client component
    class fastapi_api container
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users             CONTAINS [user, admin_user]
platform_boundary CONTAINS [platform, traefik_proxy, spa, fastapi_api, pg]
external          CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment: FastAPI Backend ─────────────────
fastapi_api CONTAINS [middleware, routes, services, core]
  middleware  CONTAINS [cors_mw, auth_dep, session_dep]
  routes      CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services    CONTAINS [crud_layer, email_utils, security_mod]
  core        CONTAINS [config_mod, db_engine, models_layer]

%% ── L2→L3 internal containment: React SPA ───────────────────────
spa CONTAINS [routing, presentation, data_layer]
  routing       CONTAINS [tanstack_router, auth_pages, dashboard_page, items_page, admin_page, settings_page]
  presentation  CONTAINS [ui_components, sidebar_component, theme_provider]
  data_layer    CONTAINS [auth_hook, api_client, query_client]
```
