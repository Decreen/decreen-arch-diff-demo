# C4 Architecture Model – Full Stack FastAPI Project

---

## L1: System Context

```mermaid
%% L1: System Context – Full Stack FastAPI Project
%% SCOPE: urn:c4:system:fastapi_project

flowchart TD

  subgraph users["Users"]
    user["👤 User\n[Person]\nRegular authenticated user\nwho manages their own items"]
    admin["👤 Admin\n[Person]\nSuperuser who manages\nusers and system settings"]
  end

  subgraph fastapi_project_boundary["Full Stack FastAPI Project"]
    fastapi_project["Full Stack FastAPI Project\n[Software System]\nWeb application for managing items\nwith JWT-based authentication,\nuser management, and email workflows"]
  end

  subgraph external["External Systems"]
    smtp["📧 SMTP Server\n[External System]\nSends transactional emails:\npassword recovery, new accounts"]
    sentry["📊 Sentry\n[External System]\nError monitoring and\nperformance tracing"]
  end

  user -->|"Uses web dashboard\nto manage items"| fastapi_project
  admin -->|"Manages users and\nsystem via dashboard"| fastapi_project
  fastapi_project -->|"Sends transactional\nemails via SMTP"| smtp
  fastapi_project -->|"Reports errors and\nperformance data"| sentry
```

---

## L2: Container Diagram

```mermaid
%% L2: Container – Full Stack FastAPI Project
%% SCOPE: urn:c4:system:fastapi_project

flowchart TD

  subgraph users["Users"]
    user["👤 User\n[Person]"]
    admin["👤 Admin\n[Person]"]
  end

  subgraph fastapi_project_boundary["Full Stack FastAPI Project"]
    traefik["🔀 Traefik\n[Container: Reverse Proxy]\nRoutes traffic by hostname,\nterminates TLS, HTTPS redirect"]
    spa["🌐 React SPA\n[Container: React / TypeScript / Vite]\nSingle-page dashboard UI\nfor items & user management"]
    api["⚡ FastAPI Backend\n[Container: Python / FastAPI / Uvicorn]\nREST API: authentication,\nbusiness logic, data access"]
    pg["🗄️ PostgreSQL\n[Container: PostgreSQL 18]\nStores users, items,\nand all application data"]
    adminer["🔧 Adminer\n[Container: PHP]\nWeb-based database\nadministration tool"]
  end

  subgraph external["External Systems"]
    smtp["📧 SMTP Server\n[External System]"]
    sentry["📊 Sentry\n[External System]"]
  end

  user -->|"HTTPS"| traefik
  admin -->|"HTTPS"| traefik
  traefik -->|"dashboard.* → port 80"| spa
  traefik -->|"api.* → port 8000"| api
  traefik -->|"adminer.* → port 8080"| adminer
  spa -->|"REST API calls\n[HTTPS / JSON]"| api
  api -->|"Reads & writes\n[SQL via SQLAlchemy + psycopg]"| pg
  api -->|"Sends emails\n[SMTP / TLS]"| smtp
  api -->|"Reports errors\n[HTTPS]"| sentry
  adminer -->|"Administers\n[SQL]"| pg
```

---

## L3: FastAPI Backend – Component Diagram

```mermaid
%% L3: Component – FastAPI Backend
%% SCOPE: urn:c4:container:api

flowchart TD

  spa["🌐 React SPA"]
  pg["🗄️ PostgreSQL"]
  smtp["📧 SMTP Server"]

  subgraph api["FastAPI Backend"]

    subgraph middleware["Middleware & Dependencies"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n[Starlette Middleware]\nEnforces allowed origins\nfor cross-origin requests"]
      %% KIND: boundary
      auth_dep["Auth Dependencies\n[FastAPI Depends]\nJWT token validation,\nDB session injection,\nsuperuser guard"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes\n[Router: /login, /password-recovery, /reset-password]\nOAuth2 token login, password\nrecovery and reset flows"]
      %% KIND: router
      users_routes["Users Routes\n[Router: /users]\nUser CRUD, self-service\nregistration, profile updates"]
      %% KIND: router
      items_routes["Items Routes\n[Router: /items]\nItem CRUD with\nowner-based access control"]
      %% KIND: router
      utils_routes["Utils Routes\n[Router: /utils]\nHealth check endpoint,\ntest email utility"]
      %% KIND: router
      private_routes["Private Routes\n[Router: /private]\nInternal-only user creation\n(local environment only)"]
    end

    subgraph services["Service Layer"]
      %% KIND: data_access
      crud_layer["CRUD Module\n[Service: crud.py]\nUser & item data access:\ncreate, read, update,\nauthenticate"]
      %% KIND: service_layer
      email_utils["Email Utilities\n[Service: utils.py]\nJinja2 template rendering,\nSMTP email dispatch,\npassword-reset token generation"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      security_mod["Security Module\n[Core: security.py]\nJWT creation (HS256),\npassword hashing\n(Argon2 / Bcrypt via pwdlib)"]
      %% KIND: service_layer
      config_mod["Config Module\n[Core: config.py]\nPydantic BaseSettings,\nenv-based configuration,\nall application settings"]
    end

    subgraph data["Data Layer"]
      %% KIND: data_access
      models_layer["SQLModel Models\n[Models: models.py]\nUser, Item ORM tables,\nPydantic request/response schemas,\nToken payloads"]
      %% KIND: storage
      db_engine["DB Engine\n[Data Access: db.py]\nSQLAlchemy engine,\nsession factory,\ninitial superuser seeding"]
      %% KIND: data_access
      alembic_mig["Alembic Migrations\n[Data Access: alembic/]\nDatabase schema versioning\nand migration scripts"]
    end

  end

  spa -->|"API requests\n[HTTPS / JSON]"| cors_mw
  cors_mw --> auth_dep

  auth_dep --> login_routes
  auth_dep --> users_routes
  auth_dep --> items_routes
  auth_dep --> utils_routes
  auth_dep --> private_routes

  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_utils
  users_routes --> crud_layer
  users_routes --> security_mod
  users_routes --> email_utils
  items_routes --> crud_layer
  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> db_engine
  crud_layer --> security_mod
  email_utils --> config_mod
  security_mod --> config_mod
  db_engine --> config_mod

  db_engine -->|"SQL\n[psycopg]"| pg
  alembic_mig -->|"DDL migrations"| pg
  email_utils -->|"SMTP / TLS"| smtp
```

