# C4 Architecture — Full Stack FastAPI Platform

> Auto-generated C4 model. Node IDs are **stable** across all diagram levels.
> Each diagram declares its scope element via a `%% SCOPE:` comment.

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:context:fullstack-fastapi-platform

  subgraph users[" Users "]
    user(["End User<br/><i>Person</i><br/>Browses dashboard, manages own items"])
    admin_user(["Administrator<br/><i>Person</i><br/>Manages all users and system settings"])
  end

  subgraph platform_boundary[" Full Stack FastAPI Platform "]
    platform["Full Stack FastAPI Platform<br/><i>Software System</i><br/>Web application for user &amp; item management<br/>with JWT auth, email notifications, and admin tools"]
  end

  subgraph external[" External Systems "]
    smtp["SMTP Server<br/><i>External System</i><br/>Transactional email delivery"]
    sentry["Sentry<br/><i>External System</i><br/>Error monitoring &amp; tracing"]
  end

  user -- "Manages items via web dashboard" --> platform
  admin_user -- "Manages users &amp; settings" --> platform
  platform -- "Sends transactional emails" --> smtp
  platform -- "Reports runtime errors" --> sentry

  classDef person fill:#08427b,color:#fff,stroke:#073b6e
  classDef system fill:#1168bd,color:#fff,stroke:#0e5ca8
  classDef ext fill:#999999,color:#fff,stroke:#888888

  class user,admin_user person
  class platform system
  class smtp,sentry ext
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi-platform

  subgraph users[" Users "]
    user(["End User"])
    admin_user(["Administrator"])
  end

  subgraph platform_boundary[" Full Stack FastAPI Platform "]
    traefik["Traefik<br/><i>Container: Reverse Proxy</i><br/>TLS termination, host-based routing<br/>Let's Encrypt certificates"]
    spa["React SPA<br/><i>Container: React 19 + TanStack Router/Query</i><br/>Single-page app served by Nginx<br/>Tailwind CSS, shadcn/ui"]
    fastapi_backend["FastAPI Backend<br/><i>Container: Python / FastAPI</i><br/>REST API with JWT authentication<br/>Pydantic validation, OpenAPI spec"]
    pg[("PostgreSQL<br/><i>Container: Database</i><br/>Stores users, items<br/>Managed via Alembic migrations")]
  end

  subgraph external[" External Systems "]
    smtp["SMTP Server"]
    sentry["Sentry"]
  end

  user -- "HTTPS" --> traefik
  admin_user -- "HTTPS" --> traefik
  traefik -- "dashboard.* — serves SPA" --> spa
  traefik -- "api.* — proxies API" --> fastapi_backend
  spa -- "REST/JSON via /api/v1" --> fastapi_backend
  fastapi_backend -- "SQL via psycopg" --> pg
  fastapi_backend -- "SMTP" --> smtp
  fastapi_backend -- "HTTPS" --> sentry

  classDef person fill:#08427b,color:#fff,stroke:#073b6e
  classDef container fill:#438dd5,color:#fff,stroke:#3c7fc0
  classDef database fill:#438dd5,color:#fff,stroke:#3c7fc0
  classDef ext fill:#999999,color:#fff,stroke:#888888

  class user,admin_user person
  class traefik,spa,fastapi_backend container
  class pg database
  class smtp,sentry ext
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi-backend

  subgraph fastapi_backend[" FastAPI Backend "]

    subgraph mw[" Middleware "]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware</i><br/>Enforces allowed origins"]
      %% KIND: boundary
      auth_deps["Auth Dependencies<br/><i>api/deps.py</i><br/>OAuth2 bearer token extraction<br/>get_current_user, get_db"]
    end

    subgraph routes[" API Routes (/api/v1) "]
      %% KIND: router
      login_routes["Login Routes<br/><i>/login</i><br/>OAuth2 token, password recovery &amp; reset"]
      %% KIND: router
      users_routes["Users Routes<br/><i>/users</i><br/>CRUD, signup, profile, superuser mgmt"]
      %% KIND: router
      items_routes["Items Routes<br/><i>/items</i><br/>Item CRUD for authenticated users"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>/utils</i><br/>Health check, test email"]
    end

    subgraph services[" Service Layer "]
      %% KIND: service_layer
      crud_layer["CRUD Operations<br/><i>crud.py</i><br/>create/update/authenticate user<br/>create item"]
      %% KIND: service_layer
      security_mod["Security Module<br/><i>core/security.py</i><br/>JWT creation (HS256)<br/>Argon2/Bcrypt password hashing"]
      %% KIND: integration
      email_utils["Email Utilities<br/><i>utils.py</i><br/>Jinja2 template rendering<br/>SMTP sending, token generation"]
    end

    subgraph data_access[" Data Access "]
      %% KIND: data_access
      models_layer["SQLModel Models<br/><i>models.py</i><br/>User, Item, Token, TokenPayload"]
      %% KIND: storage
      db_engine["Database Engine<br/><i>core/db.py</i><br/>SQLAlchemy engine, session factory<br/>init_db seed logic"]
      %% KIND: boundary
      config["Configuration<br/><i>core/config.py</i><br/>Pydantic Settings from env vars"]
    end

  end

  pg[("PostgreSQL")]
  smtp["SMTP Server"]
  sentry["Sentry"]

  cors_mw --> auth_deps
  auth_deps --> security_mod
  auth_deps --> db_engine

  login_routes --> crud_layer
  login_routes --> email_utils
  users_routes --> crud_layer
  items_routes --> crud_layer
  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> security_mod
  crud_layer --> db_engine

  email_utils --> smtp
  db_engine --> pg
  security_mod -.-> config
  db_engine -.-> config
  cors_mw -.-> config

  classDef component fill:#85bbf0,color:#000,stroke:#78a8d8
  classDef ext fill:#999999,color:#fff,stroke:#888888

  class cors_mw,auth_deps component
  class login_routes,users_routes,items_routes,utils_routes component
  class crud_layer,security_mod,email_utils component
  class models_layer,db_engine,config component
  class pg,smtp,sentry ext
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  subgraph spa[" React SPA "]

    subgraph routing[" Routing &amp; State Management "]
      %% KIND: router
      router["TanStack Router<br/><i>File-based routing</i><br/>routeTree.gen.ts<br/>beforeLoad auth guards"]
      %% KIND: service_layer
      query_client["TanStack Query Client<br/><i>Data fetching &amp; caching</i><br/>QueryCache, MutationCache<br/>Auto error handling"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>Generated SDK (sdk.gen.ts)</i><br/>Axios-based, fully typed<br/>LoginService, UsersService, ItemsService"]
    end

    subgraph features[" Feature Pages "]
      %% KIND: boundary
      auth_pages["Auth Pages<br/><i>Login, Signup</i><br/>Password Recovery, Reset Password"]
      %% KIND: boundary
      dashboard_layout["Dashboard Layout<br/><i>_layout.tsx</i><br/>Sidebar, Header, Footer<br/>Protected route wrapper"]
      %% KIND: boundary
      admin_pages["Admin Pages<br/><i>_layout/admin.tsx</i><br/>User CRUD, superuser only<br/>DataTable with actions"]
      %% KIND: boundary
      items_pages["Items Pages<br/><i>_layout/items.tsx</i><br/>Item CRUD<br/>DataTable, Add/Edit/Delete dialogs"]
      %% KIND: boundary
      settings_pages["Settings Pages<br/><i>_layout/settings.tsx</i><br/>User profile, change password<br/>Account deletion, appearance"]
    end

    subgraph shared[" Shared Layer "]
      %% KIND: service_layer
      auth_hook["useAuth Hook<br/><i>hooks/useAuth.ts</i><br/>Login, logout, signup mutations<br/>Token management via localStorage"]
      %% KIND: boundary
      ui_components["UI Components<br/><i>shadcn/ui + Radix primitives</i><br/>Buttons, Forms, Dialogs, Tables<br/>DataTable, Sidebar, Pagination"]
      %% KIND: boundary
      theme_provider["Theme Provider<br/><i>next-themes</i><br/>Dark / Light mode toggle<br/>Persisted in localStorage"]
    end

  end

  fastapi_backend["FastAPI Backend"]

  router --> auth_pages
  router --> dashboard_layout
  dashboard_layout --> admin_pages
  dashboard_layout --> items_pages
  dashboard_layout --> settings_pages

  auth_pages --> auth_hook
  auth_hook --> query_client
  admin_pages --> query_client
  items_pages --> query_client
  settings_pages --> query_client

  query_client --> api_client
  api_client -- "REST/JSON over HTTPS" --> fastapi_backend

  auth_pages --> ui_components
  admin_pages --> ui_components
  items_pages --> ui_components
  settings_pages --> ui_components
  dashboard_layout --> ui_components
  theme_provider -.-> ui_components

  classDef component fill:#85bbf0,color:#000,stroke:#78a8d8
  classDef ext fill:#999999,color:#fff,stroke:#888888

  class router,query_client,api_client component
  class auth_pages,dashboard_layout,admin_pages,items_pages,settings_pages component
  class auth_hook,ui_components,theme_provider component
  class fastapi_backend ext
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers L1, L2, and all L3 diagrams
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users             CONTAINS [user, admin_user]
platform_boundary CONTAINS [spa, fastapi_backend, pg, traefik]
external          CONTAINS [smtp, sentry]

%% ── L2 containers (platform_boundary expanded) ────────────────────
%%   traefik        (leaf — no internal structure)
%%   pg             (leaf — no internal structure)
%%   fastapi_backend → decomposed in L3
%%   spa             → decomposed in L3

%% ── L3: fastapi_backend internal containment ──────────────────────
fastapi_backend   CONTAINS [mw, routes, services, data_access]
  mw              CONTAINS [cors_mw, auth_deps]
  routes          CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services        CONTAINS [crud_layer, security_mod, email_utils]
  data_access     CONTAINS [models_layer, db_engine, config]

%% ── L3: spa internal containment ──────────────────────────────────
spa               CONTAINS [routing, features, shared]
  routing         CONTAINS [router, query_client, api_client]
  features        CONTAINS [auth_pages, dashboard_layout, admin_pages, items_pages, settings_pages]
  shared          CONTAINS [auth_hook, ui_components, theme_provider]
```
