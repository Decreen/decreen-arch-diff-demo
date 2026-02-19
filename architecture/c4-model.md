# C4 Architecture Model — Full Stack FastAPI Template

---

## L1: System Context

```mermaid
graph TD
  %% SCOPE: urn:c4:system:fullstack_app

  subgraph users_boundary["Users"]
    user["End User\n[Person]\nUses the web dashboard\nto manage items"]
    admin_user["Admin User\n[Person]\nManages users,\nitems, and system settings"]
  end

  subgraph fullstack_app_boundary["Full Stack FastAPI App"]
    fullstack_app["Full Stack FastAPI App\n[Software System]\nItem management platform\nwith user authentication"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n[External System]\nSends transactional emails"]
    sentry["Sentry\n[External System]\nError tracking\nand performance monitoring"]
  end

  user -->|"Browses dashboard,\nmanages own items"| fullstack_app
  admin_user -->|"Manages users,\nall items, system config"| fullstack_app
  fullstack_app -->|"Sends password reset\nand account emails"| smtp
  fullstack_app -->|"Reports errors\nand traces"| sentry
```

---

## L2: Container

```mermaid
graph TD
  %% SCOPE: urn:c4:system:fullstack_app

  subgraph users_boundary["Users"]
    user["End User\n[Person]"]
    admin_user["Admin User\n[Person]"]
  end

  subgraph fullstack_app_boundary["Full Stack FastAPI App"]
    traefik["Traefik\n[Container: Reverse Proxy]\nRoutes HTTP/HTTPS traffic,\nTLS termination via Let's Encrypt"]
    spa["React SPA\n[Container: React / Vite / TypeScript]\nSingle-page dashboard served\nvia Nginx, uses TanStack Router"]
    fastapi_backend["FastAPI Backend\n[Container: Python / FastAPI]\nREST API with JWT auth,\nserves /api/v1/* endpoints"]
    pg["PostgreSQL\n[Container: Database]\nStores users, items,\nand application data"]
    adminer["Adminer\n[Container: DB Admin Tool]\nWeb-based database\nadministration UI"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n[External System]"]
    sentry["Sentry\n[External System]"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.*"| spa
  traefik -->|"api.*"| fastapi_backend
  traefik -->|"adminer.*"| adminer
  spa -->|"REST / JSON\n/api/v1/*"| fastapi_backend
  fastapi_backend -->|"SQL\npsycopg driver"| pg
  fastapi_backend -->|"SMTP\npassword reset,\naccount emails"| smtp
  fastapi_backend -->|"HTTPS\nerror & trace reporting"| sentry
  adminer -->|"SQL"| pg
```

---

## L3: FastAPI Backend

```mermaid
graph TD
  %% SCOPE: urn:c4:container:fastapi_backend

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n[Component: Starlette]\nEnforces allowed origins,\ncredentials, methods, headers"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      api_router["API Router\n[Component: FastAPI APIRouter]\nMounts all route modules\nunder /api/v1"]

      %% KIND: router
      login_routes["Login Routes\n[Component: FastAPI Router]\nOAuth2 token login,\npassword recovery & reset"]

      %% KIND: router
      users_routes["Users Routes\n[Component: FastAPI Router]\nUser CRUD, registration,\nprofile & password update"]

      %% KIND: router
      items_routes["Items Routes\n[Component: FastAPI Router]\nItem CRUD with\nowner-based access control"]

      %% KIND: router
      utils_routes["Utils Routes\n[Component: FastAPI Router]\nHealth check,\ntest email endpoint"]

      %% KIND: router
      private_routes["Private Routes\n[Component: FastAPI Router]\nDev-only user creation\n(local environment)"]
    end

    subgraph deps["Dependencies"]
      %% KIND: service_layer
      auth_dep["Auth Dependencies\n[Component: FastAPI Depends]\nOAuth2 bearer token validation,\ncurrent user & superuser checks"]

      %% KIND: service_layer
      session_dep["Session Dependency\n[Component: FastAPI Depends]\nProvides SQLModel Session\nper-request via generator"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      core_config["Configuration\n[Component: Pydantic Settings]\nLoads env vars, validates\nsecrets, builds DB URI"]

      %% KIND: service_layer
      core_security["Security\n[Component: PyJWT / pwdlib]\nJWT token creation,\nArgon2/bcrypt password hashing"]

      %% KIND: data_access
      core_db["Database Engine\n[Component: SQLModel / SQLAlchemy]\nCreates engine, runs\nDB init & seed"]
    end

    subgraph data_access["Data Access"]
      %% KIND: data_access
      crud_layer["CRUD Layer\n[Component: Python Module]\nUser & Item create/read/\nupdate/delete operations"]

      %% KIND: storage
      models_layer["Models\n[Component: SQLModel / Pydantic]\nUser, Item DB models &\nrequest/response schemas"]
    end

    subgraph utils["Utilities"]
      %% KIND: integration
      email_utils["Email Utilities\n[Component: Python Module]\nRenders Jinja2 templates,\nsends via SMTP"]
    end

  end

  spa["React SPA"]
  pg["PostgreSQL"]
  smtp["SMTP Server"]
  sentry["Sentry"]

  spa -->|"REST / JSON"| cors_mw
  cors_mw --> api_router
  api_router --> login_routes
  api_router --> users_routes
  api_router --> items_routes
  api_router --> utils_routes
  api_router -.->|"local env only"| private_routes

  login_routes --> auth_dep
  login_routes --> core_security
  login_routes --> crud_layer
  login_routes --> email_utils

  users_routes --> auth_dep
  users_routes --> crud_layer
  users_routes --> email_utils

  items_routes --> auth_dep
  items_routes --> session_dep

  utils_routes --> auth_dep
  utils_routes --> email_utils

  private_routes --> session_dep
  private_routes --> core_security

  auth_dep --> core_security
  auth_dep --> core_config
  auth_dep --> session_dep
  session_dep --> core_db

  crud_layer --> models_layer
  crud_layer --> core_security
  crud_layer --> session_dep

  core_db --> core_config
  core_db --> models_layer
  core_security --> core_config

  email_utils --> core_config
  email_utils --> core_security
  email_utils -->|"SMTP"| smtp

  core_db -->|"SQL / psycopg"| pg
  cors_mw -.->|"error reporting"| sentry
```

