# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:platform_boundary

  subgraph users["Users"]
    user["👤 End User<br/><i>Browses items, manages<br/>own account and items</i>"]
    admin_user["👤 Administrator<br/><i>Superuser: manages all<br/>users and items</i>"]
  end

  platform_boundary["🖥️ Full Stack FastAPI Platform<br/><i>Web application for item<br/>management with authentication</i>"]

  subgraph external["External Services"]
    smtp["📧 SMTP Provider<br/><i>Transactional email<br/>delivery</i>"]
    sentry["📊 Sentry<br/><i>Error tracking &<br/>performance monitoring</i>"]
  end

  user -->|"Browses & manages items"| platform_boundary
  admin_user -->|"Manages users & system"| platform_boundary
  platform_boundary -->|"Sends emails via SMTP"| smtp
  platform_boundary -->|"Reports errors via SDK"| sentry
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:platform_boundary

  subgraph users["Users"]
    user["👤 End User"]
    admin_user["👤 Administrator"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["🔀 Traefik<br/><i>Reverse proxy<br/>TLS termination (Let's Encrypt)</i>"]
    spa["⚛️ React SPA<br/><i>Vite + React 19<br/>TanStack Router & Query<br/>shadcn/ui · Served by Nginx</i>"]
    fastapi_api["🐍 FastAPI API<br/><i>Python 3.10 · REST + OpenAPI<br/>SQLModel ORM<br/>4 Uvicorn workers</i>"]
    pg["🐘 PostgreSQL 18<br/><i>Primary relational<br/>data store</i>"]
    adminer["🔧 Adminer<br/><i>Database administration<br/>web UI</i>"]
  end

  subgraph external["External Services"]
    smtp["📧 SMTP Provider<br/><i>Transactional email</i>"]
    sentry["📊 Sentry<br/><i>Error tracking</i>"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.domain"| spa
  traefik -->|"api.domain"| fastapi_api
  traefik -->|"adminer.domain"| adminer
  spa -->|"REST / JSON over HTTPS"| fastapi_api
  fastapi_api -->|"SQL via psycopg"| pg
  fastapi_api -->|"SMTP / TLS"| smtp
  fastapi_api -->|"Sentry SDK"| sentry
  adminer -->|"SQL"| pg
```

---

## L3: FastAPI API

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_api

  spa["⚛️ React SPA"]

  subgraph fastapi_api["FastAPI API"]

    %% KIND: boundary
    subgraph mw["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware<br/>configurable origins</i>"]
    end

    %% KIND: router
    subgraph routes["Routes"]
      %% KIND: router
      login_routes["Login Routes<br/><i>POST /login/access-token<br/>POST /password-recovery/{email}<br/>POST /reset-password</i>"]
      %% KIND: router
      users_routes["Users Routes<br/><i>GET/POST /users<br/>PATCH/DELETE /users/me<br/>POST /users/signup</i>"]
      %% KIND: router
      items_routes["Items Routes<br/><i>GET/POST/PUT/DELETE<br/>/items/*</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>GET /health-check<br/>POST /test-email</i>"]
    end

    %% KIND: boundary
    subgraph deps["Dependencies"]
      %% KIND: boundary
      auth_deps["Auth Dependencies<br/><i>OAuth2 password bearer<br/>get_current_user<br/>get_current_active_superuser</i>"]
      %% KIND: data_access
      db_session["DB Session<br/><i>SQLModel Session<br/>per-request lifecycle</i>"]
    end

    %% KIND: data_access
    subgraph data_layer["Data Access"]
      %% KIND: data_access
      crud_layer["CRUD Layer<br/><i>create_user · update_user<br/>authenticate · create_item</i>"]
      %% KIND: data_access
      models["SQLModel Models<br/><i>User · Item tables<br/>Pydantic request/response schemas</i>"]
    end

    %% KIND: service_layer
    subgraph core["Core"]
      %% KIND: service_layer
      security["Security<br/><i>JWT HS256 via PyJWT<br/>Argon2 + Bcrypt hashing</i>"]
      %% KIND: service_layer
      config["Configuration<br/><i>pydantic-settings<br/>env-based (.env)</i>"]
      %% KIND: storage
      db_engine["Database Engine<br/><i>SQLAlchemy create_engine<br/>PostgreSQL connection pool</i>"]
    end

    %% KIND: integration
    email_utils["Email Utilities<br/><i>Jinja2 + MJML templates<br/>password reset · welcome email</i>"]

    %% KIND: data_access
    alembic["Alembic Migrations<br/><i>Schema versioning<br/>upgrade / downgrade</i>"]

  end

  pg["🐘 PostgreSQL 18"]
  smtp["📧 SMTP Provider"]

  spa -->|"HTTP / JSON"| cors_mw

  cors_mw --> login_routes
  cors_mw --> users_routes
  cors_mw --> items_routes
  cors_mw --> utils_routes

  login_routes --> auth_deps
  login_routes --> crud_layer
  login_routes --> email_utils
  users_routes --> auth_deps
  users_routes --> crud_layer
  items_routes --> auth_deps
  items_routes --> crud_layer
  utils_routes --> email_utils

  auth_deps --> security
  auth_deps --> db_session
  db_session --> db_engine

  crud_layer --> models
  crud_layer --> security
  crud_layer --> db_session

  db_engine -->|"SQL"| pg
  alembic --> db_engine
  email_utils -->|"SMTP"| smtp
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    %% KIND: router
    subgraph routing["Routing"]
      %% KIND: router
      router["TanStack Router<br/><i>File-based routing<br/>beforeLoad guards<br/>code-splitting</i>"]
    end

    %% KIND: boundary
    subgraph pages["Pages / Features"]
      %% KIND: boundary
      dashboard_page["Dashboard<br/><i>/ — welcome view<br/>current user info</i>"]
      %% KIND: boundary
      items_page["Items Feature<br/><i>/items — DataTable<br/>Add · Edit · Delete dialogs</i>"]
      %% KIND: boundary
      admin_page["Admin Feature<br/><i>/admin — user management<br/>superuser-only route guard</i>"]
      %% KIND: boundary
      settings_page["Settings Feature<br/><i>/settings — profile<br/>password · danger zone</i>"]
      %% KIND: boundary
      auth_pages["Auth Pages<br/><i>Login · Signup<br/>Password recovery & reset</i>"]
    end

    %% KIND: service_layer
    subgraph state_mgmt["State Management"]
      %% KIND: service_layer
      query_client["TanStack Query<br/><i>Server state cache<br/>mutations · error handling<br/>401 auto-redirect</i>"]
      %% KIND: service_layer
      theme_ctx["Theme Provider<br/><i>Dark / light / system<br/>React Context + localStorage</i>"]
      %% KIND: service_layer
      auth_hooks["Auth Hooks<br/><i>useAuth: login · logout · signup<br/>JWT token management</i>"]
    end

    %% KIND: integration
    subgraph client_layer["API Client"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>@hey-api/openapi-ts<br/>Axios transport<br/>auto-generated from spec</i>"]
    end

    %% KIND: boundary
    subgraph ui["UI Components"]
      %% KIND: boundary
      ui_lib["shadcn / Radix UI<br/><i>Tailwind CSS 4<br/>forms · dialogs · tables<br/>buttons · inputs · toasts</i>"]
      %% KIND: boundary
      sidebar_comp["Sidebar<br/><i>App navigation<br/>user dropdown menu</i>"]
    end

  end

  fastapi_api["🐍 FastAPI API"]

  router --> dashboard_page
  router --> items_page
  router --> admin_page
  router --> settings_page
  router --> auth_pages

  dashboard_page --> query_client
  items_page --> query_client
  admin_page --> query_client
  settings_page --> query_client
  settings_page --> auth_hooks

  auth_pages --> auth_hooks
  auth_hooks --> query_client
  query_client --> api_client
  api_client -->|"HTTP / JSON"| fastapi_api

  dashboard_page --> ui_lib
  items_page --> ui_lib
  admin_page --> ui_lib
  settings_page --> ui_lib
  auth_pages --> ui_lib

  dashboard_page --> sidebar_comp
  items_page --> sidebar_comp
  admin_page --> sidebar_comp
  settings_page --> sidebar_comp
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% ═══════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi_api, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment (FastAPI API) ────────────────────
fastapi_api CONTAINS [mw, routes, deps, data_layer, core, email_utils, alembic]
  mw         CONTAINS [cors_mw]
  routes     CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  deps       CONTAINS [auth_deps, db_session]
  data_layer CONTAINS [crud_layer, models]
  core       CONTAINS [security, config, db_engine]

%% ── L2→L3 internal containment (React SPA) ─────────────────────
spa CONTAINS [routing, pages, state_mgmt, client_layer, ui]
  routing      CONTAINS [router]
  pages        CONTAINS [dashboard_page, items_page, admin_page, settings_page, auth_pages]
  state_mgmt   CONTAINS [query_client, theme_ctx, auth_hooks]
  client_layer CONTAINS [api_client]
  ui           CONTAINS [ui_lib, sidebar_comp]
```
