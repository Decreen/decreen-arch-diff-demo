# C4 Architecture Model — Full Stack FastAPI Project

> Auto-generated C4 architecture diagrams using Mermaid.
> Covers L1 (System Context), L2 (Container), and L3 (Component) levels.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
%% C4 Level 1 — System Context Diagram

graph TB
    subgraph users["Users"]
        user["👤 End User<br/><i>Browser-based user</i>"]
        admin_user["👤 Admin User<br/><i>Superuser with elevated privileges</i>"]
    end

    subgraph platform_boundary["Full Stack FastAPI Project"]
        platform["🖥️ Full Stack FastAPI Platform<br/><i>Web application with API backend,<br/>SPA frontend, and database</i>"]
    end

    subgraph external["External Systems"]
        smtp["📧 SMTP Provider<br/><i>Email delivery service</i>"]
        sentry["📊 Sentry<br/><i>Error tracking & APM</i>"]
        letsencrypt["🔒 Let's Encrypt<br/><i>TLS certificate authority</i>"]
    end

    user -- "Uses web application<br/>[HTTPS]" --> platform
    admin_user -- "Manages users & settings<br/>[HTTPS]" --> platform
    platform -- "Sends emails<br/>[SMTP]" --> smtp
    platform -- "Reports errors<br/>[HTTPS]" --> sentry
    platform -- "Obtains TLS certificates<br/>[ACME]" --> letsencrypt
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
%% C4 Level 2 — Container Diagram

graph TB
    subgraph users["Users"]
        user["👤 End User"]
        admin_user["👤 Admin User"]
    end

    subgraph platform_boundary["Full Stack FastAPI Project"]
        traefik["🔀 Traefik<br/><i>Reverse Proxy</i><br/>[Traefik v3.6]"]
        spa["🌐 Frontend SPA<br/><i>Single-Page Application</i><br/>[React 19, Vite, Nginx]"]
        fastapi["⚙️ Backend API<br/><i>REST API Server</i><br/>[Python, FastAPI, Uvicorn]"]
        pg["🗄️ PostgreSQL<br/><i>Relational Database</i><br/>[PostgreSQL 18]"]
        adminer["🔧 Adminer<br/><i>Database Admin UI</i><br/>[Adminer]"]
    end

    subgraph external["External Systems"]
        smtp["📧 SMTP Provider"]
        sentry["📊 Sentry"]
        letsencrypt["🔒 Let's Encrypt"]
    end

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik

    traefik -- "dashboard.*<br/>[HTTP :80]" --> spa
    traefik -- "api.*<br/>[HTTP :8000]" --> fastapi
    traefik -- "adminer.*<br/>[HTTP :8080]" --> adminer

    spa -- "API calls<br/>[HTTP /api/v1]" --> fastapi
    fastapi -- "SQL queries<br/>[psycopg]" --> pg
    adminer -- "SQL queries<br/>[TCP :5432]" --> pg

    fastapi -- "Sends emails<br/>[SMTP]" --> smtp
    fastapi -- "Reports errors<br/>[HTTPS]" --> sentry
    traefik -- "ACME challenge<br/>[HTTPS]" --> letsencrypt
```

---

## L3: Backend API (FastAPI)

```mermaid
%% SCOPE: urn:c4:container:fastapi
%% C4 Level 3 — Component Diagram for Backend API

