# C4 Architecture Model – FastAPI Full Stack Platform

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for every container with internal structure.

---

## L1: System Context

```mermaid
---
title: "L1: System Context – FastAPI Full Stack Platform"
---
flowchart TB
  %% SCOPE: urn:c4:system:fullstack_platform

  subgraph users["Users"]
    user["End User\n[Person]\nUses the web application\nto manage items"]
    admin_user["Admin User\n[Person]\nManages users and\nplatform configuration"]
  end

  subgraph fullstack_boundary["FastAPI Full Stack Platform"]
    fullstack_system["FastAPI Full Stack Platform\n[Software System]\nFull-stack web application providing\nuser management and item CRUD"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Service\n[External System]\nMailgun / SendGrid / etc.\nTransactional email delivery"]
    sentry["Sentry\n[External System]\nError monitoring\nand performance tracing"]
    letsencrypt["Let's Encrypt\n[External System]\nAutomatic TLS certificate\nissuance via ACME"]
  end

  user -->|"Uses web app\n[HTTPS]"| fullstack_system
  admin_user -->|"Manages users & config\n[HTTPS]"| fullstack_system
  fullstack_system -->|"Sends transactional email\n[SMTP/TLS]"| smtp
  fullstack_system -->|"Reports errors & traces\n[HTTPS]"| sentry
  fullstack_system -->|"Obtains TLS certificates\n[ACME]"| letsencrypt
```

---

## L2: Container

```mermaid
---
title: "L2: Container – FastAPI Full Stack Platform"
---
flowchart TB
  %% SCOPE: urn:c4:system:fullstack_platform

  subgraph users["Users"]
    user["End User\n[Person]"]
    admin_user["Admin User\n[Person]"]
  end

  subgraph fullstack_boundary["FastAPI Full Stack Platform"]
    traefik["Traefik\n[Container: Traefik 3.6]\nReverse proxy, TLS termination,\nHTTP routing"]
    spa["Frontend SPA\n[Container: React 19 / Nginx]\nSingle-page application\nserving the web UI"]
    fastapi_backend["FastAPI Backend\n[Container: Python / FastAPI]\nREST API providing business logic,\nauthentication, and data access"]
    pg["PostgreSQL\n[Container: PostgreSQL 18]\nRelational database storing\nusers, items, and sessions"]
    adminer["Adminer\n[Container: Adminer]\nWeb-based database\nadministration tool"]
    prestart["Prestart\n[Container: Python / Alembic]\nRuns DB migrations and\nbootstraps first superuser"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Service\n[External System]\nTransactional email delivery"]
    sentry["Sentry\n[External System]\nError monitoring"]
    letsencrypt["Let's Encrypt\n[External System]\nTLS certificates"]
  end

  user -->|"Browses\n[HTTPS]"| traefik
  admin_user -->|"Manages\n[HTTPS]"| traefik
  traefik -->|"Routes to dashboard.*\n[HTTP]"| spa
  traefik -->|"Routes to api.*\n[HTTP]"| fastapi_backend
  traefik -->|"Routes to adminer.*\n[HTTP]"| adminer
  traefik -->|"Obtains certs\n[ACME]"| letsencrypt
  spa -->|"API calls\n[HTTP/JSON]"| fastapi_backend
  fastapi_backend -->|"Reads/writes data\n[SQL via psycopg]"| pg
  fastapi_backend -->|"Sends email\n[SMTP]"| smtp
  fastapi_backend -->|"Reports errors\n[HTTPS]"| sentry
  adminer -->|"Browses schema & data\n[SQL]"| pg
  prestart -->|"Runs migrations\n[SQL via Alembic]"| pg
```

---

## L3: FastAPI Backend

