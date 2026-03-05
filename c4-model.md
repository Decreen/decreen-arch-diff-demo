# C4 Architecture Model — Full Stack FastAPI Project

## L1: System Context

```mermaid
graph TD
    subgraph users["Users"]
        user["User<br/><i>Person</i><br/>Authenticates, manages items<br/>and personal profile"]
        admin_user["Admin / Superuser<br/><i>Person</i><br/>Manages users and<br/>system configuration"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Project"]
        fullstack_app["Full Stack FastAPI Project<br/><i>Software System</i><br/>Web application for user<br/>and item management"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server<br/><i>External System</i><br/>Delivers transactional<br/>emails (password reset, etc.)"]
        sentry["Sentry<br/><i>External System</i><br/>Error monitoring and<br/>performance tracking"]
        letsencrypt["Let's Encrypt<br/><i>External System</i><br/>Automated TLS certificate<br/>provisioning via ACME"]
    end

    user -->|"Browses dashboard,<br/>manages items"| fullstack_app
    admin_user -->|"Manages users,<br/>administers system"| fullstack_app
    fullstack_app -->|"Sends transactional<br/>emails via SMTP"| smtp
    fullstack_app -->|"Reports errors and<br/>performance data"| sentry
    fullstack_app -->|"Obtains TLS<br/>certificates"| letsencrypt

    classDef person fill:#08427B,color:#fff,stroke:#073B6F
    classDef system fill:#1168BD,color:#fff,stroke:#0E5CA6
    classDef ext fill:#999999,color:#fff,stroke:#808080
    classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5,color:#444

    class user,admin_user person
    class fullstack_app system
    class smtp,sentry,letsencrypt ext
    class users,fullstack_boundary,external boundary
```

---

## L2: Container

```mermaid
graph TD
    subgraph users["Users"]
        user["User<br/><i>Person</i>"]
        admin_user["Admin / Superuser<br/><i>Person</i>"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Project"]
        traefik["Traefik<br/><i>Reverse Proxy</i><br/>Routes traffic, terminates TLS,<br/>HTTP → HTTPS redirect"]
        spa["React SPA<br/><i>Container: TypeScript, React 19,<br/>Vite, Nginx</i><br/>Single-page application served<br/>at dashboard.DOMAIN"]
        fastapi_api["FastAPI Backend<br/><i>Container: Python, FastAPI,<br/>Uvicorn</i><br/>REST API at api.DOMAIN<br/>with OpenAPI spec"]
        pg["PostgreSQL<br/><i>Container: PostgreSQL 18</i><br/>Primary relational data store<br/>for users and items"]
    end

    subgraph external["External Systems"]
        smtp["SMTP Server<br/><i>External System</i>"]
        sentry["Sentry<br/><i>External System</i>"]
        letsencrypt["Let's Encrypt<br/><i>External System</i>"]
    end

    user -->|"Browses"| traefik
    admin_user -->|"Administers"| traefik
    traefik -->|"Routes dashboard.*"| spa
    traefik -->|"Routes api.*"| fastapi_api
    spa -->|"REST API calls<br/>(JSON over HTTPS)"| fastapi_api
    fastapi_api -->|"Reads/writes via<br/>SQLAlchemy + psycopg"| pg
    fastapi_api -->|"Sends emails<br/>via SMTP"| smtp
    fastapi_api -->|"Reports errors<br/>via Sentry SDK"| sentry
    traefik -->|"ACME certificate<br/>requests"| letsencrypt

    classDef person fill:#08427B,color:#fff,stroke:#073B6F
    classDef container fill:#438DD5,color:#fff,stroke:#3C7FC0
    classDef ext fill:#999999,color:#fff,stroke:#808080
    classDef datastore fill:#438DD5,color:#fff,stroke:#3C7FC0
    classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5,color:#444

    class user,admin_user person
    class spa,fastapi_api,traefik container
    class pg datastore
    class smtp,sentry,letsencrypt ext
    class users,fullstack_boundary,external boundary
```

---

## L3: FastAPI Backend