graph TB
    subgraph fastapi["Backend API — FastAPI"]

        subgraph middleware["Middleware"]
            %% KIND: boundary
            cors_mw["🛡️ CORS Middleware<br/><i>Cross-origin request handling</i><br/>[Starlette CORSMiddleware]"]
        end

        subgraph api_deps["API Dependencies"]
            %% KIND: boundary
            auth_dep["🔑 Auth Dependency<br/><i>JWT token validation,<br/>current user injection</i>"]
            %% KIND: boundary
            db_session_dep["💉 DB Session Dependency<br/><i>Per-request SQLModel session</i>"]
            %% KIND: boundary
            superuser_dep["🛡️ Superuser Guard<br/><i>Restricts to is_superuser=true</i>"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["🔐 Login Routes<br/><i>/login/access-token,<br/>/password-recovery,<br/>/reset-password</i>"]
            %% KIND: router
            user_routes["👥 User Routes<br/><i>/users CRUD,<br/>/users/me,<br/>/users/signup</i>"]
            %% KIND: router
            item_routes["📦 Item Routes<br/><i>/items CRUD</i>"]
            %% KIND: router
            util_routes["🔧 Utility Routes<br/><i>/utils/health-check,<br/>/utils/test-email</i>"]
        end

        subgraph services["Business Logic"]
            %% KIND: data_access
            crud_layer["📋 CRUD Layer<br/><i>create_user, update_user,<br/>authenticate, get_user_by_email</i>"]
            %% KIND: service_layer
            email_utils["✉️ Email Utilities<br/><i>Jinja2 templates, SMTP sending,<br/>reset tokens, welcome emails</i>"]
        end

        subgraph core["Core"]
            %% KIND: service_layer
            core_config["⚙️ Configuration<br/><i>Pydantic Settings,<br/>env-based config</i>"]
            %% KIND: service_layer
            core_security["🔒 Security<br/><i>JWT creation/validation,<br/>Argon2/Bcrypt hashing</i>"]
            %% KIND: data_access
            core_db["🗃️ Database Engine<br/><i>SQLAlchemy engine,<br/>init_db, session factory</i>"]
        end

        subgraph models_layer["Models & Schemas"]
            %% KIND: storage
            models["📐 SQLModel Models<br/><i>User, Item tables</i>"]
            %% KIND: boundary
            schemas["📝 Pydantic Schemas<br/><i>Request/Response DTOs:<br/>UserPublic, ItemPublic, Token, etc.</i>"]
        end
    end

    spa["🌐 Frontend SPA"]
    pg["🗄️ PostgreSQL"]
    smtp["📧 SMTP Provider"]
    sentry["📊 Sentry"]

    spa -- "HTTP /api/v1/*" --> cors_mw
    cors_mw --> routes

    login_routes --> auth_dep
    login_routes --> crud_layer
    login_routes --> core_security
    login_routes --> email_utils

    user_routes --> auth_dep
    user_routes --> superuser_dep
    user_routes --> crud_layer
    user_routes --> email_utils

    item_routes --> auth_dep
    item_routes --> db_session_dep

    util_routes --> superuser_dep

    auth_dep --> core_security
    auth_dep --> db_session_dep

    crud_layer --> models
    crud_layer --> core_security
    crud_layer --> db_session_dep

    email_utils --> core_config
    email_utils --> smtp

    core_db --> pg
    db_session_dep --> core_db

    item_routes --> models
    item_routes --> schemas

    user_routes --> schemas
    login_routes --> schemas

    fastapi -- "Sentry SDK" --> sentry
```

---

## L3: Frontend SPA (React)

```mermaid
%% SCOPE: urn:c4:container:spa
%% C4 Level 3 — Component Diagram for Frontend SPA

graph TB
    subgraph spa["Frontend SPA — React"]

        subgraph routing["Routing"]
            %% KIND: router
            router["🧭 TanStack Router<br/><i>File-based routing,<br/>auto code-splitting</i>"]
            %% KIND: boundary
            auth_guard["🛡️ Auth Guard<br/><i>beforeLoad isLoggedIn() check,<br/>redirect to /login</i>"]
        end

        subgraph api_client["API Client"]
            %% KIND: integration
            openapi_client["🔗 OpenAPI Client<br/><i>Auto-generated from openapi.json,<br/>Axios-based HTTP</i>"]
            %% KIND: service_layer
            api_services["📡 API Services<br/><i>LoginService, UsersService,<br/>ItemsService, UtilsService</i>"]
        end

        subgraph auth_module["Auth Module"]
            %% KIND: service_layer
            use_auth["🔑 useAuth Hook<br/><i>Login/logout, current user,<br/>token management</i>"]
        end

        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            query_client["📊 React Query<br/><i>TanStack Query for<br/>server state caching</i>"]
            %% KIND: service_layer
            theme_provider["🎨 Theme Provider<br/><i>Light/dark/system theme,<br/>localStorage persistence</i>"]
        end

        subgraph features["Feature Pages"]
            %% KIND: boundary
            dashboard_page["🏠 Dashboard<br/><i>Landing page</i>"]
            %% KIND: boundary
            items_page["📦 Items Management<br/><i>CRUD with DataTable</i>"]
            %% KIND: boundary
            admin_page["👑 Admin Panel<br/><i>User management,<br/>superuser-only</i>"]
            %% KIND: boundary
            settings_page["⚙️ User Settings<br/><i>Profile, password,<br/>account deletion</i>"]
            %% KIND: boundary
            auth_pages["🔐 Auth Pages<br/><i>Login, Signup,<br/>Password Recovery/Reset</i>"]
        end

        subgraph ui_layer["UI Layer"]
            %% KIND: boundary
            layout_cmp["📐 Layout Components<br/><i>AuthLayout, AppSidebar,<br/>Footer, Logo</i>"]
            %% KIND: boundary
            ui_cmp["🎛️ UI Primitives<br/><i>shadcn/Radix: Button, Dialog,<br/>DataTable, Form, Input, etc.</i>"]
        end
    end

    user["👤 End User"]
    admin_user["👤 Admin User"]
    fastapi["⚙️ Backend API"]

    user -- "Browser" --> router
    admin_user -- "Browser" --> router

    router --> auth_guard
    auth_guard --> use_auth

    router --> features

    dashboard_page --> query_client
    items_page --> api_services
    items_page --> query_client
    admin_page --> api_services
    admin_page --> query_client
    settings_page --> api_services
    settings_page --> query_client
    auth_pages --> use_auth

    features --> layout_cmp
    features --> ui_cmp

    use_auth --> api_services
    api_services --> openapi_client
    openapi_client -- "HTTP /api/v1/*" --> fastapi

    query_client --> api_services
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers all levels (L1 → L2 → L3)
%% Every subgraph and its children from every diagram above.
%% ══════════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [platform]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 container-level ──────────────────────────────────────────────
platform_boundary  CONTAINS [traefik, spa, fastapi, pg, adminer]

%% ── L3: Backend API (fastapi) ───────────────────────────────────────
fastapi            CONTAINS [middleware, api_deps, routes, services, core, models_layer]
  middleware       CONTAINS [cors_mw]
  api_deps         CONTAINS [auth_dep, db_session_dep, superuser_dep]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services         CONTAINS [crud_layer, email_utils]
  core             CONTAINS [core_config, core_security, core_db]
  models_layer     CONTAINS [models, schemas]

%% ── L3: Frontend SPA (spa) ──────────────────────────────────────────
spa                CONTAINS [routing, api_client, auth_module, state_mgmt, features, ui_layer]
  routing          CONTAINS [router, auth_guard]
  api_client       CONTAINS [openapi_client, api_services]
  auth_module      CONTAINS [use_auth]
  state_mgmt       CONTAINS [query_client, theme_provider]
  features         CONTAINS [dashboard_page, items_page, admin_page, settings_page, auth_pages]
  ui_layer         CONTAINS [layout_cmp, ui_cmp]
```
