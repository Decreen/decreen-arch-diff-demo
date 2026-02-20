# C4 Architecture Model — Full Stack FastAPI Template

## L1: System Context

```mermaid
---
title: "L1: System Context — Full Stack FastAPI Platform"
---
flowchart TB

  subgraph users["Users"]
    end_user["End User\n[Person]\nManages items via\nthe web dashboard"]
    admin_user["Admin User\n[Person]\nManages users, system\nconfiguration, and items"]
  end

  subgraph fullstack_boundary["Full Stack FastAPI Platform"]
    fullstack_system["Full Stack FastAPI System\n[Software System]\nWeb application for item management\nwith user authentication and admin"]
  end

  subgraph external["External Systems"]
    smtp_server["SMTP Server\n[External System]\nDelivers transactional emails\n(password recovery, welcome)"]
    sentry["Sentry\n[External System]\nError tracking and\nperformance monitoring"]
  end

  end_user -->|"Uses\n[HTTPS]"| fullstack_system
  admin_user -->|"Administers\n[HTTPS]"| fullstack_system
  fullstack_system -->|"Sends emails via\n[SMTP/TLS]"| smtp_server
  fullstack_system -->|"Reports errors to\n[HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
---
title: "L2: Container Diagram — Full Stack FastAPI Platform"
---
flowchart TB
  %% SCOPE: urn:c4:system:fullstack_system

  end_user["End User\n[Person]"]
  admin_user["Admin User\n[Person]"]

  subgraph fullstack_system["Full Stack FastAPI Platform"]
    traefik["Traefik\n[Container: Traefik 3.6]\nReverse proxy, TLS termination,\nHTTP→HTTPS redirect, load balancing"]
    react_spa["React SPA\n[Container: React 19 · TypeScript · Vite]\nSingle-page app served via Nginx;\nTanStack Router, shadcn/ui, Tailwind CSS"]
    fastapi_backend["FastAPI Backend\n[Container: Python 3.10 · FastAPI]\nREST API: JWT auth, CRUD,\nemail, health-check"]
    pg["PostgreSQL\n[Container: PostgreSQL 18]\nStores users, items,\nand migration history"]
    adminer["Adminer\n[Container: Adminer]\nLightweight database\nadministration UI"]
  end

  smtp_server["SMTP Server\n[External System]"]
  sentry["Sentry\n[External System]"]

  end_user -->|"Browses\n[HTTPS]"| traefik
  admin_user -->|"Administers\n[HTTPS]"| traefik

  traefik -->|"dashboard.*\n[HTTP :80]"| react_spa
  traefik -->|"api.*\n[HTTP :8000]"| fastapi_backend
  traefik -->|"adminer.*\n[HTTP :8080]"| adminer

  react_spa -->|"API calls\n[JSON/HTTP]"| fastapi_backend
  fastapi_backend -->|"Reads / writes\n[SQL · psycopg]"| pg
  adminer -->|"Queries\n[SQL]"| pg
  fastapi_backend -->|"Sends email\n[SMTP/TLS]"| smtp_server
  fastapi_backend -->|"Reports errors\n[HTTPS]"| sentry
```

---

## L3: FastAPI Backend — Component Diagram

```mermaid
---
title: "L3: FastAPI Backend — Components"
---
flowchart TB
  %% SCOPE: urn:c4:container:fastapi_backend

  react_spa["React SPA\n[Container]"]

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: router
    subgraph middleware["Middleware"]
      cors_mw["CORS Middleware\n[Component: Starlette CORSMiddleware]\nEnforces allowed origins,\nmethods, and headers"]
    end

    %% KIND: router
    subgraph routes["API Routes (/api/v1)"]
      login_routes["Login Routes\n[Component: FastAPI Router]\nOAuth2 token login, token test,\npassword recovery & reset"]
      users_routes["Users Routes\n[Component: FastAPI Router]\nUser CRUD, signup, profile\nupdate, password change"]
      items_routes["Items Routes\n[Component: FastAPI Router]\nItem CRUD with\nownership enforcement"]
      utils_routes["Utils Routes\n[Component: FastAPI Router]\nHealth-check endpoint,\ntest-email utility"]
      private_routes["Private Routes\n[Component: FastAPI Router]\nLocal-env-only\nuser creation"]
    end

    %% KIND: service_layer
    subgraph deps["Auth & Session Dependencies"]
      auth_deps["Auth Dependencies\n[Component: FastAPI Depends]\nJWT token validation, DB session\nprovider, permission guards"]
    end

    %% KIND: data_access
    subgraph data_access["Data Access Layer"]
      crud_layer["CRUD Operations\n[Component: Python Module]\ncreate/read/update/delete\nfor Users and Items"]
      models_layer["SQLModel Models\n[Component: SQLModel · Pydantic]\nORM table models, request/response\nschemas, validation rules"]
    end

    %% KIND: service_layer
    subgraph core["Core Services"]
      core_config["Configuration\n[Component: Pydantic Settings]\nEnvironment-based settings,\nDB URL, CORS origins"]
      core_security["Security\n[Component: PyJWT · pwdlib]\nJWT creation/validation,\nArgon2/Bcrypt password hashing"]
      db_engine["Database Engine\n[Component: SQLAlchemy Engine]\nConnection pool management,\ninitial superuser seeding"]
    end

    %% KIND: integration
    subgraph utilities["Utilities"]
      email_utils["Email Service\n[Component: emails · Jinja2]\nRender & send transactional\nemails from templates"]
      %% KIND: storage
      alembic_migrations["Alembic Migrations\n[Component: Alembic]\nVersioned database\nschema migrations"]
    end

  end

  pg["PostgreSQL\n[Container]"]
  smtp_server["SMTP Server\n[External System]"]
  sentry["Sentry\n[External System]"]

  react_spa -->|"HTTP requests"| cors_mw

  cors_mw --> login_routes
  cors_mw --> users_routes
  cors_mw --> items_routes
  cors_mw --> utils_routes
  cors_mw --> private_routes

  login_routes --> auth_deps
  users_routes --> auth_deps
  items_routes --> auth_deps
  utils_routes --> auth_deps

  login_routes --> crud_layer
  users_routes --> crud_layer
  items_routes --> models_layer

  auth_deps --> core_security
  auth_deps --> db_engine
  auth_deps --> models_layer

  crud_layer --> models_layer
  crud_layer --> core_security

  login_routes --> email_utils
  users_routes --> email_utils

  db_engine --> core_config
  core_security --> core_config
  email_utils --> core_config

  db_engine -->|"SQL"| pg
  alembic_migrations -->|"DDL"| pg
  email_utils -->|"SMTP"| smtp_server
  fastapi_backend -.->|"Sentry SDK"| sentry
```