```mermaid
graph TD
    %% SCOPE: urn:c4:container:fastapi_api

    subgraph fastapi_api["FastAPI Backend"]

        subgraph mw["Middleware"]
            %% KIND: boundary
            cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware</i><br/>Enforces allowed origins,<br/>credentials, methods"]
        end

        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes<br/><i>/login/*</i><br/>OAuth2 token, password<br/>recovery and reset"]
            %% KIND: router
            user_routes["User Routes<br/><i>/users/*</i><br/>User CRUD, signup,<br/>profile management"]
            %% KIND: router
            item_routes["Item Routes<br/><i>/items/*</i><br/>Item CRUD with<br/>ownership enforcement"]
            %% KIND: router
            util_routes["Utility Routes<br/><i>/utils/*</i><br/>Health check,<br/>test email"]
        end

        subgraph services["Services"]
            %% KIND: service_layer
            auth_deps["Auth Dependencies<br/><i>FastAPI Depends</i><br/>JWT validation, current user<br/>injection, superuser guard"]
            %% KIND: data_access
            crud_layer["CRUD Operations<br/><i>SQLModel</i><br/>User and Item create,<br/>read, update, delete"]
            %% KIND: integration
            email_utils["Email Utilities<br/><i>emails + Jinja2</i><br/>Renders and sends password<br/>reset and account emails"]
        end

        subgraph core["Core"]
            %% KIND: service_layer
            security_mod["Security Module<br/><i>PyJWT, pwdlib</i><br/>JWT token creation,<br/>password hash and verify"]
            %% KIND: boundary
            config_mod["Configuration<br/><i>Pydantic Settings</i><br/>Loads env vars, validates<br/>settings, computes DSN"]
            %% KIND: data_access
            db_engine["Database Engine<br/><i>SQLAlchemy Engine</i><br/>Connection pool,<br/>session factory"]
        end

        subgraph data["Data Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models<br/><i>SQLModel</i><br/>User, Item tables and<br/>Pydantic schemas"]
            %% KIND: data_access
            alembic_mig["Alembic Migrations<br/><i>Alembic</i><br/>Schema versioning<br/>and upgrades"]
        end
    end

    pg["PostgreSQL<br/><i>Database</i>"]
    smtp["SMTP Server<br/><i>External</i>"]
    sentry["Sentry<br/><i>External</i>"]

    cors_mw -->|"Passes requests<br/>to router"| routes
    login_routes -->|"Authenticates via"| auth_deps
    user_routes -->|"Validates user via"| auth_deps
    item_routes -->|"Validates user via"| auth_deps
    util_routes -->|"Validates superuser via"| auth_deps
    auth_deps -->|"Decodes JWT with"| security_mod
    auth_deps -->|"Obtains session from"| db_engine
    login_routes -->|"Calls"| crud_layer
    user_routes -->|"Calls"| crud_layer
    item_routes -->|"Calls"| crud_layer
    login_routes -->|"Creates tokens via"| security_mod
    login_routes -->|"Sends recovery<br/>email via"| email_utils
    crud_layer -->|"Uses"| models_layer
    crud_layer -->|"Queries via"| db_engine
    db_engine -->|"Connects to"| pg
    alembic_mig -->|"Migrates schema in"| pg
    email_utils -->|"Sends via SMTP"| smtp
    config_mod -.->|"Provides settings to<br/>all components"| fastapi_api
    security_mod -->|"Reads secret from"| config_mod
    fastapi_api -->|"Reports errors to"| sentry

    classDef component fill:#85BBF0,color:#000,stroke:#78A8D8
    classDef ext fill:#999999,color:#fff,stroke:#808080
    classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5,color:#444
    classDef groupBoundary fill:#E8F4FD,stroke:#438DD5,stroke-width:2px,color:#1168BD

    class cors_mw,login_routes,user_routes,item_routes,util_routes component
    class auth_deps,crud_layer,email_utils component
    class security_mod,config_mod,db_engine component
    class models_layer,alembic_mig component
    class pg,smtp,sentry ext
    class fastapi_api boundary
    class mw,routes,services,core,data groupBoundary
```

---

## L3: React SPA

