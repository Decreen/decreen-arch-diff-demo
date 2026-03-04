# C4 Architecture Model — FastAPI Full-Stack Template

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-app
flowchart TB

    subgraph users["Users"]
        anonymous_user(["<b>Anonymous User</b><br/><i>Signs up, logs in,<br/>recovers password</i>"])
        auth_user(["<b>Authenticated User</b><br/><i>Manages own items<br/>and profile</i>"])
        superuser(["<b>Superuser</b><br/><i>Administers users<br/>and system</i>"])
    end

    subgraph fullstack_boundary["FastAPI Full-Stack App"]
        fullstack_app["<b>FastAPI Full-Stack App</b><br/><i>[Software System]</i><br/>Web application with REST API,<br/>React SPA, and PostgreSQL database"]
    end

    subgraph external["External Systems"]
        smtp["<b>SMTP Server</b><br/><i>[External System]</i><br/>Email delivery"]
        sentry["<b>Sentry</b><br/><i>[External System]</i><br/>Error monitoring"]
        letsencrypt["<b>Let's Encrypt</b><br/><i>[External System]</i><br/>TLS certificate authority"]
    end

    anonymous_user -->|"HTTPS"| fullstack_app
    auth_user -->|"HTTPS"| fullstack_app
    superuser -->|"HTTPS"| fullstack_app

    fullstack_app -->|"SMTP"| smtp
    fullstack_app -->|"HTTPS"| sentry
    fullstack_app -->|"ACME"| letsencrypt
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fullstack-app
flowchart TB

    subgraph users["Users"]
        anonymous_user(["<b>Anonymous User</b>"])
        auth_user(["<b>Authenticated User</b>"])
        superuser(["<b>Superuser</b>"])
    end

    subgraph fullstack_boundary["FastAPI Full-Stack App"]
        traefik["<b>Traefik</b><br/><i>[Container: Traefik 3.6]</i><br/>Reverse proxy, TLS termination,<br/>request routing"]
        spa["<b>React SPA</b><br/><i>[Container: React 19 / Nginx]</i><br/>Single-page application<br/>dashboard"]
        fastapi_backend["<b>FastAPI Backend</b><br/><i>[Container: Python / FastAPI]</i><br/>REST API server with<br/>JWT authentication"]
        pg["<b>PostgreSQL</b><br/><i>[Container: PostgreSQL 18]</i><br/>Primary relational<br/>data store"]
        adminer["<b>Adminer</b><br/><i>[Container: Adminer]</i><br/>Database admin UI"]
    end

    subgraph external["External Systems"]
        smtp["<b>SMTP Server</b><br/><i>[External]</i>"]
        sentry["<b>Sentry</b><br/><i>[External]</i>"]
        letsencrypt["<b>Let's Encrypt</b><br/><i>[External]</i>"]
    end

    anonymous_user -->|"HTTPS"| traefik
    auth_user -->|"HTTPS"| traefik
    superuser -->|"HTTPS"| traefik

    traefik -->|"HTTP<br/>dashboard.*"| spa
    traefik -->|"HTTP<br/>api.*"| fastapi_backend
    traefik -->|"HTTP<br/>adminer.*"| adminer
    traefik -->|"ACME"| letsencrypt

    spa -->|"REST / JSON<br/>/api/v1/*"| fastapi_backend

    fastapi_backend -->|"SQL<br/>psycopg"| pg
    fastapi_backend -->|"SMTP"| smtp
    fastapi_backend -->|"HTTPS"| sentry

    adminer -->|"SQL"| pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi-backend
