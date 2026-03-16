# C4 Architecture Model – Full Stack FastAPI Project

> Auto-generated C4 model describing the system at three levels of abstraction.
> Each successive diagram zooms into an element from the level above.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:platform
graph TB

    subgraph users["Users"]
        user(["End User\nManages items and own profile"])
        admin_user(["Admin User\nSuperuser – manages all users"])
    end

    subgraph platform_boundary["Full Stack FastAPI Project"]
        platform["Full Stack FastAPI Project\n[Software System]\nWeb application for user and\nitem management with JWT auth"]
    end

    subgraph external["External Systems"]
        smtp_server["SMTP Server\n[External]\nEmail delivery service"]
        sentry_ext["Sentry\n[External]\nError monitoring and APM"]
    end

    user -- "Uses web app [HTTPS]" --> platform
    admin_user -- "Administers system [HTTPS]" --> platform
    platform -- "Sends emails [SMTP/TLS]" --> smtp_server
    platform -- "Reports errors [HTTPS]" --> sentry_ext

    classDef person fill:#08427b,color:#fff,stroke:#073b6f
    classDef system fill:#1168bd,color:#fff,stroke:#0b4884
    classDef external_sys fill:#999999,color:#fff,stroke:#6b6b6b

    class user,admin_user person
    class platform system
    class smtp_server,sentry_ext external_sys
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:platform
graph TB

    subgraph users["Users"]
        user(["End User"])
        admin_user(["Admin User"])
    end

    subgraph platform_boundary["Full Stack FastAPI Project"]
        traefik["Traefik Reverse Proxy\n[Container: traefik:3.6]\nHTTPS termination, routing,\nLet's Encrypt TLS"]
        spa["React SPA\n[Container: TypeScript, React 19, Vite]\nServed by Nginx – single-page app\nwith TanStack Router and Query"]
        fastapi_backend["FastAPI Backend\n[Container: Python, FastAPI, SQLModel]\nREST API with JWT auth,\nserves /api/v1/*"]
        pg[("PostgreSQL\n[Container: postgres:18]\nPrimary relational data store\nfor users and items")]
        adminer["Adminer\n[Container: adminer]\nDatabase admin UI"]
    end

    subgraph external["External Systems"]
        smtp_server["SMTP Server\n[External]"]
        sentry_ext["Sentry\n[External]"]
    end

    user -- "Browses dashboard [HTTPS]" --> traefik
    admin_user -- "Administers [HTTPS]" --> traefik
    traefik -- "Routes dashboard.*" --> spa
    traefik -- "Routes api.*" --> fastapi_backend
    traefik -- "Routes adminer.*" --> adminer
    spa -- "API calls [JSON/HTTPS]" --> fastapi_backend
    fastapi_backend -- "Reads/writes [SQL, psycopg]" --> pg
    fastapi_backend -- "Sends emails [SMTP/TLS]" --> smtp_server
    fastapi_backend -- "Reports errors [HTTPS]" --> sentry_ext
    adminer -- "Queries [SQL]" --> pg

    classDef person fill:#08427b,color:#fff,stroke:#073b6f
    classDef container fill:#1168bd,color:#fff,stroke:#0b4884
    classDef database fill:#2b78e4,color:#fff,stroke:#1a5bb5
    classDef external_sys fill:#999999,color:#fff,stroke:#6b6b6b
    classDef infra fill:#5b9bd5,color:#fff,stroke:#3a7ab8

    class user,admin_user person
    class spa,fastapi_backend container
    class pg database
    class traefik,adminer infra
    class smtp_server,sentry_ext external_sys
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
graph TB

    spa["React SPA"]

    subgraph fastapi_backend["FastAPI Backend"]

        %% KIND: router
        subgraph routes["API Routes (/api/v1)"]
            %% KIND: router
            login_routes["Login Routes\n/login/access-token\n/password-recovery/*\n/reset-password"]
            %% KIND: router
            user_routes["User Routes\n/users, /users/me\n/users/signup, /users/{id}"]
            %% KIND: router
            item_routes["Item Routes\n/items, /items/{id}"]
            %% KIND: router
            utils_routes["Utils Routes\n/utils/health-check\n/utils/test-email"]
        end

        %% KIND: boundary
        subgraph middleware["Middleware and Dependencies"]
            %% KIND: service_layer
            cors_mw["CORS Middleware\nStarlette CORSMiddleware\nAllows configured origins"]
            %% KIND: service_layer
            auth_dep["Auth Dependencies\nOAuth2PasswordBearer\nJWT token validation\nget_current_user / superuser"]
            %% KIND: data_access
            db_dep["DB Session Dependency\nSQLModel Session via get_db\nYields per-request sessions"]
        end

        %% KIND: service_layer
        crud_layer["CRUD Layer\ncreate_user, update_user\nauthenticate, create_item\nget_user_by_email"]

        %% KIND: data_access
        models_layer["Models Layer\nSQLModel ORM\nUser, Item tables\nPydantic request/response schemas"]

        %% KIND: boundary
        subgraph core["Core"]
            %% KIND: service_layer
            security["Security Module\nJWT create/verify (HS256)\nArgon2 + Bcrypt password hashing"]
            %% KIND: storage
            config["Configuration\nPydantic BaseSettings\nLoads from .env"]
            %% KIND: data_access
            db_engine["Database Engine\nSQLAlchemy create_engine\nPostgreSQL + psycopg"]
        end

        %% KIND: integration
        email_utils["Email Utilities\nSMTP email sending\nJinja2 HTML templates\nPassword-reset token generation"]

    end

    pg[("PostgreSQL")]
    smtp_server["SMTP Server"]

    spa -- "HTTP requests" --> routes
    routes --> auth_dep
    routes --> db_dep
    login_routes --> crud_layer
    login_routes --> security
    login_routes --> email_utils
    user_routes --> crud_layer
    item_routes --> crud_layer
    utils_routes --> email_utils
    auth_dep --> security
    auth_dep --> models_layer
    crud_layer --> models_layer
    crud_layer --> security
    models_layer --> db_engine
    db_engine --> pg
    db_dep --> db_engine
    email_utils --> smtp_server
    config -. "provides settings" .-> security
    config -. "provides settings" .-> db_engine
    config -. "provides settings" .-> email_utils

    classDef component fill:#4b8bbe,color:#fff,stroke:#2d6a9f
    classDef boundary_style fill:none,stroke:#888,stroke-dasharray:5 5
    classDef external_sys fill:#999999,color:#fff,stroke:#6b6b6b
    classDef database fill:#2b78e4,color:#fff,stroke:#1a5bb5

    class login_routes,user_routes,item_routes,utils_routes component
    class cors_mw,auth_dep,db_dep component
    class crud_layer,models_layer component
    class security,config,db_engine component
    class email_utils component
    class pg database
    class smtp_server,spa external_sys
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

    user(["End User"])

    subgraph spa["React SPA"]

        %% KIND: router
        routing["TanStack Router\nFile-based routing\nAuth guards in beforeLoad"]

        %% KIND: boundary
        subgraph pages["Pages and Features"]
            %% KIND: service_layer
            auth_pages["Auth Pages\nLogin, Signup\nRecover Password, Reset Password"]
            %% KIND: service_layer
            dashboard_page["Dashboard Page\nWelcome view\nCurrent user greeting"]
            %% KIND: service_layer
            items_feature["Items Feature\nCRUD UI with DataTable\nAdd, Edit, Delete items"]
            %% KIND: service_layer
            admin_feature["Admin Feature\nUser management (superuser)\nAdd, Edit, Delete users"]
            %% KIND: service_layer
            settings_feature["Settings Feature\nProfile info, Change password\nDelete account"]
        end

        %% KIND: boundary
        subgraph hooks["Hooks"]
            %% KIND: service_layer
            auth_hook["useAuth Hook\nLogin, logout, signup mutations\nCurrent user query"]
            %% KIND: service_layer
            toast_hook["useCustomToast Hook\nSonner toast notifications"]
        end

        %% KIND: service_layer
        query_layer["TanStack Query\nServer state management\nQuery caching and invalidation\nMutation error handling"]

        %% KIND: integration
        api_client["Generated API Client\n@hey-api/openapi-ts\nAxios HTTP layer\nItemsService, UsersService,\nLoginService"]

        %% KIND: boundary
        subgraph ui["UI Component Library"]
            %% KIND: service_layer
            ui_primitives["Radix / shadcn Primitives\nButton, Dialog, Form, Table\nInput, Card, Tabs, Select"]
            %% KIND: service_layer
            common_components["Common Components\nDataTable, AppSidebar\nAuthLayout, ErrorBoundary\nLogo, Footer, ThemeProvider"]
        end

    end

    fastapi_backend["FastAPI Backend"]

    user -- "Browses" --> routing
    routing --> pages
    auth_pages --> auth_hook
    items_feature --> query_layer
    admin_feature --> query_layer
    settings_feature --> query_layer
    settings_feature --> auth_hook
    dashboard_page --> query_layer
    auth_hook --> api_client
    query_layer --> api_client
    pages --> ui
    api_client -- "REST API [JSON/HTTPS]" --> fastapi_backend

    classDef component fill:#4b8bbe,color:#fff,stroke:#2d6a9f
    classDef external_sys fill:#999999,color:#fff,stroke:#6b6b6b
    classDef person fill:#08427b,color:#fff,stroke:#073b6f

    class routing,auth_pages,dashboard_page,items_feature,admin_feature,settings_feature component
    class auth_hook,toast_hook component
    class query_layer,api_client component
    class ui_primitives,common_components component
    class fastapi_backend external_sys
    class user person
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP – every subgraph and its direct children
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [platform]
external           CONTAINS [smtp_server, sentry_ext]

%% ── L2 container-level (platform_boundary expands) ──────────────
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
users              CONTAINS [user, admin_user]
external           CONTAINS [smtp_server, sentry_ext]

%% ── L3: fastapi_backend internals ───────────────────────────────
fastapi_backend CONTAINS [routes, middleware, crud_layer, models_layer, core, email_utils]
  routes      CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  middleware  CONTAINS [cors_mw, auth_dep, db_dep]
  core        CONTAINS [security, config, db_engine]

%% ── L3: spa internals ──────────────────────────────────────────
spa CONTAINS [routing, pages, hooks, query_layer, api_client, ui]
  pages CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
  hooks CONTAINS [auth_hook, toast_hook]
  ui    CONTAINS [ui_primitives, common_components]
```
