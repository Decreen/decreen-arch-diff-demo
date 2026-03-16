# C4 Architecture Model – Full Stack FastAPI Project

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for every container with internal structure.

---

## L1: System Context

```mermaid
graph TB
    %% SCOPE: urn:c4:system:full-stack-fastapi

    subgraph users["Users"]
        user["👤 User\n<i>End user who manages\ntheir own items</i>"]
        admin_user["👤 Admin\n<i>Superuser who manages\nall users and data</i>"]
    end

    subgraph system_boundary["Full Stack FastAPI Project"]
        fastapi_system["Full Stack FastAPI Project\n<i>[Software System]</i>\nWeb application for user\nand item management"]
    end

    subgraph external["External Systems"]
        sentry["Sentry\n<i>[External Service]</i>\nError monitoring & tracing"]
        smtp_server["SMTP Server\n<i>[External Service]</i>\nTransactional email delivery"]
    end

    user -->|"Browses dashboard,\nmanages own items"| fastapi_system
    admin_user -->|"Manages users,\nconfigures system"| fastapi_system
    fastapi_system -->|"Reports errors\n[HTTPS]"| sentry
    fastapi_system -->|"Sends emails\n[SMTP/TLS]"| smtp_server
```

---

## L2: Container

```mermaid
graph TB
    %% SCOPE: urn:c4:system:full-stack-fastapi

    subgraph users["Users"]
        user["👤 User"]
        admin_user["👤 Admin"]
    end

    subgraph system_boundary["Full Stack FastAPI Project"]
        traefik["Traefik\n<i>[Container: Traefik 3.x]</i>\nReverse proxy, TLS\ntermination, routing"]

        spa["React SPA\n<i>[Container: React 19 / Nginx]</i>\nSingle-page app serving\nthe web dashboard"]

        fastapi_backend["FastAPI Backend\n<i>[Container: Python / FastAPI]</i>\nREST API – authentication,\nbusiness logic, data access"]

        pg["PostgreSQL\n<i>[Container: PostgreSQL 18]</i>\nStores users, items,\nand application data"]
    end

    subgraph external["External Systems"]
        sentry["Sentry\n<i>[External Service]</i>\nError monitoring"]
        smtp_server["SMTP Server\n<i>[External Service]</i>\nEmail delivery"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"Routes dashboard.*"| spa
    traefik -->|"Routes api.*"| fastapi_backend
    spa -->|"REST API calls\n[JSON/HTTPS]"| fastapi_backend
    fastapi_backend -->|"Reads & writes\n[SQL/TCP]"| pg
    fastapi_backend -->|"Reports errors\n[HTTPS]"| sentry
    fastapi_backend -->|"Sends emails\n[SMTP]"| smtp_server
```

---

## L3: FastAPI Backend