flowchart TB

    spa["<b>React SPA</b><br/><i>[Container]</i>"]
    pg["<b>PostgreSQL</b><br/><i>[Container]</i>"]
    smtp["<b>SMTP Server</b><br/><i>[External]</i>"]
    sentry["<b>Sentry</b><br/><i>[External]</i>"]

    subgraph fastapi_backend["FastAPI Backend"]

        subgraph middleware["Middleware"]
            %% KIND: boundary
            cors_mw["<b>CORS Middleware</b><br/><i>[Component]</i><br/>Cross-origin request<br/>handling"]
            %% KIND: integration
            sentry_mw["<b>Sentry Integration</b><br/><i>[Component]</i><br/>Error tracking SDK"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            api_router["<b>API Router</b><br/><i>[Component: api/main.py]</i><br/>Route aggregation<br/>under /api/v1"]
            %% KIND: router
            login_routes["<b>Login Routes</b><br/><i>[Component]</i><br/>POST /login/access-token<br/>Password recovery"]
            %% KIND: router
            users_routes["<b>Users Routes</b><br/><i>[Component]</i><br/>User CRUD &amp; profile<br/>GET/POST/PATCH/DELETE /users"]
            %% KIND: router
            items_routes["<b>Items Routes</b><br/><i>[Component]</i><br/>Item CRUD<br/>GET/POST/PUT/DELETE /items"]
            %% KIND: router
            utils_routes["<b>Utils Routes</b><br/><i>[Component]</i><br/>Health check &amp;<br/>test email"]
            %% KIND: router
            private_routes["<b>Private Routes</b><br/><i>[Component]</i><br/>Local-only dev<br/>endpoints"]
        end

        subgraph auth_deps["Auth &amp; Dependencies"]
            %% KIND: boundary
            oauth2_dep["<b>OAuth2 Bearer</b><br/><i>[Component]</i><br/>Token extraction from<br/>Authorization header"]
            %% KIND: boundary
            session_dep["<b>DB Session Provider</b><br/><i>[Component]</i><br/>Request-scoped<br/>SQLAlchemy session"]
            %% KIND: boundary
            current_user_dep["<b>Current User Resolver</b><br/><i>[Component]</i><br/>JWT decode &amp;<br/>user load"]
            %% KIND: boundary
            superuser_dep["<b>Superuser Guard</b><br/><i>[Component]</i><br/>Admin authorization<br/>check"]
        end

        %% KIND: service_layer
        crud_layer["<b>CRUD Layer</b><br/><i>[Component: crud.py]</i><br/>User &amp; item persistence<br/>authenticate, get_by_email"]
        %% KIND: data_access
        models_layer["<b>Models</b><br/><i>[Component: SQLModel]</i><br/>ORM entities &amp; Pydantic<br/>schemas (User, Item)"]
        %% KIND: service_layer
        security_mod["<b>Security Module</b><br/><i>[Component: core/security.py]</i><br/>JWT encoding, Argon2<br/>password hashing"]
        %% KIND: data_access
        db_mod["<b>DB Module</b><br/><i>[Component: core/db.py]</i><br/>SQLAlchemy engine,<br/>init_db seeding"]
        %% KIND: service_layer
        config_mod["<b>Config</b><br/><i>[Component: Pydantic Settings]</i><br/>Environment-based<br/>application settings"]
        %% KIND: service_layer
        email_utils["<b>Email Utils</b><br/><i>[Component: utils.py]</i><br/>Jinja2 templates &amp;<br/>SMTP sending"]
        %% KIND: data_access
        alembic_mig["<b>Alembic Migrations</b><br/><i>[Component]</i><br/>Database schema<br/>versioning"]
    end

    spa -->|"REST / JSON"| api_router

    api_router --> login_routes
    api_router --> users_routes
    api_router --> items_routes
    api_router --> utils_routes
    api_router --> private_routes

    login_routes --> auth_deps
    users_routes --> auth_deps
    items_routes --> auth_deps
    utils_routes --> auth_deps

    login_routes --> crud_layer
    login_routes --> email_utils
    users_routes --> crud_layer
    users_routes --> email_utils
    items_routes --> crud_layer
    utils_routes --> email_utils

    current_user_dep --> oauth2_dep
    superuser_dep --> current_user_dep
    current_user_dep --> security_mod
    current_user_dep --> session_dep
    session_dep --> db_mod

    crud_layer --> models_layer
    crud_layer --> security_mod
    db_mod --> models_layer

    db_mod -->|"SQL / psycopg"| pg
    alembic_mig -->|"SQL / psycopg"| pg

    email_utils --> security_mod
    email_utils -->|"SMTP"| smtp
    sentry_mw -->|"HTTPS"| sentry

    config_mod -.->|"configures"| cors_mw
    config_mod -.->|"configures"| sentry_mw
    config_mod -.->|"configures"| db_mod
    config_mod -.->|"configures"| email_utils
    config_mod -.->|"configures"| security_mod
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TB

    fastapi_backend["<b>FastAPI Backend</b><br/><i>[Container]</i>"]

    subgraph spa["React SPA"]

        subgraph app_shell["Application Shell"]
            %% KIND: boundary
            main_bootstrap["<b>Main Bootstrap</b><br/><i>[Component: main.tsx]</i><br/>QueryClient, Router,<br/>ThemeProvider mount"]
            %% KIND: boundary
            root_layout["<b>Root Layout</b><br/><i>[Component: __root.tsx]</i><br/>Error boundary &amp;<br/>top-level outlet"]
            %% KIND: boundary
            auth_layout_cmp["<b>Auth Layout</b><br/><i>[Component]</i><br/>Split-pane layout<br/>for auth pages"]
            %% KIND: boundary
            app_layout["<b>App Layout</b><br/><i>[Component: _layout.tsx]</i><br/>Sidebar, footer,<br/>authenticated shell"]
        end

        subgraph routing["Routing &amp; State"]
            %% KIND: router
            router["<b>TanStack Router</b><br/><i>[Component]</i><br/>File-based route tree"]
            %% KIND: service_layer
            query_client["<b>React Query</b><br/><i>[Component]</i><br/>Server state caching<br/>&amp; synchronization"]
            %% KIND: service_layer
            theme_provider["<b>Theme Provider</b><br/><i>[Component]</i><br/>Light / dark / system<br/>theme context"]
        end

        subgraph auth_feature["Auth Feature"]
            %% KIND: router
            login_page["<b>Login Page</b><br/><i>[Component]</i>"]
            %% KIND: router
            signup_page["<b>Signup Page</b><br/><i>[Component]</i>"]
            %% KIND: router
            recover_page["<b>Password Recovery</b><br/><i>[Component]</i>"]
            %% KIND: router
            reset_page["<b>Password Reset</b><br/><i>[Component]</i>"]
            %% KIND: service_layer
            use_auth["<b>useAuth Hook</b><br/><i>[Component]</i><br/>Login, logout,<br/>auth state"]
        end

        subgraph items_feature["Items Feature"]
            %% KIND: router
            items_page["<b>Items Page</b><br/><i>[Component]</i><br/>Item list with DataTable"]
            %% KIND: service_layer
            item_dialogs["<b>Item Dialogs</b><br/><i>[Component]</i><br/>Add / Edit / Delete<br/>item forms"]
        end

        subgraph admin_feature["Admin Feature"]
            %% KIND: router
            admin_page["<b>Admin Page</b><br/><i>[Component]</i><br/>User management<br/>with DataTable"]
            %% KIND: service_layer
            user_dialogs["<b>User Dialogs</b><br/><i>[Component]</i><br/>Add / Edit / Delete<br/>user forms"]
        end

        subgraph settings_feature["Settings Feature"]
            %% KIND: router
            settings_page["<b>Settings Page</b><br/><i>[Component]</i><br/>Profile tabs"]
            %% KIND: service_layer
            profile_components["<b>Profile Components</b><br/><i>[Component]</i><br/>UserInfo, ChangePassword,<br/>DeleteAccount"]
        end

        %% KIND: integration
        api_client["<b>API Client</b><br/><i>[Component: OpenAPI Generated]</i><br/>Axios HTTP layer with<br/>JWT token injection"]
        %% KIND: boundary
        shared_ui["<b>Shared UI Components</b><br/><i>[Component: Radix / Tailwind]</i><br/>DataTable, dialogs, forms,<br/>sidebar, toasts"]
    end

    main_bootstrap --> router
    main_bootstrap --> query_client
    main_bootstrap --> theme_provider

    root_layout --> router

    router --> login_page
    router --> signup_page
    router --> recover_page
    router --> reset_page
    router --> items_page
    router --> admin_page
    router --> settings_page

    login_page --> auth_layout_cmp
    signup_page --> auth_layout_cmp
    recover_page --> auth_layout_cmp
    reset_page --> auth_layout_cmp
    items_page --> app_layout
    admin_page --> app_layout
    settings_page --> app_layout

    login_page --> use_auth
    signup_page --> use_auth

    use_auth --> api_client
    recover_page --> api_client
    reset_page --> api_client

    items_page --> item_dialogs
    items_page --> shared_ui
    item_dialogs --> api_client

    admin_page --> user_dialogs
    admin_page --> shared_ui
    user_dialogs --> api_client

    settings_page --> profile_components
    settings_page --> shared_ui
    profile_components --> api_client

    api_client -->|"REST / JSON<br/>/api/v1/*"| fastapi_backend
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [anonymous_user, auth_user, superuser]
fullstack_boundary CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 → L3: FastAPI Backend ───────────────────────────────────
fastapi_backend CONTAINS [middleware, routes, auth_deps, crud_layer, models_layer, security_mod, db_mod, config_mod, email_utils, alembic_mig]
  middleware   CONTAINS [cors_mw, sentry_mw]
  routes       CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes, private_routes]
  auth_deps    CONTAINS [oauth2_dep, session_dep, current_user_dep, superuser_dep]

%% ── L2 → L3: React SPA ────────────────────────────────────────
spa CONTAINS [app_shell, routing, auth_feature, items_feature, admin_feature, settings_feature, api_client, shared_ui]
  app_shell        CONTAINS [main_bootstrap, root_layout, auth_layout_cmp, app_layout]
  routing          CONTAINS [router, query_client, theme_provider]
  auth_feature     CONTAINS [login_page, signup_page, recover_page, reset_page, use_auth]
  items_feature    CONTAINS [items_page, item_dialogs]
  admin_feature    CONTAINS [admin_page, user_dialogs]
  settings_feature CONTAINS [settings_page, profile_components]
```
