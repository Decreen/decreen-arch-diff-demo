# C4 Architecture Model — Full Stack FastAPI Template

> Auto-generated C4 model describing the system at L1 (context),
> L2 (container), and L3 (component) levels.

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    user(["User [Person]\nBrowses the dashboard\nand manages items"])
    admin_user(["Admin [Person]\nSuperuser managing\nusers and system"])
  end

  subgraph system_boundary["Full Stack FastAPI Template"]
    system["Full Stack FastAPI Template\n[Software System]\nWeb application with authentication,\nitem CRUD, and admin dashboard"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n[External System]\nEmail delivery service"]
    sentry["Sentry\n[External System]\nError tracking and monitoring"]
    letsencrypt["Let's Encrypt\n[External System]\nTLS certificate authority"]
  end

  user -->|"Uses [HTTPS]"| system
  admin_user -->|"Administers [HTTPS]"| system
  system -->|"Sends emails [SMTP/TLS]"| smtp
  system -->|"Reports errors [HTTPS]"| sentry
  system -->|"Obtains TLS certs [ACME]"| letsencrypt

  classDef person fill:#08427B,color:#fff,stroke:#052E56
  classDef systemBox fill:#1168BD,color:#fff,stroke:#0B4884
  classDef ext fill:#999999,color:#fff,stroke:#6B6B6B

  class user,admin_user person
  class system systemBox
  class smtp,sentry,letsencrypt ext
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    user(["User [Person]"])
    admin_user(["Admin [Person]"])
  end

  subgraph system_boundary["Full Stack FastAPI Template"]
    traefik_proxy["Traefik\n[Container: Traefik v3]\nReverse proxy &\nTLS termination"]
    spa["React SPA\n[Container: React 19 / Nginx]\nSingle-page application\nserved as static files"]
    fastapi_api["FastAPI Backend\n[Container: Python / FastAPI]\nREST API serving /api/v1\nwith JWT authentication"]
    pg[("PostgreSQL\n[Container: PostgreSQL 18]\nPrimary relational database")]
    adminer["Adminer\n[Container: Adminer]\nDatabase admin web UI"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n[External System]"]
    sentry["Sentry\n[External System]"]
    letsencrypt["Let's Encrypt\n[External System]"]
  end

  user -->|"HTTPS requests"| traefik_proxy
  admin_user -->|"HTTPS requests"| traefik_proxy
  traefik_proxy -->|"Proxies [HTTP :80]"| spa
  traefik_proxy -->|"Proxies [HTTP :8000]"| fastapi_api
  traefik_proxy -->|"Proxies [HTTP :8080]"| adminer
  spa -->|"API calls [REST/JSON]"| fastapi_api
  fastapi_api -->|"Reads & writes [SQL via psycopg]"| pg
  fastapi_api -->|"Sends emails [SMTP/TLS]"| smtp
  fastapi_api -->|"Reports errors [HTTPS]"| sentry
  adminer -->|"DB management [SQL]"| pg
  traefik_proxy -->|"Obtains TLS certs [ACME]"| letsencrypt

  classDef person fill:#08427B,color:#fff,stroke:#052E56
  classDef container fill:#438DD5,color:#fff,stroke:#2E6295
  classDef ext fill:#999999,color:#fff,stroke:#6B6B6B

  class user,admin_user person
  class traefik_proxy,spa,fastapi_api,adminer container
  class pg container
  class smtp,sentry,letsencrypt ext
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_api

  subgraph fastapi_api["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n[Component: Starlette]\nCross-origin request filtering"]
    end

    subgraph deps["Dependencies"]
      %% KIND: service_layer
      auth_dep["Auth Dependency\n[Component: FastAPI Depends]\nOAuth2 bearer + JWT validation\nExtracts current user"]
      %% KIND: data_access
      db_dep["DB Session Dependency\n[Component: FastAPI Depends]\nSQLModel session-per-request\ninjection"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes\n[Component: APIRouter]\n/login/access-token\n/password-recovery\n/reset-password"]
      %% KIND: router
      users_routes["Users Routes\n[Component: APIRouter]\n/users CRUD\n/signup, /users/me"]
      %% KIND: router
      items_routes["Items Routes\n[Component: APIRouter]\n/items CRUD"]
      %% KIND: router
      utils_routes["Utils Routes\n[Component: APIRouter]\n/health-check\n/test-email"]
      %% KIND: router
      private_routes["Private Routes\n[Component: APIRouter]\nLocal-env-only\nuser creation"]
    end

    subgraph business["Business Logic"]
      %% KIND: service_layer
      crud_layer["CRUD Layer\n[Component: Python module]\ncreate/read/update/delete\nauthenticate"]
      %% KIND: integration
      email_utils["Email Utilities\n[Component: Python module]\nJinja2 template rendering\nSMTP sending"]
    end

    subgraph data_layer["Data Access"]
      %% KIND: data_access
      models_layer["SQLModel Models\n[Component: SQLModel]\nUser & Item table models\n+ Pydantic schemas"]
      %% KIND: storage
      db_engine["Database Engine\n[Component: SQLAlchemy]\nConnection pool\nand engine"]
      %% KIND: data_access
      alembic_mig["Alembic Migrations\n[Component: Alembic]\nSchema versioning\nand upgrades"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      security_core["Security\n[Component: PyJWT / pwdlib]\nJWT creation & validation\nArgon2/Bcrypt hashing"]
      %% KIND: boundary
      config_core["Configuration\n[Component: Pydantic Settings]\nEnvironment variables\nand secrets"]
    end

  end

  pg[("PostgreSQL\n[Container]")]
  smtp["SMTP Server\n[External]"]
  sentry["Sentry\n[External]"]

  cors_mw --> login_routes
  cors_mw --> users_routes
  cors_mw --> items_routes
  cors_mw --> utils_routes
  cors_mw --> private_routes

  login_routes --> db_dep
  users_routes --> auth_dep
  users_routes --> db_dep
  items_routes --> auth_dep
  items_routes --> db_dep
  utils_routes --> auth_dep
  private_routes --> db_dep

  login_routes --> crud_layer
  users_routes --> crud_layer
  items_routes --> crud_layer
  private_routes --> crud_layer
  login_routes --> email_utils
  users_routes --> email_utils
  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> security_core
  auth_dep --> security_core
  auth_dep --> models_layer
  db_dep --> db_engine

  db_engine -->|"SQL via psycopg"| pg
  alembic_mig -->|"Manages schema"| pg
  email_utils -->|"SMTP/TLS"| smtp
  fastapi_api -.->|"Reports errors"| sentry

  classDef comp fill:#85BBF0,color:#000,stroke:#5D82A8
  classDef ext fill:#999999,color:#fff,stroke:#6B6B6B
  classDef db fill:#438DD5,color:#fff,stroke:#2E6295

  class cors_mw,auth_dep,db_dep comp
  class login_routes,users_routes,items_routes,utils_routes,private_routes comp
  class crud_layer,email_utils comp
  class models_layer,db_engine,alembic_mig comp
  class security_core,config_core comp
  class pg db
  class smtp,sentry ext
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      router_layer["TanStack Router\n[Component: @tanstack/router]\nFile-based routing\nwith auth guards"]
    end

    subgraph features["Pages & Features"]
      %% KIND: boundary
      auth_pages["Auth Pages\n[Component: React]\nLogin, Signup\nRecover & Reset Password"]
      %% KIND: boundary
      dashboard_page["Dashboard\n[Component: React]\nWelcome page\nwith user greeting"]
      %% KIND: boundary
      items_feature["Items Management\n[Component: React]\nCRUD data table with\nadd / edit / delete dialogs"]
      %% KIND: boundary
      admin_feature["Admin Panel\n[Component: React]\nUser management\n(superuser only)"]
      %% KIND: boundary
      settings_feature["Settings\n[Component: React]\nProfile, password\nand account deletion"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["API Client\n[Component: @hey-api/openapi-ts]\nAuto-generated from OpenAPI\nAxios HTTP transport"]
      %% KIND: service_layer
      auth_hook["Auth Hook\n[Component: React Hook]\nuseAuth — login, logout\ntoken management"]
    end

    subgraph ui_layer["UI Layer"]
      %% KIND: boundary
      ui_components["UI Components\n[Component: shadcn/ui + Radix]\nButtons, dialogs, tables\nforms, sidebar, pagination"]
      %% KIND: boundary
      theme_provider["Theme Provider\n[Component: next-themes]\nDark / light mode\ntoggle and persistence"]
    end

  end

  fastapi_api["FastAPI Backend\n[Container]"]

  router_layer --> auth_pages
  router_layer --> dashboard_page
  router_layer --> items_feature
  router_layer --> admin_feature
  router_layer --> settings_feature

  auth_pages --> api_client
  auth_pages --> auth_hook
  items_feature --> api_client
  admin_feature --> api_client
  settings_feature --> api_client
  dashboard_page --> auth_hook

  auth_pages --> ui_components
  dashboard_page --> ui_components
  items_feature --> ui_components
  admin_feature --> ui_components
  settings_feature --> ui_components

  api_client -->|"REST/JSON over HTTPS"| fastapi_api

  classDef comp fill:#85BBF0,color:#000,stroke:#5D82A8
  classDef container fill:#438DD5,color:#fff,stroke:#2E6295

  class router_layer,auth_pages,dashboard_page,items_feature comp
  class admin_feature,settings_feature comp
  class api_client,auth_hook comp
  class ui_components,theme_provider comp
  class fastapi_api container
```

---

## Containment Map

```text
%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [user, admin_user]
system_boundary  CONTAINS [system]
external         CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 system decomposition ─────────────────────────────────────
system_boundary  CONTAINS [traefik_proxy, spa, fastapi_api, pg, adminer]

%% ── L3: FastAPI Backend internal containment ────────────────────
fastapi_api      CONTAINS [middleware, deps, routes, business, data_layer, core]
  middleware     CONTAINS [cors_mw]
  deps           CONTAINS [auth_dep, db_dep]
  routes         CONTAINS [login_routes, users_routes, items_routes, utils_routes, private_routes]
  business       CONTAINS [crud_layer, email_utils]
  data_layer     CONTAINS [models_layer, db_engine, alembic_mig]
  core           CONTAINS [security_core, config_core]

%% ── L3: React SPA internal containment ──────────────────────────
spa              CONTAINS [routing, features, services_layer, ui_layer]
  routing        CONTAINS [router_layer]
  features       CONTAINS [auth_pages, dashboard_page, items_feature, admin_feature, settings_feature]
  services_layer CONTAINS [api_client, auth_hook]
  ui_layer       CONTAINS [ui_components, theme_provider]
```
