# C4 Architecture Model — Full Stack FastAPI Template

> Auto-generated C4 architecture model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI Template project.

---

## L1: System Context

Shows actors, the platform as a whole, and external dependencies.

```mermaid
%% L1: System Context
flowchart TB

  subgraph users["Users"]
    user["👤 User\n[Person]\nRegistered application user\nwho manages personal items"]
    admin["👤 Admin\n[Person]\nSuperuser who manages\nall users and items"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    fullstack_platform["Full Stack FastAPI Platform\n[Software System]\nWeb application for user\nregistration, authentication,\nand item management"]
  end

  subgraph external["External Systems"]
    smtp["📧 SMTP Email Service\n[External System]\nDelivers transactional emails:\npassword recovery, new accounts"]
    sentry["📊 Sentry\n[External System]\nError monitoring\nand performance tracking"]
  end

  user -->|"Browses dashboard,\nmanages own items"| fullstack_platform
  admin -->|"Manages all users\nand items via dashboard"| fullstack_platform
  fullstack_platform -->|"Sends transactional\nemails via SMTP"| smtp
  fullstack_platform -->|"Reports errors\nand traces"| sentry
```

---

## L2: Container Diagram

Zooms into the platform boundary to show all deployable containers and their interactions.

```mermaid
%% L2: Container Diagram
flowchart TB

  subgraph users["Users"]
    user["👤 User\n[Person]"]
    admin["👤 Admin\n[Person]"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik\n[Container: Traefik v3]\nReverse proxy, TLS termination,\nroute-based load balancing"]
    spa["React SPA\n[Container: React 19 / Vite / Nginx]\nSingle-page application providing\nthe user dashboard UI"]
    fastapi_backend["FastAPI Backend\n[Container: Python / FastAPI / Uvicorn]\nREST API serving /api/v1/*\nwith JWT authentication"]
    pg["PostgreSQL\n[Container: PostgreSQL 18]\nStores users, items,\nand application data"]
    adminer["Adminer\n[Container: Adminer]\nDatabase administration\nweb interface (dev/staging)"]
  end

  subgraph external["External Systems"]
    smtp["📧 SMTP Email Service\n[External System]"]
    sentry["📊 Sentry\n[External System]"]
  end

  user -->|"HTTPS"| traefik
  admin -->|"HTTPS"| traefik
  traefik -->|"dashboard.*\nHTTP/80"| spa
  traefik -->|"api.*\nHTTP/8000"| fastapi_backend
  traefik -->|"adminer.*\nHTTP/8080"| adminer
  spa -->|"REST API calls\n/api/v1/*"| fastapi_backend
  fastapi_backend -->|"SQL via psycopg\nport 5432"| pg
  fastapi_backend -->|"Sends emails"| smtp
  fastapi_backend -->|"Reports errors"| sentry
  adminer -->|"SQL\nport 5432"| pg
```

---

## L3: FastAPI Backend — Component Diagram

Zooms into the FastAPI Backend container to show its internal components.