```mermaid
graph TD
    %% SCOPE: urn:c4:container:spa

    subgraph spa["React SPA"]

        subgraph routing["Routing"]
            %% KIND: router
            router["TanStack Router<br/><i>File-based routing</i><br/>Declares routes, layout<br/>guards, lazy loading"]
        end

        subgraph state["State Management"]
            %% KIND: service_layer
            query_client["React Query Client<br/><i>TanStack Query</i><br/>Server-state cache,<br/>mutation and invalidation"]
            %% KIND: service_layer
            auth_hook["Auth Hook<br/><i>useAuth</i><br/>Login, logout, signup,<br/>current user query"]
            %% KIND: service_layer
            theme_provider["Theme Provider<br/><i>React Context</i><br/>Dark / light / system theme<br/>with localStorage persistence"]
        end

        subgraph api_layer["API Layer"]
            %% KIND: integration
            api_client["OpenAPI SDK Client<br/><i>@hey-api/openapi-ts</i><br/>Generated TypeScript client<br/>for all backend endpoints"]
        end

        subgraph features["Feature Modules"]
            %% KIND: boundary
            auth_pages["Auth Pages<br/><i>React Components</i><br/>Login, Signup, Recover<br/>Password, Reset Password"]
            %% KIND: boundary
            dashboard_feat["Dashboard<br/><i>React Components</i><br/>Welcome view with<br/>user greeting"]
            %% KIND: boundary
            items_feat["Items Management<br/><i>React Components</i><br/>CRUD with DataTable,<br/>add/edit/delete dialogs"]
            %% KIND: boundary
            settings_feat["User Settings<br/><i>React Components</i><br/>Profile, password change,<br/>account deletion"]
            %% KIND: boundary
            admin_feat["Admin Panel<br/><i>React Components</i><br/>User management table<br/>(superuser only)"]
        end

        subgraph ui["UI Layer"]
            %% KIND: boundary
            ui_lib["UI Component Library<br/><i>shadcn/ui, Radix, Tailwind</i><br/>Buttons, dialogs, forms,<br/>inputs, cards, etc."]
            %% KIND: boundary
            sidebar_comp["Sidebar Navigation<br/><i>React Components</i><br/>App navigation,<br/>user menu"]
            %% KIND: boundary
            common_comp["Common Components<br/><i>React Components</i><br/>DataTable, Logo, Footer,<br/>AuthLayout, ErrorComponent"]
        end
    end

    fastapi_api["FastAPI Backend<br/><i>REST API</i>"]

    router -->|"Renders"| auth_pages
    router -->|"Renders"| dashboard_feat
    router -->|"Renders"| items_feat
    router -->|"Renders"| settings_feat
    router -->|"Renders (superuser)"| admin_feat
    router -->|"Wraps layout with"| sidebar_comp

    auth_pages -->|"Calls login/signup via"| auth_hook
    auth_hook -->|"Uses"| api_client
    auth_hook -->|"Triggers"| query_client

    items_feat -->|"Queries/mutates via"| query_client
    settings_feat -->|"Queries/mutates via"| query_client
    admin_feat -->|"Queries/mutates via"| query_client
    dashboard_feat -->|"Reads user via"| query_client

    query_client -->|"HTTP requests via"| api_client
    api_client -->|"REST calls to<br/>/api/v1/*"| fastapi_api

    items_feat -->|"Uses"| ui_lib
    settings_feat -->|"Uses"| ui_lib
    admin_feat -->|"Uses"| ui_lib
    auth_pages -->|"Uses"| ui_lib
    items_feat -->|"Uses"| common_comp
    admin_feat -->|"Uses"| common_comp

    classDef component fill:#85BBF0,color:#000,stroke:#78A8D8
    classDef ext fill:#999999,color:#fff,stroke:#808080
    classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5,color:#444
    classDef groupBoundary fill:#E8F4FD,stroke:#438DD5,stroke-width:2px,color:#1168BD

    class router,query_client,auth_hook,theme_provider component
    class api_client component
    class auth_pages,dashboard_feat,items_feat,settings_feat,admin_feat component
    class ui_lib,sidebar_comp,common_comp component
    class fastapi_api ext
    class spa boundary
    class routing,state,api_layer,features,ui groupBoundary
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
fullstack_boundary CONTAINS [fullstack_app]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 container-level (fullstack_boundary expands) ─────────────
fullstack_boundary CONTAINS [spa, fastapi_api, pg, traefik]

%% ── L3: FastAPI Backend (scope: fastapi_api) ────────────────────
fastapi_api CONTAINS [mw, routes, services, core, data]
  mw         CONTAINS [cors_mw]
  routes     CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services   CONTAINS [auth_deps, crud_layer, email_utils]
  core       CONTAINS [security_mod, config_mod, db_engine]
  data       CONTAINS [models_layer, alembic_mig]

%% ── L3: React SPA (scope: spa) ─────────────────────────────────
spa CONTAINS [routing, state, api_layer, features, ui]
  routing    CONTAINS [router]
  state      CONTAINS [query_client, auth_hook, theme_provider]
  api_layer  CONTAINS [api_client]
  features   CONTAINS [auth_pages, dashboard_feat, items_feat, settings_feat, admin_feat]
  ui         CONTAINS [ui_lib, sidebar_comp, common_comp]
```
