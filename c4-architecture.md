# C4 Architecture Model — Full Stack FastAPI Project

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI Project.

---

## L1: System Context

```mermaid
graph TB
    %% SCOPE: urn:c4:system:fastapi_project

    subgraph users["Users"]
        end_user["End User<br/><i>[Person]</i><br/>Manages personal items<br/>via the web application"]
        admin_user["Administrator<br/><i>[Person]</i><br/>Superuser who manages<br/>users and configuration"]
    end

    subgraph system_boundary["Full Stack FastAPI Project"]
        fastapi_project["Full Stack FastAPI Project<br/><i>[Software System]</i><br/>Web application providing user<br/>registration, authentication, and<br/>item management with email notifications"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Service<br/><i>[External System]</i><br/>Delivers transactional emails<br/>(password reset, new account)"]
        sentry["Sentry<br/><i>[External System]</i><br/>Error monitoring<br/>and performance tracking"]
    end

    end_user -->|"Uses web application<br/>[HTTPS]"| fastapi_project
    admin_user -->|"Manages users and system<br/>[HTTPS]"| fastapi_project
    fastapi_project -->|"Sends transactional emails<br/>[SMTP/TLS]"| smtp
    fastapi_project -->|"Reports errors<br/>[HTTPS]"| sentry
```

---

## L2: Container

```mermaid
graph TB
    %% SCOPE: urn:c4:system:fastapi_project

    subgraph users["Users"]
        end_user["End User<br/><i>[Person]</i>"]
        admin_user["Administrator<br/><i>[Person]</i>"]
    end

    subgraph system_boundary["Full Stack FastAPI Project"]
        traefik["Traefik<br/><i>[Container: Traefik 3.6]</i><br/>Reverse proxy, TLS termination,<br/>host-based routing"]
        react_spa["React SPA<br/><i>[Container: React 19 / Vite / Nginx]</i><br/>Single-page application serving<br/>dashboard, items, settings, admin"]
        fastapi_backend["FastAPI Backend<br/><i>[Container: Python / FastAPI / Uvicorn]</i><br/>REST API with JWT auth,<br/>CRUD operations, email services"]
        pg["PostgreSQL<br/><i>[Container: PostgreSQL 18]</i><br/>Stores users, items,<br/>and application data"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Service<br/><i>[External System]</i><br/>Email delivery"]
        sentry["Sentry<br/><i>[External System]</i><br/>Error monitoring"]
    end

    end_user -->|"Browses application<br/>[HTTPS]"| traefik
    admin_user -->|"Manages system<br/>[HTTPS]"| traefik
    traefik -->|"Routes dashboard.*<br/>[HTTP :80]"| react_spa
    traefik -->|"Routes api.*<br/>[HTTP :8000]"| fastapi_backend
    react_spa -->|"API calls /api/v1/*<br/>[HTTP/JSON]"| fastapi_backend
    fastapi_backend -->|"Reads/writes data<br/>[SQL via psycopg]"| pg
    fastapi_backend -->|"Sends emails<br/>[SMTP/TLS]"| smtp
    fastapi_backend -->|"Reports errors<br/>[HTTPS]"| sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TB
    %% SCOPE: urn:c4:container:fastapi_backend

    react_spa["React SPA<br/><i>[Container]</i>"]
    traefik["Traefik<br/><i>[Container]</i>"]
    pg["PostgreSQL<br/><i>[Container]</i>"]
    smtp["SMTP Service<br/><i>[External]</i>"]

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph middleware["Middleware & Dependencies"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/><i>[Component: Starlette CORSMiddleware]</i><br/>Enforces allowed origins,<br/>methods, and headers"]
            %% KIND: boundary
            auth_deps["Auth Dependencies<br/><i>[Component: FastAPI Depends]</i><br/>OAuth2 bearer token extraction,<br/>JWT decode, user loading"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes<br/><i>[Component: /login/*]</i><br/>access-token, test-token,<br/>password recovery/reset"]
            %% KIND: router
            user_routes["User Routes<br/><i>[Component: /users/*]</i><br/>CRUD, signup, profile,<br/>password change"]
            %% KIND: router
            item_routes["Item Routes<br/><i>[Component: /items/*]</i><br/>List, create, read,<br/>update, delete items"]
            %% KIND: router
            util_routes["Utility Routes<br/><i>[Component: /utils/*]</i><br/>Health check,<br/>test email"]
        end

        subgraph services["Service Layer"]
            %% KIND: data_access
            crud_layer["CRUD Layer<br/><i>[Component: app/crud.py]</i><br/>create_user, update_user,<br/>authenticate, create_item"]
            %% KIND: integration
            email_utils["Email Utilities<br/><i>[Component: app/utils.py]</i><br/>Template rendering, SMTP sending,<br/>token generation/verification"]
        end

        subgraph core["Core"]
            %% KIND: service_layer
            security_core["Security<br/><i>[Component: app/core/security.py]</i><br/>JWT creation (HS256),<br/>Argon2/Bcrypt password hashing"]
            %% KIND: service_layer
            config_core["Configuration<br/><i>[Component: app/core/config.py]</i><br/>Pydantic Settings,<br/>env-based configuration"]
        end

        subgraph data["Data Access"]
            %% KIND: data_access
            models_layer["SQLModel Models<br/><i>[Component: app/models.py]</i><br/>User, Item tables<br/>and Pydantic schemas"]
            %% KIND: storage
            db_engine["DB Engine<br/><i>[Component: app/core/db.py]</i><br/>SQLAlchemy engine,<br/>session management, init_db"]
        end

    end

    traefik -->|"HTTP requests"| cors_mw
    react_spa -->|"REST API calls"| cors_mw
    cors_mw --> login_routes
    cors_mw --> user_routes
    cors_mw --> item_routes
    cors_mw --> util_routes

    login_routes --> auth_deps
    user_routes --> auth_deps
    item_routes --> auth_deps
    auth_deps --> security_core
    auth_deps --> db_engine

    login_routes --> crud_layer
    login_routes --> security_core
    login_routes --> email_utils
    user_routes --> crud_layer
    item_routes --> crud_layer
    util_routes --> email_utils

    crud_layer --> models_layer
    crud_layer --> security_core
    db_engine --> config_core
    security_core --> config_core
    email_utils --> security_core
    email_utils --> config_core
    models_layer --> db_engine

    db_engine -->|"SQL via psycopg"| pg
    email_utils -->|"SMTP/TLS"| smtp
```