```mermaid
%% L3: FastAPI Backend — Component Diagram
%% SCOPE: urn:c4:container:fastapi_backend
flowchart TB

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n[Component]\nEnforces allowed origins,\nmethods, headers"]
      %% KIND: boundary
      auth_deps["Auth Dependencies\n[Component]\nOAuth2 bearer token extraction,\nJWT validation, current user injection"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes\n[Component: /login/*, /password-recovery/*, /reset-password/]\nOAuth2 token login,\npassword recovery & reset"]
      %% KIND: router
      user_routes["User Routes\n[Component: /users/*]\nCRUD for users, self-service\nprofile and password updates, signup"]
      %% KIND: router
      item_routes["Item Routes\n[Component: /items/*]\nCRUD for items with\nownership-based access control"]
      %% KIND: router
      utils_routes["Utils Routes\n[Component: /utils/*]\nHealth check endpoint\nand test email trigger"]
    end

    subgraph services["Service Layer"]
      %% KIND: service_layer
      crud_layer["CRUD Operations\n[Component]\nUser & Item create/read/\nupdate/delete with SQLModel"]
      %% KIND: service_layer
      security_mod["Security Module\n[Component]\nJWT token creation (HS256),\nArgon2/Bcrypt password hashing"]
      %% KIND: integration
      email_utils["Email Utilities\n[Component]\nJinja2 email templates,\nSMTP dispatch for recovery\nand account emails"]
      %% KIND: service_layer
      config_mod["Configuration\n[Component]\nPydantic Settings with\nenv-file loading and validation"]
    end

    subgraph data_access["Data Access"]
      %% KIND: data_access
      models_layer["SQLModel Models\n[Component]\nUser, Item table definitions\nand Pydantic request/response schemas"]
      %% KIND: storage
      db_engine["Database Engine\n[Component]\nSQLAlchemy engine, session\nfactory, DB initialization"]
    end

  end

  spa["React SPA"] -->|"HTTP requests\n/api/v1/*"| cors_mw
  cors_mw --> auth_deps
  auth_deps --> routes

  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_utils
  user_routes --> crud_layer
  user_routes --> security_mod
  user_routes --> email_utils
  item_routes --> crud_layer
  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> db_engine
  crud_layer --> security_mod
  security_mod --> config_mod
  email_utils --> config_mod
  db_engine --> config_mod
  db_engine -->|"SQL via psycopg"| pg

  pg["PostgreSQL"]
  smtp["SMTP Email Service"]
  sentry["Sentry"]

  email_utils -->|"SMTP"| smtp
  config_mod -.->|"Sentry DSN"| sentry
```

---

## L3: React SPA — Component Diagram

Zooms into the React SPA container to show its internal components.

```mermaid
%% L3: React SPA — Component Diagram
%% SCOPE: urn:c4:container:spa
flowchart TB

  subgraph spa["React SPA"]

    %% KIND: router
    routing["TanStack Router\n[Component]\nFile-based routing with\nauth guards and lazy loading"]

    subgraph pages["Pages"]
      %% KIND: boundary
      auth_pages["Auth Pages\n[Component]\nLogin, Signup,\nRecover Password, Reset Password"]
      %% KIND: boundary
      dashboard_page["Dashboard Page\n[Component]\nMain landing page\nafter authentication"]
      %% KIND: boundary
      admin_page["Admin Page\n[Component]\nUser management table\nwith CRUD actions"]
      %% KIND: boundary
      items_page["Items Page\n[Component]\nItem management table\nwith CRUD actions"]
      %% KIND: boundary
      settings_page["Settings Page\n[Component]\nUser profile, password\nchange, account deletion"]
    end

    subgraph features["Feature Components"]
      %% KIND: service_layer
      admin_features["Admin Components\n[Component]\nAddUser, EditUser, DeleteUser,\nUserActionsMenu, columns"]
      %% KIND: service_layer
      items_features["Items Components\n[Component]\nAddItem, EditItem, DeleteItem,\nItemActionsMenu, columns"]
      %% KIND: service_layer
      settings_features["Settings Components\n[Component]\nUserInformation, ChangePassword,\nDeleteAccount, DeleteConfirmation"]
      %% KIND: service_layer
      common_features["Common Components\n[Component]\nDataTable, AuthLayout, Footer,\nLogo, ErrorComponent, NotFound"]
    end

    subgraph services_layer["Client Services"]
      %% KIND: integration
      api_client["OpenAPI Client\n[Component]\nAuto-generated TypeScript SDK\n(ItemsService, UsersService,\nLoginService, UtilsService)"]
      %% KIND: service_layer
      hooks_layer["React Hooks\n[Component]\nuseAuth, useCustomToast,\nuseMobile, useCopyToClipboard"]
    end

    %% KIND: boundary
    ui_layer["shadcn/ui Components\n[Component]\nButton, Dialog, DataTable, Form,\nSidebar, Tabs, and other\nTailwind CSS primitives"]

  end

  user["User"] -->|"HTTPS"| routing
  admin["Admin"] -->|"HTTPS"| routing

  routing --> auth_pages
  routing --> dashboard_page
  routing --> admin_page
  routing --> items_page
  routing --> settings_page

  auth_pages --> hooks_layer
  admin_page --> admin_features
  items_page --> items_features
  settings_page --> settings_features

  admin_features --> api_client
  admin_features --> hooks_layer
  items_features --> api_client
  items_features --> hooks_layer
  settings_features --> api_client
  settings_features --> hooks_layer
  auth_pages --> api_client
  common_features --> hooks_layer

  admin_features --> ui_layer
  items_features --> ui_layer
  settings_features --> ui_layer
  common_features --> ui_layer
  pages --> common_features

  api_client -->|"REST /api/v1/*"| fastapi_backend

  fastapi_backend["FastAPI Backend"]
```

