# C4 Architecture Model — FastAPI Full Stack Platform

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the FastAPI Full Stack Template.

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:platform

  subgraph users["Users"]
    user["User<br/>[Person]<br/>Manages personal items<br/>via the web dashboard"]
    admin_user["Admin<br/>[Person]<br/>Manages users and<br/>system configuration"]
  end

  subgraph platform_boundary["FastAPI Full Stack Platform"]
    platform["FastAPI Full Stack Platform<br/>[Software System]<br/>Full-stack web application for<br/>user and item management"]
  end

  subgraph external["External Systems"]
    smtp_ext["SMTP Service<br/>[External System]<br/>Transactional email delivery"]
    sentry_ext["Sentry<br/>[External System]<br/>Error tracking and performance monitoring"]
  end

  user -->|"Uses via browser"| platform
  admin_user -->|"Administers via browser"| platform
  platform -->|"Sends emails via SMTP"| smtp_ext
  platform -->|"Reports errors via HTTPS"| sentry_ext
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:platform

  subgraph users["Users"]
    user["User<br/>[Person]"]
    admin_user["Admin<br/>[Person]"]
  end

  subgraph platform_boundary["FastAPI Full Stack Platform"]
    traefik["Traefik<br/>[Container: Traefik v3.6]<br/>Reverse proxy, TLS termination,<br/>host-based routing"]
    spa["React SPA<br/>[Container: React 19 / Nginx]<br/>Single-page application<br/>with dashboard UI"]
    fastapi_backend["FastAPI Backend<br/>[Container: Python / FastAPI]<br/>REST API with JWT auth,<br/>serves /api/v1"]
    pg["PostgreSQL<br/>[Container: PostgreSQL 18]<br/>Primary relational database"]
    adminer["Adminer<br/>[Container: Adminer]<br/>Database administration UI"]
  end

  subgraph external["External Systems"]
    smtp_ext["SMTP Service<br/>[External System]<br/>Email delivery"]
    sentry_ext["Sentry<br/>[External System]<br/>Error tracking"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"Routes dashboard.*"| spa
  traefik -->|"Routes api.*"| fastapi_backend
  traefik -->|"Routes adminer.*"| adminer
  spa -->|"REST API calls<br/>[JSON/HTTPS]"| fastapi_backend
  fastapi_backend -->|"SQL queries<br/>[psycopg driver]"| pg
  fastapi_backend -->|"Sends email<br/>[SMTP]"| smtp_ext
  fastapi_backend -->|"Reports errors<br/>[HTTPS]"| sentry_ext
  adminer -->|"SQL"| pg
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_backend

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph mw["Middleware / Dependencies"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/>[Starlette Middleware]<br/>Cross-origin request handling"]
      %% KIND: boundary
      auth_deps["Auth Dependencies<br/>[FastAPI Depends]<br/>OAuth2 bearer token extraction,<br/>JWT validation, user resolution"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes<br/>[Router: /login]<br/>Access token issuance,<br/>password recovery and reset"]
      %% KIND: router
      user_routes["User Routes<br/>[Router: /users]<br/>User CRUD, self-service signup,<br/>profile and password management"]
      %% KIND: router
      item_routes["Item Routes<br/>[Router: /items]<br/>Item CRUD operations<br/>scoped to authenticated users"]
      %% KIND: router
      utils_routes["Utils Routes<br/>[Router: /utils]<br/>Health check endpoint,<br/>test email trigger"]
    end

    subgraph services["Service Layer"]
      %% KIND: service_layer
      crud_layer["CRUD Layer<br/>[Service: crud.py]<br/>User and item persistence,<br/>authentication logic"]
      %% KIND: service_layer
      email_utils["Email Utilities<br/>[Service: utils.py]<br/>Jinja2 template rendering,<br/>SMTP sending, reset tokens"]
    end

    subgraph data_access["Data Access"]
      %% KIND: data_access
      models_layer["Models<br/>[SQLModel / Pydantic]<br/>User, Item, Token schemas<br/>and DB table mappings"]
      %% KIND: data_access
      db_engine["Database Engine<br/>[SQLAlchemy Core]<br/>Connection pool and<br/>session management"]
      %% KIND: storage
      alembic_mig["Alembic Migrations<br/>[Migration Runner]<br/>Schema version control<br/>with revision history"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      security_mod["Security Module<br/>[Core: security.py]<br/>JWT creation (HS256),<br/>Argon2 / Bcrypt hashing"]
      %% KIND: service_layer
      config_mod["Configuration<br/>[Core: Pydantic Settings]<br/>Environment-based settings,<br/>secret validation"]
    end

  end

  pg["PostgreSQL"]
  smtp_ext["SMTP Service"]
  sentry_ext["Sentry"]

  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  utils_routes --> auth_deps
  auth_deps --> security_mod
  auth_deps --> db_engine
  login_routes --> crud_layer
  login_routes --> email_utils
  user_routes --> crud_layer
  item_routes --> crud_layer
  utils_routes --> email_utils
  crud_layer --> models_layer
  crud_layer --> security_mod
  email_utils --> config_mod
  models_layer --> db_engine
  db_engine --> pg
  email_utils --> smtp_ext
  alembic_mig --> db_engine
  security_mod --> config_mod
  cors_mw -.->|"wraps all routes"| routes
  config_mod -.->|"initialises"| sentry_ext
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      router["TanStack Router<br/>[Router: file-based]<br/>Declarative routing with<br/>auth guard beforeLoad"]
    end

    subgraph features["Features / Pages"]
      %% KIND: boundary
      auth_pages["Auth Pages<br/>[Feature]<br/>Login, Signup,<br/>Password Recovery, Reset"]
      %% KIND: boundary
      dashboard_page["Dashboard<br/>[Feature]<br/>Main overview page<br/>for authenticated users"]
      %% KIND: boundary
      items_feature["Items Management<br/>[Feature]<br/>Items data table with<br/>add, edit, delete"]
      %% KIND: boundary
      admin_feature["Admin Panel<br/>[Feature]<br/>User management with<br/>add, edit, delete"]
      %% KIND: boundary
      settings_feature["User Settings<br/>[Feature]<br/>Profile info, password change,<br/>account deletion"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["OpenAPI Client<br/>[Service: @hey-api/openapi-ts]<br/>Auto-generated typed API client<br/>with Axios transport"]
      %% KIND: service_layer
      auth_hook["Auth Hook<br/>[Service: useAuth]<br/>Login/logout mutations,<br/>JWT localStorage, user query"]
    end

    subgraph ui_layer["UI Layer"]
      %% KIND: boundary
      ui_components["UI Components<br/>[UI: shadcn/ui + Radix]<br/>Buttons, dialogs, tables,<br/>forms, sidebar, etc."]
      %% KIND: boundary
      theme_provider["Theme Provider<br/>[UI: next-themes]<br/>Dark / light mode toggle"]
      %% KIND: boundary
      layout_shell["App Layout<br/>[UI: _layout.tsx]<br/>Sidebar, header, footer,<br/>protected route wrapper"]
    end

  end

  fastapi_backend["FastAPI Backend"]

  router --> auth_pages
  router --> dashboard_page
  router --> items_feature
  router --> admin_feature
  router --> settings_feature
  router --> layout_shell
  auth_pages --> auth_hook
  auth_pages --> ui_components
  dashboard_page --> api_client
  dashboard_page --> ui_components
  items_feature --> api_client
  items_feature --> ui_components
  admin_feature --> api_client
  admin_feature --> ui_components
  settings_feature --> api_client
  settings_feature --> ui_components
  auth_hook --> api_client
  api_client -->|"REST / JSON"| fastapi_backend
  layout_shell --> ui_components
  ui_components --> theme_provider
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users             CONTAINS [user, admin_user]
platform_boundary CONTAINS [platform, traefik, spa, fastapi_backend, pg, adminer]
external          CONTAINS [smtp_ext, sentry_ext]

%% ── L2→L3 internal containment ──────────────────────────────────
fastapi_backend CONTAINS [mw, routes, services, data_access, core]
  mw            CONTAINS [cors_mw, auth_deps]
  routes        CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services      CONTAINS [crud_layer, email_utils]
  data_access   CONTAINS [models_layer, db_engine, alembic_mig]
  core          CONTAINS [security_mod, config_mod]

spa CONTAINS [routing, features, services_layer, ui_layer]
  routing        CONTAINS [router]
  features       CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
  services_layer CONTAINS [api_client, auth_hook]
  ui_layer       CONTAINS [ui_components, theme_provider, layout_shell]
```