---

## L3: React SPA — Component Diagram

```mermaid
---
title: "L3: React SPA — Components"
---
flowchart TB
  %% SCOPE: urn:c4:container:react_spa

  subgraph react_spa["React SPA"]

    %% KIND: router
    router["TanStack Router\n[Component: @tanstack/react-router]\nFile-based routing with\nauthentication guards"]

    %% KIND: boundary
    subgraph pages["Pages"]
      auth_pages["Auth Pages\n[Component: React]\nLogin, Signup,\nPassword Recovery & Reset"]
      dashboard_pages["Dashboard Pages\n[Component: React]\nHome, Items list,\nAdmin panel, Settings"]
    end

    %% KIND: service_layer
    subgraph features["Feature Components"]
      admin_components["Admin Components\n[Component: React]\nAddUser, EditUser, DeleteUser,\nuser table columns"]
      items_components["Items Components\n[Component: React]\nAddItem, EditItem, DeleteItem,\nitem table columns"]
      settings_components["Settings Components\n[Component: React]\nUserInformation, ChangePassword,\nDeleteAccount"]
    end

    %% KIND: integration
    subgraph services["Services & State"]
      api_client["API Client SDK\n[Component: @hey-api/openapi-ts]\nAuto-generated typed HTTP client;\nItemsService, UsersService, LoginService"]
      custom_hooks["Custom Hooks\n[Component: React Hooks]\nuseAuth, useCustomToast,\nuseCopyToClipboard, useMobile"]
      query_layer["React Query\n[Component: @tanstack/react-query]\nServer state cache,\noptimistic mutations"]
    end

    %% KIND: boundary
    subgraph ui["UI Layer"]
      common_components["Common Components\n[Component: React]\nDataTable, AuthLayout, Footer,\nLogo, ErrorComponent, NotFound"]
      ui_lib["shadcn/ui Library\n[Component: Radix UI · Tailwind CSS]\nButton, Dialog, Form, Table,\nSidebar, Input, Select, etc."]
      theme_provider["Theme Provider\n[Component: next-themes]\nDark / light mode\npersistence & toggle"]
    end

  end

  fastapi_backend["FastAPI Backend\n[Container]"]

  router --> auth_pages
  router --> dashboard_pages

  dashboard_pages --> admin_components
  dashboard_pages --> items_components
  dashboard_pages --> settings_components

  admin_components --> query_layer
  items_components --> query_layer
  settings_components --> query_layer
  auth_pages --> custom_hooks

  custom_hooks --> query_layer
  query_layer --> api_client

  api_client -->|"HTTP / JSON"| fastapi_backend

  admin_components --> ui_lib
  items_components --> ui_lib
  settings_components --> ui_lib
  auth_pages --> ui_lib
  common_components --> ui_lib

  dashboard_pages --> common_components
  auth_pages --> common_components
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────

users              CONTAINS [end_user, admin_user]
fullstack_boundary CONTAINS [fullstack_system]
external           CONTAINS [smtp_server, sentry]

%% ── L2: fullstack_system (zoom into the platform) ───────────────

fullstack_system   CONTAINS [traefik, react_spa, fastapi_backend, pg, adminer]

%% ── L3: fastapi_backend (zoom into the backend container) ───────

fastapi_backend    CONTAINS [middleware, routes, deps, data_access, core, utilities]
  middleware       CONTAINS [cors_mw]
  routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes, private_routes]
  deps             CONTAINS [auth_deps]
  data_access      CONTAINS [crud_layer, models_layer]
  core             CONTAINS [core_config, core_security, db_engine]
  utilities        CONTAINS [email_utils, alembic_migrations]

%% ── L3: react_spa (zoom into the frontend container) ────────────

react_spa          CONTAINS [router, pages, features, services, ui]
  pages            CONTAINS [auth_pages, dashboard_pages]
  features         CONTAINS [admin_components, items_components, settings_components]
  services         CONTAINS [api_client, custom_hooks, query_layer]
  ui               CONTAINS [common_components, ui_lib, theme_provider]
```