---

## L3: Traefik — Component Diagram

Zooms into the Traefik reverse proxy container.

```mermaid
%% L3: Traefik — Component Diagram
%% SCOPE: urn:c4:container:traefik
flowchart TB

  subgraph traefik["Traefik"]

    %% KIND: router
    entrypoint_http["HTTP Entrypoint\n[Component]\nListens on port 80,\nredirects to HTTPS"]
    %% KIND: router
    entrypoint_https["HTTPS Entrypoint\n[Component]\nListens on port 443,\nTLS termination"]
    %% KIND: service_layer
    https_redirect["HTTPS Redirect Middleware\n[Component]\nRedirects all HTTP\ntraffic to HTTPS"]
    %% KIND: service_layer
    cert_resolver["Let's Encrypt Resolver\n[Component]\nAutomatic TLS certificate\nprovisioning via ACME"]
    %% KIND: router
    frontend_router["Frontend Router\n[Component]\nRoutes dashboard.* to\nReact SPA on port 80"]
    %% KIND: router
    backend_router["Backend Router\n[Component]\nRoutes api.* to\nFastAPI on port 8000"]
    %% KIND: router
    adminer_router["Adminer Router\n[Component]\nRoutes adminer.* to\nAdminer on port 8080"]

  end

  user["User"] -->|"HTTP/HTTPS"| entrypoint_http
  user -->|"HTTPS"| entrypoint_https
  entrypoint_http --> https_redirect
  https_redirect --> entrypoint_https
  entrypoint_https --> cert_resolver
  entrypoint_https --> frontend_router
  entrypoint_https --> backend_router
  entrypoint_https --> adminer_router

  frontend_router -->|"port 80"| spa["React SPA"]
  backend_router -->|"port 8000"| fastapi_backend["FastAPI Backend"]
  adminer_router -->|"port 8080"| adminer["Adminer"]
```

---

## Containment Map

Every subgraph and node across all levels, expressed as parent CONTAINS children.
Indentation shows nesting depth.

```text
%% ── L1 top-level groups ─────────────────────────────────────────────
users              CONTAINS [user, admin]
platform_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment ──────────────────────────────────────
fastapi_backend    CONTAINS [middleware, routes, services, data_access]
  middleware         CONTAINS [cors_mw, auth_deps]
  routes             CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services           CONTAINS [crud_layer, security_mod, email_utils, config_mod]
  data_access        CONTAINS [models_layer, db_engine]

spa                CONTAINS [routing, pages, features, services_layer, ui_layer]
  pages              CONTAINS [auth_pages, dashboard_page, admin_page, items_page, settings_page]
  features           CONTAINS [admin_features, items_features, settings_features, common_features]
  services_layer     CONTAINS [api_client, hooks_layer]

traefik            CONTAINS [entrypoint_http, entrypoint_https, https_redirect, cert_resolver, frontend_router, backend_router, adminer_router]
```

---

## Node ID Cross-Reference

Stable IDs used across all diagram levels:

| Stable ID            | L1 | L2 | L3 Scope            | Description                        |
|----------------------|----|----|---------------------|------------------------------------|
| `user`               | ✓  | ✓  | —                   | Regular application user           |
| `admin`              | ✓  | ✓  | —                   | Superuser / administrator          |
| `spa`                | —  | ✓  | `spa` (scope)       | React SPA frontend                 |
| `fastapi_backend`    | —  | ✓  | `fastapi_backend` (scope) | FastAPI REST API backend     |
| `pg`                 | —  | ✓  | referenced          | PostgreSQL database                |
| `traefik`            | —  | ✓  | `traefik` (scope)   | Traefik reverse proxy              |
| `adminer`            | —  | ✓  | referenced          | Adminer DB admin tool              |
| `smtp`               | ✓  | ✓  | referenced          | External SMTP email service        |
| `sentry`             | ✓  | ✓  | referenced          | External error monitoring          |
| `fullstack_platform` | ✓  | —  | —                   | System-level node (L1 only)        |