```mermaid
---
title: "L3: Component – FastAPI Backend"
---
flowchart TB
  %% SCOPE: urn:c4:container:fastapi_backend

  spa["Frontend SPA"]
  pg["PostgreSQL"]
  smtp["SMTP Service"]
  sentry["Sentry"]

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n[Middleware]\nHandles cross-origin\nrequest policies"]
      sentry_mw["Sentry Integration\n[Middleware]\nError capture and\nperformance tracing"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      api_router["API Router\n[Router]\nCentral route registration\nunder /api/v1"]
      login_routes["Login Routes\n[Routes]\nOAuth2 token issuance,\npassword recovery & reset"]
      users_routes["Users Routes\n[Routes]\nUser CRUD, profile,\nsignup, admin management"]
      items_routes["Items Routes\n[Routes]\nItem CRUD with\nownership enforcement"]
      utils_routes["Utils Routes\n[Routes]\nHealth check and\ntest email endpoints"]
    end

    subgraph deps["Dependencies"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies\n[Dependency Injection]\nJWT decode, user extraction,\nsuperuser gate"]
      db_session["DB Session Provider\n[Dependency Injection]\nPer-request SQLModel Session\nfrom engine"]
    end

    subgraph services["Business Logic"]
      %% KIND: service_layer
      crud_layer["CRUD Layer\n[Service]\ncreate_user, update_user,\nauthenticate, create_item"]
    end

    subgraph models_layer["Data Models"]
      %% KIND: data_access
      domain_models["Domain Models\n[SQLModel]\nUser, Item entities\nwith relationships"]
      api_schemas["API Schemas\n[Pydantic]\nRequest/response DTOs:\nUserCreate, ItemPublic, etc."]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      core_config["Config\n[Pydantic Settings]\nEnvironment-driven\napplication settings"]
      core_db["Database Engine\n[Data Access]\nSQLAlchemy engine\nand init_db bootstrap"]
      core_security["Security\n[Service]\nJWT token creation,\nArgon2/Bcrypt hashing"]
    end

    subgraph utils["Utilities"]
      %% KIND: integration
      email_utils["Email Utilities\n[Utility]\nJinja2 templating,\nSMTP email dispatch"]
    end

  end

  spa -->|"HTTP/JSON"| api_router
  api_router --> login_routes
  api_router --> users_routes
  api_router --> items_routes
  api_router --> utils_routes

  login_routes --> auth_deps
  login_routes --> crud_layer
  login_routes --> core_security
  login_routes --> email_utils

  users_routes --> auth_deps
  users_routes --> crud_layer
  users_routes --> email_utils

  items_routes --> auth_deps
  items_routes --> db_session
  items_routes --> domain_models

  utils_routes --> email_utils

  auth_deps --> core_security
  auth_deps --> db_session
  db_session --> core_db
  crud_layer --> domain_models
  crud_layer --> core_security
  crud_layer --> db_session

  core_db --> pg
  core_config -.->|"configures"| core_db
  core_config -.->|"configures"| core_security
  email_utils --> smtp
  sentry_mw --> sentry
```

---

## L3: Frontend SPA

```mermaid
---
title: "L3: Component – Frontend SPA"
---
flowchart TB
  %% SCOPE: urn:c4:container:spa

  traefik["Traefik"]
  fastapi_backend["FastAPI Backend"]

  subgraph spa["Frontend SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router\n[Router]\nFile-based routing\nwith code splitting"]
      root_layout["Root Layout\n[Layout]\nSidebar, header, footer,\nauthentication guard"]
    end

    subgraph pages["Pages"]
      %% KIND: boundary
      dashboard_page["Dashboard Page\n[Page]\nHome overview"]
      items_page["Items Page\n[Page]\nItem management view"]
      admin_page["Admin Page\n[Page]\nUser administration\n(superuser only)"]
      settings_page["Settings Page\n[Page]\nProfile and password\nmanagement"]
      auth_pages["Auth Pages\n[Page]\nLogin, Signup,\nPassword Recovery & Reset"]
    end

    subgraph features["Feature Components"]
      %% KIND: service_layer
      items_features["Items Components\n[Feature]\nAddItem, EditItem,\nDeleteItem, columns"]
      admin_features["Admin Components\n[Feature]\nAddUser, EditUser,\nDeleteUser, columns"]
      settings_features["Settings Components\n[Feature]\nUserInformation,\nChangePassword, DeleteAccount"]
    end

    subgraph shared["Shared UI"]
      %% KIND: boundary
      ui_library["shadcn/ui Library\n[UI Kit]\nRadix-based primitives\nwith Tailwind styling"]
      data_table["DataTable\n[Component]\nReusable table built on\nTanStack Table"]
      sidebar_component["Sidebar\n[Component]\nApp navigation\nand user menu"]
    end

    subgraph api_layer["API Layer"]
      %% KIND: integration
      api_client["OpenAPI Client\n[Generated Client]\nAxios-based SDK from\nOpenAPI spec"]
    end

    subgraph hooks_layer["Hooks"]
      %% KIND: service_layer
      use_auth["useAuth\n[Hook]\nToken management\nand auth state"]
      use_custom_toast["useCustomToast\n[Hook]\nToast notification\nhelpers"]
    end

  end

  traefik -->|"Serves static assets\n[HTTP]"| tanstack_router
  tanstack_router --> root_layout
  root_layout --> dashboard_page
  root_layout --> items_page
  root_layout --> admin_page
  root_layout --> settings_page
  tanstack_router --> auth_pages

  items_page --> items_features
  admin_page --> admin_features
  settings_page --> settings_features

  items_features --> api_client
  admin_features --> api_client
  settings_features --> api_client
  auth_pages --> api_client

  items_features --> data_table
  admin_features --> data_table
  items_features --> ui_library
  admin_features --> ui_library
  settings_features --> ui_library
  auth_pages --> ui_library

  use_auth --> api_client
  auth_pages --> use_auth

  api_client -->|"REST API calls\n[HTTP/JSON]"| fastapi_backend
```