---

## L3: React SPA

```mermaid
graph TB
    %% SCOPE: urn:c4:container:react_spa

    fastapi_backend["FastAPI Backend<br/><i>[Container]</i>"]

    subgraph react_spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            tanstack_router["TanStack Router<br/><i>[Component: @tanstack/router]</i><br/>File-based routing with<br/>code splitting and auth guards"]
        end

        subgraph features["Feature Pages"]
            %% KIND: boundary
            auth_pages["Auth Pages<br/><i>[Component: /login, /signup,<br/>/recover-password, /reset-password]</i><br/>Login, registration,<br/>password recovery flows"]
            %% KIND: boundary
            dashboard_page["Dashboard<br/><i>[Component: /]</i><br/>Home page with<br/>user greeting"]
            %% KIND: boundary
            items_feature["Items Feature<br/><i>[Component: /items]</i><br/>DataTable with CRUD,<br/>add/edit/delete dialogs"]
            %% KIND: boundary
            settings_feature["User Settings<br/><i>[Component: /settings]</i><br/>Profile info, password change,<br/>account deletion"]
            %% KIND: boundary
            admin_feature["Admin Panel<br/><i>[Component: /admin]</i><br/>User management table,<br/>superuser only"]
        end

        subgraph services_layer["Services & State"]
            %% KIND: integration
            api_client["OpenAPI Client<br/><i>[Component: @hey-api/openapi-ts]</i><br/>Generated typed API client<br/>(LoginService, UsersService, ItemsService)"]
            %% KIND: data_access
            query_layer["React Query<br/><i>[Component: @tanstack/react-query]</i><br/>Server state caching,<br/>mutations, error handling"]
            %% KIND: service_layer
            auth_hook["Auth Hook<br/><i>[Component: hooks/useAuth.ts]</i><br/>Current user state,<br/>login/logout helpers"]
        end

        subgraph ui_layer["UI Layer"]
            %% KIND: boundary
            ui_components["UI Primitives<br/><i>[Component: shadcn / Radix UI]</i><br/>Button, Dialog, Table, Form,<br/>Input, Card, DropdownMenu, etc."]
            %% KIND: boundary
            common_components["Common Components<br/><i>[Component: components/Common/*]</i><br/>DataTable, AuthLayout, Sidebar,<br/>Logo, Footer, NotFound"]
            %% KIND: service_layer
            theme_provider["Theme Provider<br/><i>[Component: next-themes]</i><br/>Dark/light/system theme<br/>with localStorage persistence"]
        end

    end

    tanstack_router --> auth_pages
    tanstack_router --> dashboard_page
    tanstack_router --> items_feature
    tanstack_router --> settings_feature
    tanstack_router --> admin_feature

    auth_pages --> api_client
    dashboard_page --> auth_hook
    items_feature --> api_client
    settings_feature --> api_client
    admin_feature --> api_client

    auth_pages --> ui_components
    auth_pages --> common_components
    items_feature --> ui_components
    items_feature --> common_components
    settings_feature --> ui_components
    admin_feature --> ui_components

    auth_hook --> api_client
    api_client --> query_layer
    ui_components --> theme_provider
    common_components --> ui_components

    query_layer -->|"HTTP/JSON to /api/v1/*"| fastapi_backend
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Project
%% Covers L1, L2, and all L3 diagrams.
%% Every subgraph and its children from every diagram are listed.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [end_user, admin_user]
system_boundary  CONTAINS [traefik, react_spa, fastapi_backend, pg]
external         CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment (FastAPI Backend) ────────────────
fastapi_backend  CONTAINS [middleware, routes, services, core, data]
  middleware     CONTAINS [cors_mw, auth_deps]
  routes         CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services       CONTAINS [crud_layer, email_utils]
  core           CONTAINS [security_core, config_core]
  data           CONTAINS [models_layer, db_engine]

%% ── L2→L3 internal containment (React SPA) ─────────────────────
react_spa        CONTAINS [routing, features, services_layer, ui_layer]
  routing        CONTAINS [tanstack_router]
  features       CONTAINS [auth_pages, dashboard_page, items_feature, settings_feature, admin_feature]
  services_layer CONTAINS [api_client, query_layer, auth_hook]
  ui_layer       CONTAINS [ui_components, common_components, theme_provider]
```
