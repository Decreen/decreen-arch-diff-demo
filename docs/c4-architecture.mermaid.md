# C4 Architecture — Full Stack FastAPI Platform

> Auto-generated C4 model for the Full Stack FastAPI Project.
> Each diagram follows [C4 model](https://c4model.com/) layering conventions.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack_fastapi_platform
graph TB

    subgraph users["Users"]
        user["User<br/>[Person]<br/>End user of the web application"]
        admin_user["Admin<br/>[Person]<br/>Superuser who manages users and data"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        platform["Full Stack FastAPI Platform<br/>[Software System]<br/>Full-stack web application with<br/>REST API and React frontend"]
    end

    subgraph external_systems["External Systems"]
        smtp["SMTP Server<br/>[External System]<br/>Email delivery service"]
        sentry["Sentry<br/>[External System]<br/>Error monitoring and tracing"]
        letsencrypt["Let's Encrypt<br/>[External System]<br/>TLS certificate authority"]
    end

    user -->|"Uses web application"| platform
    admin_user -->|"Manages users via"| platform
    platform -->|"Sends emails via SMTP"| smtp
    platform -->|"Reports errors to"| sentry
    platform -->|"Obtains TLS certificates"| letsencrypt
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fullstack_fastapi_platform
graph TB

    subgraph users["Users"]
        user["User<br/>[Person]"]
        admin_user["Admin<br/>[Person]"]
    end

    subgraph platform_boundary["Full Stack FastAPI Platform"]
        traefik["Traefik<br/>[Container: Traefik 3.6]<br/>Reverse proxy, TLS termination,<br/>subdomain-based routing"]
        spa["React SPA<br/>[Container: React 19 / Vite / Nginx]<br/>Single-page application<br/>served as static files via Nginx"]
        fastapi["FastAPI Backend<br/>[Container: Python / FastAPI]<br/>REST API with JWT auth,<br/>OpenAPI schema generation"]
        pg["PostgreSQL<br/>[Container: PostgreSQL 18]<br/>Relational database for<br/>users and items"]
        adminer["Adminer<br/>[Container: PHP]<br/>Database administration UI"]
    end

    subgraph external_systems["External Systems"]
        smtp["SMTP Server<br/>[External System]"]
        sentry["Sentry<br/>[External System]"]
        letsencrypt["Let's Encrypt<br/>[External System]"]
    end

    user -->|"HTTPS"| traefik
    admin_user -->|"HTTPS"| traefik
    traefik -->|"dashboard.DOMAIN"| spa
    traefik -->|"api.DOMAIN"| fastapi
    traefik -->|"adminer.DOMAIN"| adminer
    traefik -->|"ACME TLS challenge"| letsencrypt

    spa -->|"REST API calls<br/>[JSON/HTTPS]"| fastapi
    fastapi -->|"SQL queries<br/>[psycopg]"| pg
    fastapi -->|"Sends emails<br/>[SMTP]"| smtp
    fastapi -->|"Reports errors<br/>[HTTPS]"| sentry
    adminer -->|"SQL queries"| pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi
graph TB

    spa["React SPA"]

    subgraph fastapi["FastAPI Backend"]

        %% KIND: boundary
        subgraph middleware["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/>[Starlette CORSMiddleware]<br/>Cross-origin request handling"]
            %% KIND: integration
            sentry_mw["Sentry Integration<br/>[sentry-sdk for FastAPI]<br/>Error capture and tracing"]
        end

        %% KIND: router
        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes<br/>[/api/v1/login]<br/>OAuth2 token endpoint,<br/>password recovery and reset"]
            %% KIND: router
            user_routes["User Routes<br/>[/api/v1/users]<br/>User CRUD, profile update, signup"]
            %% KIND: router
            item_routes["Item Routes<br/>[/api/v1/items]<br/>Item CRUD, owner-scoped access"]
            %% KIND: router
            util_routes["Utils Routes<br/>[/api/v1/utils]<br/>Health check, test email"]
            %% KIND: router
            private_routes["Private Routes<br/>[/api/v1/private]<br/>Dev-only user creation"]
        end

        %% KIND: boundary
        subgraph deps["Dependencies"]
            %% KIND: service_layer
            auth_dep["Auth Dependency<br/>[OAuth2PasswordBearer + JWT]<br/>Token validation, current user resolution"]
            %% KIND: data_access
            db_dep["DB Session Dependency<br/>[SQLModel Session]<br/>Request-scoped database session"]
            %% KIND: service_layer
            superuser_dep["Superuser Guard<br/>[FastAPI Depends]<br/>Role-based access control"]
        end

        %% KIND: service_layer
        crud["CRUD Layer<br/>[crud.py]<br/>User and Item data operations,<br/>authentication logic"]

        %% KIND: boundary
        subgraph core["Core"]
            %% KIND: service_layer
            config["Config<br/>[Pydantic Settings]<br/>Environment-based configuration"]
            %% KIND: data_access
            db_engine["DB Engine<br/>[SQLAlchemy create_engine]<br/>Connection pool management"]
            %% KIND: service_layer
            security["Security<br/>[PyJWT + pwdlib]<br/>JWT token creation/validation,<br/>Argon2/Bcrypt password hashing"]
        end

        %% KIND: data_access
        models["Models<br/>[SQLModel]<br/>User, Item table definitions<br/>and Pydantic schemas"]

        %% KIND: integration
        email_utils["Email Utils<br/>[Jinja2 + emails lib]<br/>Template rendering, SMTP sending,<br/>password-reset token generation"]

    end

    pg["PostgreSQL"]
    smtp["SMTP Server"]
    sentry["Sentry"]

    spa -->|"HTTP requests"| middleware
    middleware -->|"Passes request to"| routes
    routes -->|"Injects via Depends"| deps
    routes -->|"Calls"| crud
    routes -->|"Sends via"| email_utils

    deps -->|"Validates tokens"| security
    deps -->|"Creates sessions from"| db_engine

    crud -->|"Reads / writes"| models
    crud -->|"Hashes passwords via"| security

    models -->|"Mapped by"| db_engine
    db_engine -->|"SQL"| pg
    email_utils -->|"SMTP"| smtp
    sentry_mw -->|"Reports errors"| sentry

    config -.->|"Configures"| db_engine
    config -.->|"Configures"| security
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

    subgraph spa["React SPA"]

        %% KIND: router
        subgraph routing["Routing Layer"]
            %% KIND: router
            route_tree["Route Tree<br/>[TanStack Router]<br/>File-based route generation"]
            %% KIND: boundary
            root_layout["Root Layout<br/>[__root.tsx]<br/>Error boundary, not-found,<br/>devtools integration"]
            %% KIND: boundary
            auth_layout["Auth Layout<br/>[AuthLayout component]<br/>Login / signup page wrapper"]
        end

        %% KIND: boundary
        subgraph features["Feature Modules"]
            %% KIND: service_layer
            auth_feature["Auth<br/>[Login, Signup, Recovery, Reset]<br/>Authentication user flows"]
            %% KIND: service_layer
            dashboard_feature["Dashboard<br/>[index.tsx]<br/>Welcome page with user info"]
            %% KIND: service_layer
            items_feature["Items<br/>[items.tsx + CRUD dialogs]<br/>Item list, add, edit, delete"]
            %% KIND: service_layer
            admin_feature["Admin<br/>[admin.tsx + CRUD dialogs]<br/>User management for superusers"]
            %% KIND: service_layer
            settings_feature["Settings<br/>[settings.tsx + tabs]<br/>Profile, password, account deletion"]
        end

        %% KIND: boundary
        subgraph services_layer["Services"]
            %% KIND: integration
            api_client["OpenAPI Client<br/>[Generated SDK via @hey-api]<br/>Type-safe REST API calls"]
            %% KIND: service_layer
            hooks_layer["Custom Hooks<br/>[useAuth, useCustomToast,<br/>useCopyToClipboard, useMobile]"]
        end

        %% KIND: boundary
        subgraph ui_layer["UI Components"]
            %% KIND: boundary
            common_components["Common Components<br/>[DataTable, Logo, Footer,<br/>NotFound, ErrorComponent]"]
            %% KIND: boundary
            shadcn_ui["shadcn/ui Primitives<br/>[Radix UI + Tailwind CSS]<br/>Button, Dialog, Form, Table, etc."]
            %% KIND: boundary
            sidebar_component["Sidebar Navigation<br/>[AppSidebar]<br/>Dashboard, Items, Admin links"]
        end

        %% KIND: boundary
        subgraph state["State Management"]
            %% KIND: data_access
            query_client["Query Client<br/>[TanStack Query]<br/>Server state cache,<br/>error/mutation handling"]
            %% KIND: service_layer
            theme_provider["Theme Provider<br/>[next-themes pattern]<br/>Dark / light / system mode"]
        end

    end

    fastapi["FastAPI Backend"]

    routing -->|"Renders"| features
    features -->|"Uses"| services_layer
    features -->|"Renders"| ui_layer
    features -->|"Reads / writes"| state

    hooks_layer -->|"Calls"| api_client
    hooks_layer -->|"Manages cache via"| query_client

    api_client -->|"REST API<br/>[JSON/HTTPS]"| fastapi
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi, pg, adminer]
external_systems   CONTAINS [smtp, sentry, letsencrypt]

%% ── L2→L3 internal containment ──────────────────────────────────
fastapi CONTAINS [middleware, routes, deps, crud, core, models, email_utils]
  middleware CONTAINS [cors_mw, sentry_mw]
  routes     CONTAINS [login_routes, user_routes, item_routes, util_routes, private_routes]
  deps       CONTAINS [auth_dep, db_dep, superuser_dep]
  core       CONTAINS [config, db_engine, security]

spa CONTAINS [routing, features, services_layer, ui_layer, state]
  routing        CONTAINS [route_tree, root_layout, auth_layout]
  features       CONTAINS [auth_feature, dashboard_feature, items_feature, admin_feature, settings_feature]
  services_layer CONTAINS [api_client, hooks_layer]
  ui_layer       CONTAINS [common_components, shadcn_ui, sidebar_component]
  state          CONTAINS [query_client, theme_provider]
```