---

## L3: Traefik

```mermaid
---
title: "L3: Component – Traefik"
---
flowchart TB
  %% SCOPE: urn:c4:container:traefik

  user["End User"]
  admin_user["Admin User"]
  spa["Frontend SPA"]
  fastapi_backend["FastAPI Backend"]
  adminer["Adminer"]
  letsencrypt["Let's Encrypt"]

  subgraph traefik["Traefik"]

    subgraph entrypoints["Entrypoints"]
      %% KIND: router
      http_ep["HTTP Entrypoint\n[Entrypoint]\nPort 80, redirects\nto HTTPS"]
      https_ep["HTTPS Entrypoint\n[Entrypoint]\nPort 443, TLS\ntermination"]
    end

    subgraph routers_layer["Routers"]
      %% KIND: router
      frontend_router["Frontend Router\n[Router]\nHost: dashboard.*"]
      backend_router["Backend Router\n[Router]\nHost: api.*"]
      adminer_router["Adminer Router\n[Router]\nHost: adminer.*"]
    end

    subgraph middlewares_layer["Middlewares"]
      %% KIND: boundary
      https_redirect["HTTPS Redirect\n[Middleware]\nHTTP to HTTPS\nredirection"]
    end

    subgraph tls_layer["TLS"]
      %% KIND: integration
      acme_resolver["ACME Resolver\n[Certificate Resolver]\nLet's Encrypt TLS\ncertificate management"]
    end

  end

  user -->|"HTTPS"| https_ep
  admin_user -->|"HTTPS"| https_ep
  user -->|"HTTP"| http_ep
  http_ep --> https_redirect
  https_redirect --> https_ep

  https_ep --> frontend_router
  https_ep --> backend_router
  https_ep --> adminer_router

  frontend_router -->|"HTTP"| spa
  backend_router -->|"HTTP"| fastapi_backend
  adminer_router -->|"HTTP"| adminer

  acme_resolver -->|"ACME challenge\n[HTTPS]"| letsencrypt
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers L1, L2, and all L3 diagrams
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
fullstack_boundary CONTAINS [fullstack_system, traefik, spa, fastapi_backend, pg, adminer, prestart]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 container detail (same fullstack_boundary, expanded) ─────
%% fullstack_boundary already listed above with all containers

%% ── L3: fastapi_backend ─────────────────────────────────────────
fastapi_backend CONTAINS [middleware, routes, deps, services, models_layer, core, utils]
  middleware    CONTAINS [cors_mw, sentry_mw]
  routes        CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes]
  deps          CONTAINS [auth_deps, db_session]
  services      CONTAINS [crud_layer]
  models_layer  CONTAINS [domain_models, api_schemas]
  core          CONTAINS [core_config, core_db, core_security]
  utils         CONTAINS [email_utils]

%% ── L3: spa ─────────────────────────────────────────────────────
spa CONTAINS [routing, pages, features, shared, api_layer, hooks_layer]
  routing       CONTAINS [tanstack_router, root_layout]
  pages         CONTAINS [dashboard_page, items_page, admin_page, settings_page, auth_pages]
  features      CONTAINS [items_features, admin_features, settings_features]
  shared        CONTAINS [ui_library, data_table, sidebar_component]
  api_layer     CONTAINS [api_client]
  hooks_layer   CONTAINS [use_auth, use_custom_toast]

%% ── L3: traefik ─────────────────────────────────────────────────
traefik CONTAINS [entrypoints, routers_layer, middlewares_layer, tls_layer]
  entrypoints       CONTAINS [http_ep, https_ep]
  routers_layer     CONTAINS [frontend_router, backend_router, adminer_router]
  middlewares_layer CONTAINS [https_redirect]
  tls_layer         CONTAINS [acme_resolver]
```

---

## Stable ID Reference

| ID | Name | Appears On |
|----|------|-----------|
| `user` | End User | L1, L2, L3:traefik |
| `admin_user` | Admin User | L1, L2, L3:traefik |
| `fullstack_system` | Platform (abstract) | L1 |
| `fullstack_boundary` | Platform boundary | L1, L2 |
| `traefik` | Traefik | L2, L3:traefik (scope), L3:spa |
| `spa` | Frontend SPA | L2, L3:spa (scope), L3:fastapi_backend |
| `fastapi_backend` | FastAPI Backend | L2, L3:fastapi_backend (scope), L3:spa |
| `pg` | PostgreSQL | L2, L3:fastapi_backend |
| `adminer` | Adminer | L2, L3:traefik |
| `prestart` | Prestart | L2 |
| `smtp` | SMTP Service | L1, L2, L3:fastapi_backend |
| `sentry` | Sentry | L1, L2, L3:fastapi_backend |
| `letsencrypt` | Let's Encrypt | L1, L2, L3:traefik |
