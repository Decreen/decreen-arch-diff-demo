# C4 Architecture Model — Full Stack FastAPI Template

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for every container with internal structure.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi-platform
flowchart TB

    subgraph users["Users"]
        user(["👤 User<br/><i>Web application end-user who<br/>manages items and profile</i>"])
        admin_user(["👤 Admin<br/><i>Superuser who manages<br/>users and system settings</i>"])
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["🖥️ Full Stack FastAPI Platform<br/><i>Web application for user management,<br/>item CRUD, and administration</i>"]
    end

    subgraph external["External Services"]
        smtp_service["📧 SMTP Email Service<br/><i>Sends transactional emails:<br/>password recovery, new account</i>"]
        sentry["🐛 Sentry<br/><i>Error monitoring &amp;<br/>performance tracing</i>"]
    end

    user -- "Uses web application<br/>[HTTPS]" --> platform
    admin_user -- "Manages users &amp; system<br/>[HTTPS]" --> platform
    platform -- "Sends emails<br/>[SMTP/TLS]" --> smtp_service
    platform -- "Reports errors<br/>[HTTPS]" --> sentry
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi-platform
flowchart TB

    user(["👤 User"])
    admin_user(["👤 Admin"])

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["🔀 Traefik<br/><i>Reverse proxy &amp; TLS termination<br/>[Docker container]</i>"]
        spa["🌐 React SPA<br/><i>React 19 + Vite + TanStack Router<br/>Served via Nginx [Docker container]</i>"]
        fastapi_backend["⚙️ FastAPI Backend<br/><i>Python FastAPI REST API<br/>Port 8000 [Docker container]</i>"]
        pg[("🗄️ PostgreSQL<br/><i>PostgreSQL 18<br/>Primary relational data store</i>")]
        adminer["🔧 Adminer<br/><i>Database administration UI<br/>[Docker container]</i>"]
    end

    subgraph external["External Services"]
        smtp_service["📧 SMTP Email Service"]
        sentry["🐛 Sentry"]
    end

    user -- "HTTPS" --> traefik
    admin_user -- "HTTPS" --> traefik
    traefik -- "dashboard.DOMAIN<br/>[HTTP]" --> spa
    traefik -- "api.DOMAIN<br/>[HTTP]" --> fastapi_backend
    traefik -- "adminer.DOMAIN<br/>[HTTP]" --> adminer
    spa -- "REST/JSON<br/>[Axios via OpenAPI client]" --> fastapi_backend
    fastapi_backend -- "SQL<br/>[psycopg / SQLModel]" --> pg
    fastapi_backend -- "Sends emails<br/>[SMTP/TLS]" --> smtp_service
    fastapi_backend -- "Reports errors<br/>[HTTPS / Sentry SDK]" --> sentry
    adminer -- "SQL" --> pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TB

    spa["🌐 React SPA"]
    pg[("🗄️ PostgreSQL")]
    smtp_service["📧 SMTP Email Service"]
    sentry["🐛 Sentry"]

    subgraph fastapi_backend["FastAPI Backend"]

        %% KIND: boundary
        cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware<br/>Origin allowlist from config</i>"]

        subgraph api_routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes<br/><i>/api/v1/login<br/>OAuth2 token, password recovery/reset</i>"]
            %% KIND: router
            users_routes["Users Routes<br/><i>/api/v1/users<br/>CRUD, signup, profile, admin</i>"]
            %% KIND: router
            items_routes["Items Routes<br/><i>/api/v1/items<br/>Item CRUD, owner/superuser scoped</i>"]
            %% KIND: router
            utils_routes["Utils Routes<br/><i>/api/v1/utils<br/>Health check, test email</i>"]
        end

        subgraph auth_deps["Auth &amp; Dependencies"]
            %% KIND: service_layer
            oauth2_bearer["OAuth2PasswordBearer<br/><i>Token extraction from header</i>"]
            %% KIND: service_layer
            get_current_user_dep["get_current_user<br/><i>JWT decode &amp; user lookup</i>"]
            %% KIND: service_layer
            get_superuser_dep["get_current_active_superuser<br/><i>Superuser privilege check</i>"]
            %% KIND: data_access
            get_db_dep["get_db<br/><i>SQLModel Session provider</i>"]
        end

        %% KIND: service_layer
        crud_layer["CRUD Layer<br/><i>create_user, update_user,<br/>authenticate, create_item</i>"]

        %% KIND: data_access
        models_layer["SQLModel Models<br/><i>User, Item, Token,<br/>request/response schemas</i>"]

        subgraph core["Core"]
            %% KIND: service_layer
            core_config["Config<br/><i>Pydantic Settings<br/>env-based configuration</i>"]
            %% KIND: data_access
            core_db["DB Engine<br/><i>SQLAlchemy create_engine<br/>session factory, init_db</i>"]
            %% KIND: service_layer
            core_security["Security<br/><i>JWT creation/verification<br/>Argon2 + Bcrypt hashing</i>"]
        end

        %% KIND: integration
        email_utils["Email Utils<br/><i>Jinja2 template rendering<br/>SMTP send, token generation</i>"]

        %% KIND: storage
        alembic["Alembic Migrations<br/><i>Schema versioning<br/>upgrade/downgrade scripts</i>"]

    end

    spa -- "REST/JSON" --> cors_mw
    cors_mw --> api_routes
    api_routes --> auth_deps
    auth_deps --> crud_layer
    crud_layer --> models_layer
    crud_layer --> core_security
    models_layer --> core_db
    core_db -- "SQL" --> pg
    get_db_dep --> core_db
    get_current_user_dep --> core_security
    oauth2_bearer --> get_current_user_dep
    get_current_user_dep --> get_superuser_dep
    login_routes --> email_utils
    users_routes --> email_utils
    email_utils -- "SMTP/TLS" --> smtp_service
    core_config --> sentry
    alembic -- "DDL" --> pg
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TB

    user(["👤 User"])
    fastapi_backend["⚙️ FastAPI Backend"]

    subgraph spa["React SPA"]

        subgraph routing["TanStack Router"]
            %% KIND: router
            route_login["Login Route<br/><i>/login</i>"]
            %% KIND: router
            route_signup["Signup Route<br/><i>/signup</i>"]
            %% KIND: router
            route_admin["Admin Route<br/><i>/admin</i>"]
            %% KIND: router
            route_items["Items Route<br/><i>/items</i>"]
            %% KIND: router
            route_settings["Settings Route<br/><i>/settings</i>"]
            %% KIND: router
            route_password["Password Recovery Routes<br/><i>/recover-password, /reset-password</i>"]
        end

        subgraph features["Feature Components"]
            %% KIND: boundary
            admin_components["Admin Components<br/><i>AddUser, EditUser, DeleteUser,<br/>UserActionsMenu, columns</i>"]
            %% KIND: boundary
            items_components["Items Components<br/><i>AddItem, EditItem, DeleteItem,<br/>ItemActionsMenu, columns</i>"]
            %% KIND: boundary
            settings_components["UserSettings Components<br/><i>UserInformation, ChangePassword,<br/>DeleteAccount, DeleteConfirmation</i>"]
            %% KIND: boundary
            sidebar_components["Sidebar Components<br/><i>AppSidebar, Main nav,<br/>User menu</i>"]
            %% KIND: boundary
            common_components["Common Components<br/><i>DataTable, AuthLayout, Footer,<br/>Logo, ErrorComponent, NotFound</i>"]
        end

        subgraph ui_layer["UI Primitives (shadcn/ui)"]
            %% KIND: boundary
            shadcn_ui["shadcn/ui + Radix<br/><i>Button, Dialog, Form, Table,<br/>Input, Select, Sheet, Tabs, etc.</i>"]
        end

        subgraph hooks_layer["Custom Hooks"]
            %% KIND: service_layer
            use_auth["useAuth<br/><i>Authentication state &amp; actions</i>"]
            %% KIND: service_layer
            use_toast["useCustomToast<br/><i>Toast notifications</i>"]
            %% KIND: service_layer
            use_clipboard["useCopyToClipboard<br/><i>Clipboard API wrapper</i>"]
            %% KIND: service_layer
            use_mobile["useMobile<br/><i>Responsive breakpoint detection</i>"]
        end

        subgraph api_client["OpenAPI Client"]
            %% KIND: integration
            openapi_config["OpenAPI Config<br/><i>Base URL, token injection,<br/>Axios HTTP client</i>"]
            %% KIND: integration
            login_service["LoginService<br/><i>Auth API calls</i>"]
            %% KIND: integration
            users_service["UsersService<br/><i>User CRUD API calls</i>"]
            %% KIND: integration
            items_service["ItemsService<br/><i>Item CRUD API calls</i>"]
            %% KIND: integration
            utils_service["UtilsService<br/><i>Health check &amp; email API calls</i>"]
        end

        %% KIND: service_layer
        theme_provider["Theme Provider<br/><i>Dark/Light mode context<br/>next-themes based</i>"]

        %% KIND: service_layer
        query_client["React Query Client<br/><i>TanStack Query cache,<br/>error handling, mutations</i>"]

    end

    user -- "Interacts via browser" --> routing
    routing --> features
    features --> ui_layer
    features --> hooks_layer
    hooks_layer --> api_client
    features --> api_client
    api_client -- "REST/JSON<br/>[Axios]" --> fastapi_backend
    routing --> theme_provider
    query_client --> api_client
    use_auth --> login_service
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Template
%% Every subgraph/node from every diagram is listed below.
%% Nesting is expressed by indentation.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [platform]
external           CONTAINS [smtp_service, sentry]

%% ── L2 container-level ──────────────────────────────────────────
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp_service, sentry]

%% ── L3: FastAPI Backend (fastapi_backend) ───────────────────────
fastapi_backend    CONTAINS [cors_mw, api_routes, auth_deps, crud_layer, models_layer, core, email_utils, alembic]
  api_routes       CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  auth_deps        CONTAINS [oauth2_bearer, get_current_user_dep, get_superuser_dep, get_db_dep]
  core             CONTAINS [core_config, core_db, core_security]

%% ── L3: React SPA (spa) ────────────────────────────────────────
spa                CONTAINS [routing, features, ui_layer, hooks_layer, api_client, theme_provider, query_client]
  routing          CONTAINS [route_login, route_signup, route_admin, route_items, route_settings, route_password]
  features         CONTAINS [admin_components, items_components, settings_components, sidebar_components, common_components]
  ui_layer         CONTAINS [shadcn_ui]
  hooks_layer      CONTAINS [use_auth, use_toast, use_clipboard, use_mobile]
  api_client       CONTAINS [openapi_config, login_service, users_service, items_service, utils_service]
```