---

## L3: React SPA

```mermaid
graph TD
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      router_root["Root Route\n[Component: TanStack Router]\nTop-level layout with\ndevtools & error boundary"]

      %% KIND: router
      router_layout["Authenticated Layout\n[Component: TanStack Router]\nSidebar + main content,\nredirects unauthenticated users"]

      %% KIND: router
      router_pages["Page Routes\n[Component: TanStack Router]\nLogin, Signup,\nRecover/Reset Password"]
    end

    subgraph features["Feature Components"]
      %% KIND: boundary
      admin_components["Admin Components\n[Component: React]\nUser management table,\nadd/edit/delete users"]

      %% KIND: boundary
      items_components["Items Components\n[Component: React]\nItem management table,\nadd/edit/delete items"]

      %% KIND: boundary
      settings_components["User Settings\n[Component: React]\nProfile info, change\npassword, delete account"]
    end

    subgraph shared["Shared Components"]
      %% KIND: boundary
      common_components["Common Components\n[Component: React]\nDataTable, Logo, Footer,\nAuthLayout, Error & NotFound"]

      %% KIND: boundary
      sidebar_component["Sidebar\n[Component: React]\nApp navigation,\nuser menu, logout"]

      %% KIND: boundary
      ui_lib["UI Library\n[Component: shadcn/ui]\nButton, Dialog, Form,\nTable, Input, etc."]

      %% KIND: service_layer
      theme_provider["Theme Provider\n[Component: next-themes]\nDark/light mode\nwith localStorage persistence"]
    end

    subgraph services["Services"]
      %% KIND: integration
      api_client["API Client\n[Component: Auto-generated]\nTyped SDK from OpenAPI spec,\nAxios-based HTTP requests"]

      %% KIND: service_layer
      auth_hook["Auth Hook\n[Component: React Hook]\nLogin/logout/signup mutations,\ncurrent user query"]

      %% KIND: service_layer
      query_client["Query Client\n[Component: TanStack Query]\nCache management, error\nhandling, auth interceptor"]
    end

  end

  fastapi_backend["FastAPI Backend"]
  traefik["Traefik"]

  traefik -->|"Static assets"| router_root
  router_root --> router_layout
  router_root --> router_pages

  router_layout --> admin_components
  router_layout --> items_components
  router_layout --> settings_components
  router_layout --> sidebar_component

  router_pages --> auth_hook
  router_pages --> common_components

  admin_components --> api_client
  admin_components --> query_client
  admin_components --> common_components
  admin_components --> ui_lib

  items_components --> api_client
  items_components --> query_client
  items_components --> common_components
  items_components --> ui_lib

  settings_components --> api_client
  settings_components --> query_client
  settings_components --> ui_lib

  sidebar_component --> auth_hook
  sidebar_component --> ui_lib

  auth_hook --> api_client
  auth_hook --> query_client

  api_client -->|"REST / JSON\n/api/v1/*"| fastapi_backend
  query_client --> api_client

  common_components --> ui_lib
  theme_provider --> ui_lib
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Template
%% Every subgraph and node from L1–L3 must appear here.
%% Indentation encodes parent → child nesting.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users_boundary          CONTAINS [user, admin_user]
fullstack_app_boundary  CONTAINS [fullstack_app, traefik, spa, fastapi_backend, pg, adminer]
external                CONTAINS [smtp, sentry]

%% ── L2 containers (inside fullstack_app_boundary) ───────────────
fullstack_app_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]

%% ── L3: FastAPI Backend ─────────────────────────────────────────
fastapi_backend         CONTAINS [middleware, routes, deps, core, data_access, utils]
  middleware            CONTAINS [cors_mw]
  routes                CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes, private_routes]
  deps                  CONTAINS [auth_dep, session_dep]
  core                  CONTAINS [core_config, core_security, core_db]
  data_access           CONTAINS [crud_layer, models_layer]
  utils                 CONTAINS [email_utils]

%% ── L3: React SPA ──────────────────────────────────────────────
spa                     CONTAINS [routing, features, shared, services]
  routing               CONTAINS [router_root, router_layout, router_pages]
  features              CONTAINS [admin_components, items_components, settings_components]
  shared                CONTAINS [common_components, sidebar_component, ui_lib, theme_provider]
  services              CONTAINS [api_client, auth_hook, query_client]
```
