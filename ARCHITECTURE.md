# C4 Architecture Model — Full Stack FastAPI Platform

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi-platform
graph TB

    subgraph users["Users"]
        user["👤 User\n(Regular application user)"]
        admin_user["👤 Administrator\n(Superuser managing\nusers and system)"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["🖥️ Full Stack FastAPI Platform\nWeb application for managing\nusers and items with\nJWT authentication"]
    end

    subgraph external["External Systems"]
        smtp["📧 SMTP Server\n(Email delivery service)"]
        sentry["📊 Sentry\n(Error tracking &\nperformance monitoring)"]
    end

    user -->|"Uses web application\n[HTTPS]"| platform
    admin_user -->|"Manages users & items\n[HTTPS]"| platform
    platform -->|"Sends transactional emails\n[SMTP/TLS]"| smtp
    platform -->|"Reports errors & traces\n[HTTPS]"| sentry
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi-platform
graph TB

    subgraph users["Users"]
        user["👤 User"]
        admin_user["👤 Administrator"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["🔀 Traefik\nReverse Proxy / Load Balancer\n[Docker · Traefik 3.6]"]
        spa["⚛️ React SPA\nSingle-Page Application\n[Vite · React 19 · TypeScript\nServed via Nginx]"]
        fastapi_backend["🐍 FastAPI Backend\nREST API Server\n[Python 3.10 · FastAPI\nUvicorn · 4 workers]"]
        pg["🐘 PostgreSQL\nPrimary Database\n[PostgreSQL 18]"]
        adminer["🔧 Adminer\nDatabase Admin UI\n[Docker Container]"]
    end

    subgraph external["External Systems"]
        smtp["📧 SMTP Server"]
        sentry["📊 Sentry"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"Routes dashboard.*\n[HTTP :80]"| spa
    traefik -->|"Routes api.*\n[HTTP :8000]"| fastapi_backend
    traefik -->|"Routes adminer.*\n[HTTP :8080]"| adminer
    spa -->|"REST API calls\n[JSON over HTTPS]"| fastapi_backend
    fastapi_backend -->|"SQL queries\n[psycopg · TCP :5432]"| pg
    fastapi_backend -->|"Sends emails\n[SMTP/TLS]"| smtp
    fastapi_backend -->|"Error reports\n[HTTPS]"| sentry
    adminer -->|"Database admin\n[TCP :5432]"| pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi-backend
graph TB

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph middleware["Middleware & Dependencies"]
            %% KIND: boundary
            cors_mw["CORS Middleware\n(Cross-origin request handling\nvia Starlette CORSMiddleware)"]
            %% KIND: boundary
            auth_dep["Auth Dependencies\n(OAuth2 bearer token extraction,\nJWT validation, session injection,\nsuperuser guard)"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes\n/login/access-token\n/password-recovery/*\n/reset-password/"]
            %% KIND: router
            user_routes["User Routes\n/users/ · /users/me\n/users/signup\n/users/{id}"]
            %% KIND: router
            item_routes["Item Routes\n/items/ CRUD\n(list, get, create,\nupdate, delete)"]
            %% KIND: router
            util_routes["Utility Routes\n/utils/health-check\n/utils/test-email"]
            %% KIND: router
            private_routes["Private Routes\n/private/users/\n(local dev only)"]
        end

        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Module\n(create/read/update users,\nauthenticate, create items)"]
            %% KIND: integration
            email_utils["Email Utilities\n(Jinja2 template rendering,\nSMTP sending, password\nreset token generation)"]
        end

        subgraph core["Core"]
            %% KIND: service_layer
            security_core["Security Module\n(JWT creation via PyJWT,\npassword hashing via\nArgon2 + Bcrypt)"]
            %% KIND: service_layer
            config_core["Configuration\n(Pydantic Settings,\nenv variable loading,\nSQLAlchemy URI building)"]
            %% KIND: data_access
            db_engine["Database Engine\n(SQLAlchemy create_engine,\nsession factory,\nDB initialization)"]
        end

        subgraph data["Data Layer"]
            %% KIND: storage
            models_layer["SQLModel Models\n(User, Item tables;\nrequest/response schemas;\nToken, Message DTOs)"]
        end

    end

    pg["🐘 PostgreSQL"]
    smtp["📧 SMTP Server"]

    cors_mw --> auth_dep
    auth_dep --> login_routes
    auth_dep --> user_routes
    auth_dep --> item_routes
    auth_dep --> util_routes
    auth_dep --> private_routes

    login_routes --> crud_layer
    login_routes --> email_utils
    login_routes --> security_core
    user_routes --> crud_layer
    user_routes --> email_utils
    user_routes --> security_core
    item_routes --> models_layer
    util_routes --> email_utils

    crud_layer --> models_layer
    crud_layer --> security_core
    crud_layer --> db_engine
    email_utils --> security_core
    email_utils --> config_core

    auth_dep --> security_core
    auth_dep --> db_engine
    auth_dep --> models_layer
    db_engine --> config_core
    security_core --> config_core
    db_engine --> models_layer

    db_engine -->|"SQL"| pg
    email_utils -->|"SMTP"| smtp
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            tanstack_router["TanStack Router\n(File-based routing,\nauth guards via\nbeforeLoad redirect)"]
        end

        subgraph state_mgmt["State & Data Fetching"]
            %% KIND: service_layer
            tanstack_query["TanStack Query\n(QueryClient, caching,\nmutation management,\nauto-refetch)"]
            %% KIND: integration
            api_client["API Client\n(Auto-generated from OpenAPI\nvia @hey-api/openapi-ts;\nAxios HTTP transport)"]
            %% KIND: service_layer
            auth_hook["Auth Hook\n(useAuth: login/logout\nmutations, signup,\ncurrent user query)"]
        end

        subgraph views["Views / Pages"]
            %% KIND: boundary
            login_views["Auth Pages\n(Login, Signup,\nRecover Password,\nReset Password)"]
            %% KIND: boundary
            admin_views["Admin Dashboard\n(User list table,\nAdd/Edit/Delete User\ndialogs)"]
            %% KIND: boundary
            item_views["Items Management\n(Item list table,\nAdd/Edit/Delete Item\ndialogs)"]
            %% KIND: boundary
            settings_views["User Settings\n(User info, Change\npassword, Delete account)"]
        end

        subgraph presentation["Presentation"]
            %% KIND: boundary
            sidebar_nav["Sidebar Navigation\n(AppSidebar, main nav,\nuser menu)"]
            %% KIND: boundary
            common_components["Common Components\n(DataTable, Footer, Logo,\nAuthLayout, ErrorComponent)"]
            %% KIND: boundary
            ui_lib["shadcn/ui Library\n(Button, Dialog, Form,\nInput, Table, Tabs,\nTooltip, etc.)"]
            %% KIND: service_layer
            theme_provider["Theme Provider\n(Dark/light mode\nvia next-themes)"]
        end

    end

    fastapi_backend["🐍 FastAPI Backend"]

    tanstack_router --> login_views
    tanstack_router --> admin_views
    tanstack_router --> item_views
    tanstack_router --> settings_views

    login_views --> auth_hook
    admin_views --> tanstack_query
    admin_views --> api_client
    item_views --> tanstack_query
    item_views --> api_client
    settings_views --> tanstack_query
    settings_views --> api_client
    auth_hook --> tanstack_query
    auth_hook --> api_client

    login_views --> ui_lib
    admin_views --> ui_lib
    admin_views --> common_components
    item_views --> ui_lib
    item_views --> common_components
    settings_views --> ui_lib

    tanstack_router --> sidebar_nav
    sidebar_nav --> ui_lib
    common_components --> ui_lib
    ui_lib --> theme_provider

    api_client -->|"REST / JSON"| fastapi_backend
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment ──────────────────────────────────
fastapi_backend    CONTAINS [middleware, routes, services, core, data]
  middleware       CONTAINS [cors_mw, auth_dep]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes, private_routes]
  services         CONTAINS [crud_layer, email_utils]
  core             CONTAINS [security_core, config_core, db_engine]
  data             CONTAINS [models_layer]

spa                CONTAINS [routing, state_mgmt, views, presentation]
  routing          CONTAINS [tanstack_router]
  state_mgmt       CONTAINS [tanstack_query, api_client, auth_hook]
  views            CONTAINS [login_views, admin_views, item_views, settings_views]
  presentation     CONTAINS [sidebar_nav, common_components, ui_lib, theme_provider]
```

---

## Stable ID Reference

| ID | Name | Level(s) | Kind |
|----|------|----------|------|
| `user` | User | L1, L2 | actor |
| `admin_user` | Administrator | L1, L2 | actor |
| `platform_boundary` | Full Stack FastAPI Platform | L1, L2 | boundary |
| `traefik` | Traefik Reverse Proxy | L2 | container |
| `spa` | React SPA | L2, L3-scope | container |
| `fastapi_backend` | FastAPI Backend | L2, L3-scope | container |
| `pg` | PostgreSQL | L2, L3-backend | storage |
| `adminer` | Adminer DB Admin | L2 | container |
| `smtp` | SMTP Server | L1, L2, L3-backend | integration |
| `sentry` | Sentry | L1, L2 | integration |
| `cors_mw` | CORS Middleware | L3-backend | boundary |
| `auth_dep` | Auth Dependencies | L3-backend | boundary |
| `login_routes` | Login Routes | L3-backend | router |
| `user_routes` | User Routes | L3-backend | router |
| `item_routes` | Item Routes | L3-backend | router |
| `util_routes` | Utility Routes | L3-backend | router |
| `private_routes` | Private Routes | L3-backend | router |
| `crud_layer` | CRUD Module | L3-backend | service_layer |
| `email_utils` | Email Utilities | L3-backend | integration |
| `security_core` | Security Module | L3-backend | service_layer |
| `config_core` | Configuration | L3-backend | service_layer |
| `db_engine` | Database Engine | L3-backend | data_access |
| `models_layer` | SQLModel Models | L3-backend | storage |
| `tanstack_router` | TanStack Router | L3-spa | router |
| `tanstack_query` | TanStack Query | L3-spa | service_layer |
| `api_client` | API Client | L3-spa | integration |
| `auth_hook` | Auth Hook | L3-spa | service_layer |
| `login_views` | Auth Pages | L3-spa | boundary |
| `admin_views` | Admin Dashboard | L3-spa | boundary |
| `item_views` | Items Management | L3-spa | boundary |
| `settings_views` | User Settings | L3-spa | boundary |
| `sidebar_nav` | Sidebar Navigation | L3-spa | boundary |
| `common_components` | Common Components | L3-spa | boundary |
| `ui_lib` | shadcn/ui Library | L3-spa | boundary |
| `theme_provider` | Theme Provider | L3-spa | service_layer |