---

## L3: React SPA – Component Diagram

```mermaid
%% L3: Component – React SPA
%% SCOPE: urn:c4:container:spa

flowchart TD

  user["👤 User / Admin"]
  api["⚡ FastAPI Backend"]

  subgraph spa["React SPA"]

    subgraph routing["Routing Layer"]
      %% KIND: router
      tanstack_router["TanStack Router\n[Router]\nFile-based routing,\nauth guards via beforeLoad,\nroute tree code generation"]
      %% KIND: router
      auth_pages["Auth Pages\n[Pages]\nLogin, Signup,\nRecover Password,\nReset Password"]
      %% KIND: router
      dashboard_pages["Dashboard Pages\n[Pages: /_layout/*]\nHome, Items, Admin,\nSettings (protected routes)"]
    end

    subgraph features["Feature Components"]
      %% KIND: service_layer
      admin_feat["Admin Feature\n[Components: Admin/]\nAddUser, EditUser, DeleteUser,\nUserActionsMenu, columns"]
      %% KIND: service_layer
      items_feat["Items Feature\n[Components: Items/]\nAddItem, EditItem, DeleteItem,\nItemActionsMenu, columns"]
      %% KIND: service_layer
      settings_feat["UserSettings Feature\n[Components: UserSettings/]\nChangePassword, UserInformation,\nDeleteAccount, DeleteConfirmation"]
    end

    subgraph services_layer["Services & State Management"]
      %% KIND: integration
      api_client["API Client SDK\n[Service: client/ – auto-generated]\nItemsService, LoginService,\nUsersService, UtilsService,\nPrivateService"]
      %% KIND: service_layer
      query_client["React Query Client\n[State: @tanstack/react-query]\nServer-state caching,\nmutation management,\nauto error handling"]
      %% KIND: service_layer
      auth_hooks["Auth Hooks\n[Hooks: useAuth]\nLogin, logout, signup mutations,\ncurrent user query,\ntoken management"]
    end

    subgraph ui["UI Layer"]
      %% KIND: boundary
      sidebar_comp["Sidebar\n[Components: Sidebar/]\nAppSidebar, Main nav,\nUser menu"]
      %% KIND: boundary
      common_comp["Common Components\n[Components: Common/]\nDataTable, AuthLayout,\nErrorComponent, NotFound,\nFooter, Logo, Appearance"]
      %% KIND: boundary
      ui_primitives["UI Primitives\n[Components: ui/ – shadcn/ui]\nButton, Dialog, Form, Input,\nTable, Tabs, Select, Card,\nDropdownMenu, etc."]
      %% KIND: service_layer
      theme_prov["Theme Provider\n[Component: theme-provider]\nDark / light mode toggle\nvia next-themes"]
    end

  end

  user -->|"Browses\n[HTTPS]"| tanstack_router

  tanstack_router --> auth_pages
  tanstack_router --> dashboard_pages

  dashboard_pages --> admin_feat
  dashboard_pages --> items_feat
  dashboard_pages --> settings_feat
  dashboard_pages --> sidebar_comp

  auth_pages --> auth_hooks
  admin_feat --> query_client
  items_feat --> query_client
  settings_feat --> query_client
  auth_hooks --> query_client

  query_client --> api_client
  api_client -->|"REST API\n[HTTPS / JSON]"| api

  admin_feat --> common_comp
  items_feat --> common_comp
  admin_feat --> ui_primitives
  items_feat --> ui_primitives
  settings_feat --> ui_primitives
  sidebar_comp --> ui_primitives
  common_comp --> ui_primitives
```

---

## Containment Map

```
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ──────────────────────────────────────────
users                    CONTAINS [user, admin]
fastapi_project_boundary CONTAINS [fastapi_project]
external                 CONTAINS [smtp, sentry]

%% ── L2 system internals (L1 → L2 expansion) ─────────────────────
fastapi_project_boundary CONTAINS [spa, api, pg, traefik, adminer]

%% ── L3: FastAPI Backend (api) ────────────────────────────────────
api        CONTAINS [middleware, routes, services, core, data]
  middleware CONTAINS [cors_mw, auth_dep]
  routes     CONTAINS [login_routes, users_routes, items_routes, utils_routes, private_routes]
  services   CONTAINS [crud_layer, email_utils]
  core       CONTAINS [security_mod, config_mod]
  data       CONTAINS [models_layer, db_engine, alembic_mig]

%% ── L3: React SPA (spa) ─────────────────────────────────────────
spa        CONTAINS [routing, features, services_layer, ui]
  routing        CONTAINS [tanstack_router, auth_pages, dashboard_pages]
  features       CONTAINS [admin_feat, items_feat, settings_feat]
  services_layer CONTAINS [api_client, query_client, auth_hooks]
  ui             CONTAINS [sidebar_comp, common_comp, ui_primitives, theme_prov]
```