```mermaid
graph TB
    %% SCOPE: urn:c4:container:fastapi-backend

    spa["React SPA"]
    pg["PostgreSQL"]
    sentry["Sentry"]
    smtp_server["SMTP Server"]

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph middleware["Middleware & Auth"]
            %% KIND: boundary
            cors_mw["CORS Middleware\n<i>[Starlette Middleware]</i>\nEnforces allowed origins,\nmethods, and headers"]
            %% KIND: boundary
            auth_deps["Auth Dependencies\n<i>[FastAPI Depends]</i>\nOAuth2 bearer extraction,\nJWT validation, user resolution,\nsuperuser gate"]
        end

        subgraph routes["API Routes  /api/v1"]
            %% KIND: router
            login_routes["Login Routes\n<i>[/login]</i>\nAccess-token issuance,\npassword recovery & reset"]
            %% KIND: router
            user_routes["User Routes\n<i>[/users]</i>\nUser CRUD, self-registration,\nprofile & password update"]
            %% KIND: router
            item_routes["Item Routes\n<i>[/items]</i>\nItem CRUD – owned\nby authenticated user"]
            %% KIND: router
            utils_routes["Utils Routes\n<i>[/utils]</i>\nHealth check,\ntest email (superuser)"]
        end

        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Module\n<i>[crud.py]</i>\nCreate / read / update / delete\nlogic for User & Item"]
            %% KIND: integration
            email_utils["Email Utilities\n<i>[utils.py]</i>\nJinja2 template rendering,\nSMTP dispatch, token helpers"]
        end

        subgraph core["Core"]
            %% KIND: data_access
            models_layer["Models\n<i>[SQLModel / SQLAlchemy]</i>\nUser, Item tables and\nPydantic request/response schemas"]
            %% KIND: service_layer
            security_mod["Security\n<i>[core/security.py]</i>\nJWT creation (HS256),\nArgon2 / bcrypt password hashing"]
            %% KIND: service_layer
            config_mod["Configuration\n<i>[core/config.py]</i>\nPydantic Settings – DB, SMTP,\nSentry DSN, CORS origins"]
        end

    end

    spa -->|"HTTP requests"| cors_mw
    cors_mw -.->|"Passes to router"| routes

    auth_deps -.->|"Secures"| user_routes
    auth_deps -.->|"Secures"| item_routes
    auth_deps -.->|"Secures"| utils_routes

    login_routes --> crud_layer
    login_routes --> security_mod
    login_routes --> email_utils
    user_routes --> crud_layer
    item_routes --> crud_layer
    utils_routes --> email_utils

    crud_layer --> models_layer
    crud_layer --> security_mod
    models_layer -->|"SQL queries\n[psycopg]"| pg
    email_utils -->|"SMTP"| smtp_server
    config_mod -.->|"Initialises\nSentry SDK"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
    %% SCOPE: urn:c4:container:spa

    fastapi_backend["FastAPI Backend"]
    user["👤 User"]
    admin_user["👤 Admin"]

    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            router_mod["TanStack Router\n<i>[File-based routing]</i>\nDeclares all page routes,\nlayout nesting, code splitting"]
        end

        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            query_client["React Query Client\n<i>[TanStack Query 5]</i>\nServer-state cache,\nauto-refetch, error handling"]
            %% KIND: service_layer
            auth_hook["useAuth Hook\n<i>[hooks/useAuth.ts]</i>\nLogin / signup / logout,\nJWT in localStorage"]
        end

        subgraph api_layer["API Client"]
            %% KIND: integration
            api_client["OpenAPI SDK\n<i>[@hey-api/openapi-ts]</i>\nGenerated TypeScript client:\nItemsService, UsersService,\nLoginService, UtilsService"]
        end

        subgraph features["Feature Pages"]
            %% KIND: boundary
            dashboard_page["Dashboard\n<i>[routes/_layout/index]</i>\nWelcome screen,\nuser greeting"]
            %% KIND: boundary
            items_feature["Items Management\n<i>[routes/_layout/items]</i>\nItem CRUD table\nwith add/edit/delete"]
            %% KIND: boundary
            admin_feature["Admin Panel\n<i>[routes/_layout/admin]</i>\nUser management table\n(superuser only)"]
            %% KIND: boundary
            settings_feature["User Settings\n<i>[routes/_layout/settings]</i>\nProfile, password change,\naccount deletion"]
            %% KIND: boundary
            auth_pages["Auth Pages\n<i>[routes/login, signup, ...]</i>\nLogin, signup,\npassword recovery & reset"]
        end

        subgraph ui_layer["UI Components"]
            %% KIND: boundary
            ui_components["shadcn / Radix Primitives\n<i>[components/ui/*]</i>\nButton, Dialog, Table, Form,\nSidebar, Tabs, etc."]
            %% KIND: service_layer
            theme_provider["Theme Provider\n<i>[next-themes]</i>\nDark / light mode toggle"]
        end

    end

    user -->|"Interacts"| router_mod
    admin_user -->|"Interacts"| router_mod

    router_mod --> features
    features --> state_mgmt
    features --> ui_layer

    auth_hook --> api_client
    query_client --> api_client
    api_client -->|"REST / JSON\n[HTTPS]"| fastapi_backend
```

---

## Containment Map

The containment map below lists every subgraph from every diagram and
its direct children. The parser uses this to build the full
parent → child hierarchy tree for drill-down navigation.

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users            CONTAINS [user, admin_user]
system_boundary  CONTAINS [fastapi_system]
external         CONTAINS [sentry, smtp_server]

%% ── L2 container-level ─────────────────────────────────────────────
system_boundary  CONTAINS [traefik, spa, fastapi_backend, pg]

%% ── L3: FastAPI Backend internal containment ───────────────────────
fastapi_backend  CONTAINS [middleware, routes, services, core]
  middleware     CONTAINS [cors_mw, auth_deps]
  routes         CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services       CONTAINS [crud_layer, email_utils]
  core           CONTAINS [models_layer, security_mod, config_mod]

%% ── L3: React SPA internal containment ─────────────────────────────
spa              CONTAINS [routing, state_mgmt, api_layer, features, ui_layer]
  routing        CONTAINS [router_mod]
  state_mgmt     CONTAINS [query_client, auth_hook]
  api_layer      CONTAINS [api_client]
  features       CONTAINS [dashboard_page, items_feature, admin_feature, settings_feature, auth_pages]
  ui_layer       CONTAINS [ui_components, theme_provider]
```
