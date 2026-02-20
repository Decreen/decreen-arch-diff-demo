# C4 Architecture Model — Full Stack FastAPI Template

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    app_user["App User<br/><i>Person</i><br/>Uses the web dashboard to<br/>manage items and account"]
    admin_user["Admin User<br/><i>Person</i><br/>Manages users and system<br/>configuration"]
  end

  subgraph fullstack_boundary["Full Stack FastAPI Platform"]
    fullstack_system["Full Stack FastAPI<br/><i>Software System</i><br/>Web application for user and<br/>item management with<br/>JWT authentication"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server<br/><i>External System</i><br/>Sends transactional emails<br/>(password recovery, account creation)"]
    sentry["Sentry<br/><i>External System</i><br/>Error tracking and<br/>performance monitoring"]
  end

  app_user -->|"Uses web dashboard<br/>[HTTPS]"| fullstack_system
  admin_user -->|"Manages users and items<br/>[HTTPS]"| fullstack_system
  fullstack_system -->|"Sends emails<br/>[SMTP/TLS]"| smtp
  fullstack_system -->|"Reports errors<br/>[HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  app_user["App User<br/><i>Person</i>"]
  admin_user["Admin User<br/><i>Person</i>"]

  subgraph fullstack_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik<br/><i>Container: Reverse Proxy</i><br/>Routes HTTP/HTTPS traffic,<br/>TLS termination, load balancing"]
    spa["React SPA<br/><i>Container: Nginx + React</i><br/>Single-page application built<br/>with React, TanStack Router,<br/>shadcn/ui, served by Nginx"]
    fastapi_backend["FastAPI Backend<br/><i>Container: Python / FastAPI</i><br/>REST API providing user mgmt,<br/>item CRUD, auth, email services"]
    pg["PostgreSQL<br/><i>Container: Database</i><br/>Stores users, items,<br/>and application data"]
    prestart["Prestart<br/><i>Container: Init Job</i><br/>Runs DB migrations (Alembic)<br/>and seeds initial superuser"]
    adminer["Adminer<br/><i>Container: Web UI</i><br/>Database administration tool"]
  end

  smtp["SMTP Server<br/><i>External System</i>"]
  sentry["Sentry<br/><i>External System</i>"]

  app_user -->|"Browses dashboard<br/>[HTTPS]"| traefik
  admin_user -->|"Manages system<br/>[HTTPS]"| traefik

  traefik -->|"Serves frontend<br/>[HTTP :80]"| spa
  traefik -->|"Proxies API calls<br/>[HTTP :8000]"| fastapi_backend
  traefik -->|"Proxies DB admin<br/>[HTTP :8080]"| adminer

  spa -->|"Calls REST API<br/>[HTTP/JSON]"| fastapi_backend

  fastapi_backend -->|"Reads/writes data<br/>[SQL via psycopg]"| pg
  fastapi_backend -->|"Sends emails<br/>[SMTP/TLS]"| smtp
  fastapi_backend -->|"Reports errors<br/>[HTTPS]"| sentry

  prestart -->|"Runs migrations &<br/>seeds data [SQL]"| pg

  adminer -->|"Administers<br/>[SQL]"| pg
```

---

## L3: FastAPI Backend — Component Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi-backend

  spa["React SPA<br/><i>Container</i>"]
  pg["PostgreSQL<br/><i>Container</i>"]
  smtp["SMTP Server<br/><i>External</i>"]
  sentry["Sentry<br/><i>External</i>"]

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Component: Starlette</i><br/>Handles cross-origin requests"]
      %% KIND: boundary
      oauth2_mw["OAuth2 Bearer<br/><i>Component: FastAPI Security</i><br/>Extracts JWT from requests"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes<br/><i>Component: APIRouter</i><br/>POST /login/access-token<br/>POST /password-recovery<br/>POST /reset-password"]
      %% KIND: router
      user_routes["User Routes<br/><i>Component: APIRouter</i><br/>CRUD /users, /users/me<br/>POST /users/signup"]
      %% KIND: router
      item_routes["Item Routes<br/><i>Component: APIRouter</i><br/>CRUD /items"]
      %% KIND: router
      util_routes["Util Routes<br/><i>Component: APIRouter</i><br/>GET /health-check<br/>POST /test-email"]
    end

    subgraph deps["Dependencies"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies<br/><i>Component: FastAPI Depends</i><br/>get_current_user,<br/>get_current_active_superuser"]
      %% KIND: data_access
      session_dep["Session Dependency<br/><i>Component: FastAPI Depends</i><br/>Provides SQLModel Session"]
    end

    subgraph services["Service Layer"]
      %% KIND: service_layer
      crud_service["CRUD Service<br/><i>Component: Python Module</i><br/>create/update/authenticate users,<br/>create items"]
      %% KIND: service_layer
      security_service["Security Service<br/><i>Component: Python Module</i><br/>JWT creation, password<br/>hashing (Argon2/Bcrypt)"]
      %% KIND: service_layer
      email_service["Email Service<br/><i>Component: Python Module</i><br/>Render templates, send<br/>transactional emails"]
    end

    subgraph models_layer["Data Models"]
      %% KIND: data_access
      sqlmodels["SQLModel Models<br/><i>Component: SQLModel</i><br/>User, Item tables and<br/>Pydantic schemas"]
    end

    subgraph config_layer["Configuration"]
      %% KIND: integration
      app_config["Settings<br/><i>Component: Pydantic Settings</i><br/>Loads env vars, DB URI,<br/>SMTP, Sentry config"]
      %% KIND: data_access
      db_engine["DB Engine<br/><i>Component: SQLAlchemy</i><br/>Connection pool to PostgreSQL"]
    end
  end

  spa -->|"REST API calls<br/>[HTTP/JSON]"| login_routes
  spa -->|"REST API calls<br/>[HTTP/JSON]"| user_routes
  spa -->|"REST API calls<br/>[HTTP/JSON]"| item_routes

  login_routes --> auth_deps
  login_routes --> crud_service
  login_routes --> security_service
  login_routes --> email_service

  user_routes --> auth_deps
  user_routes --> crud_service

  item_routes --> auth_deps
  item_routes --> session_dep

  util_routes --> email_service

  auth_deps --> session_dep
  auth_deps --> security_service
  auth_deps --> sqlmodels

  session_dep --> db_engine

  crud_service --> session_dep
  crud_service --> security_service
  crud_service --> sqlmodels

  email_service -->|"Sends emails<br/>[SMTP]"| smtp

  db_engine --> app_config
  db_engine -->|"SQL queries<br/>[psycopg]"| pg

  security_service --> app_config

  app_config -->|"Reports errors"| sentry
```

---

## L3: React SPA — Component Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  app_user["App User<br/><i>Person</i>"]
  admin_user["Admin User<br/><i>Person</i>"]
  fastapi_backend["FastAPI Backend<br/><i>Container</i>"]

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router<br/><i>Component: React Router</i><br/>File-based routing, layout<br/>routes, auth guards"]
      %% KIND: router
      route_tree["Route Tree<br/><i>Component: Auto-generated</i><br/>login, signup, recover-password,<br/>reset-password, _layout/*"]
    end

    subgraph pages["Page Components"]
      %% KIND: boundary
      auth_pages["Auth Pages<br/><i>Component: React</i><br/>Login, Signup, Recover<br/>Password, Reset Password"]
      %% KIND: boundary
      dashboard_pages["Dashboard Pages<br/><i>Component: React</i><br/>Items list, Admin panel,<br/>User settings"]
    end

    subgraph features["Feature Components"]
      %% KIND: service_layer
      admin_components["Admin Components<br/><i>Component: React</i><br/>AddUser, EditUser, DeleteUser,<br/>UserActionsMenu"]
      %% KIND: service_layer
      item_components["Item Components<br/><i>Component: React</i><br/>AddItem, EditItem, DeleteItem,<br/>ItemActionsMenu"]
      %% KIND: service_layer
      settings_components["UserSettings Components<br/><i>Component: React</i><br/>UserInformation, ChangePassword,<br/>DeleteAccount"]
    end

    subgraph ui_layer["UI Layer"]
      %% KIND: boundary
      sidebar["Sidebar<br/><i>Component: React</i><br/>AppSidebar with navigation"]
      %% KIND: boundary
      shadcn_ui["shadcn/ui Components<br/><i>Component Library</i><br/>Dialog, Table, Form, Button,<br/>Card, Toast, etc."]
    end

    subgraph hooks_layer["Hooks"]
      %% KIND: service_layer
      use_auth["useAuth<br/><i>Component: React Hook</i><br/>Login, logout, signup<br/>mutations and user query"]
      %% KIND: service_layer
      custom_hooks["Custom Hooks<br/><i>Component: React Hooks</i><br/>useCustomToast,<br/>useCopyToClipboard, useMobile"]
    end

    subgraph api_client["API Client"]
      %% KIND: integration
      openapi_client["OpenAPI Client<br/><i>Component: Auto-generated</i><br/>ItemsService, UsersService,<br/>LoginService, UtilsService"]
      %% KIND: integration
      query_client["TanStack Query Client<br/><i>Component: React Query</i><br/>Cache, mutations, error handling"]
    end

    subgraph theme["Theme"]
      %% KIND: boundary
      theme_provider["Theme Provider<br/><i>Component: React Context</i><br/>Dark/light mode support"]
    end
  end

  app_user -->|"Interacts with UI<br/>[Browser]"| tanstack_router
  admin_user -->|"Manages system<br/>[Browser]"| tanstack_router

  tanstack_router --> route_tree
  route_tree --> auth_pages
  route_tree --> dashboard_pages

  auth_pages --> use_auth
  dashboard_pages --> admin_components
  dashboard_pages --> item_components
  dashboard_pages --> settings_components
  dashboard_pages --> sidebar

  admin_components --> shadcn_ui
  item_components --> shadcn_ui
  settings_components --> shadcn_ui

  admin_components --> openapi_client
  item_components --> openapi_client
  settings_components --> openapi_client

  use_auth --> openapi_client
  use_auth --> query_client

  openapi_client -->|"HTTP/JSON API calls"| fastapi_backend
  query_client --> openapi_client
```

---

## L3: Traefik — Component Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:container:traefik

  app_user["App User<br/><i>Person</i>"]
  admin_user["Admin User<br/><i>Person</i>"]
  spa["React SPA<br/><i>Container</i>"]
  fastapi_backend["FastAPI Backend<br/><i>Container</i>"]
  adminer["Adminer<br/><i>Container</i>"]

  subgraph traefik["Traefik"]

    subgraph entrypoints["Entrypoints"]
      %% KIND: router
      http_ep["HTTP Entrypoint<br/><i>Component: Listener</i><br/>Port 80"]
      %% KIND: router
      https_ep["HTTPS Entrypoint<br/><i>Component: Listener</i><br/>Port 443"]
    end

    subgraph middlewares["Middlewares"]
      %% KIND: boundary
      https_redirect["HTTPS Redirect<br/><i>Component: Middleware</i><br/>Redirects HTTP to HTTPS"]
      %% KIND: boundary
      basic_auth["Basic Auth<br/><i>Component: Middleware</i><br/>Protects Traefik dashboard"]
    end

    subgraph routers["Routers"]
      %% KIND: router
      frontend_router["Frontend Router<br/><i>Component: Router</i><br/>Host: dashboard.DOMAIN"]
      %% KIND: router
      backend_router["Backend Router<br/><i>Component: Router</i><br/>Host: api.DOMAIN"]
      %% KIND: router
      adminer_router["Adminer Router<br/><i>Component: Router</i><br/>Host: adminer.DOMAIN"]
    end

    subgraph tls["TLS"]
      %% KIND: integration
      le_resolver["Let's Encrypt Resolver<br/><i>Component: ACME</i><br/>Automatic TLS certificates"]
    end

    subgraph provider["Provider"]
      %% KIND: integration
      docker_provider["Docker Provider<br/><i>Component: Service Discovery</i><br/>Reads container labels"]
    end
  end

  app_user -->|"HTTPS"| https_ep
  admin_user -->|"HTTPS"| https_ep
  app_user -->|"HTTP"| http_ep

  http_ep --> https_redirect
  https_redirect --> https_ep

  https_ep --> frontend_router
  https_ep --> backend_router
  https_ep --> adminer_router

  frontend_router --> le_resolver
  backend_router --> le_resolver
  adminer_router --> le_resolver

  frontend_router -->|"Proxies to :80"| spa
  backend_router -->|"Proxies to :8000"| fastapi_backend
  adminer_router -->|"Proxies to :8080"| adminer

  docker_provider -->|"Discovers services"| frontend_router
  docker_provider -->|"Discovers services"| backend_router
  docker_provider -->|"Discovers services"| adminer_router
```

---

## Containment Map

```
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Template
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [app_user, admin_user]
fullstack_boundary CONTAINS [traefik, spa, fastapi_backend, pg, prestart, adminer]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment ──────────────────────────────────

%% FastAPI Backend (L3)
fastapi_backend    CONTAINS [middleware, routes, deps, services, models_layer, config_layer]
  middleware       CONTAINS [cors_mw, oauth2_mw]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes]
  deps             CONTAINS [auth_deps, session_dep]
  services         CONTAINS [crud_service, security_service, email_service]
  models_layer     CONTAINS [sqlmodels]
  config_layer     CONTAINS [app_config, db_engine]

%% React SPA (L3)
spa                CONTAINS [routing, pages, features, ui_layer, hooks_layer, api_client, theme]
  routing          CONTAINS [tanstack_router, route_tree]
  pages            CONTAINS [auth_pages, dashboard_pages]
  features         CONTAINS [admin_components, item_components, settings_components]
  ui_layer         CONTAINS [sidebar, shadcn_ui]
  hooks_layer      CONTAINS [use_auth, custom_hooks]
  api_client       CONTAINS [openapi_client, query_client]
  theme            CONTAINS [theme_provider]

%% Traefik (L3)
traefik            CONTAINS [entrypoints, middlewares, routers, tls, provider]
  entrypoints      CONTAINS [http_ep, https_ep]
  middlewares       CONTAINS [https_redirect, basic_auth]
  routers          CONTAINS [frontend_router, backend_router, adminer_router]
  tls              CONTAINS [le_resolver]
  provider         CONTAINS [docker_provider]
```
