# C4 Architecture Model – Full Stack FastAPI Platform

## L1: System Context

```mermaid
---
title: "L1: System Context – Full Stack FastAPI Platform"
---
%% SCOPE: urn:c4:system:fastapi_platform
flowchart TD

  subgraph users["Users"]
    end_user["End User
    <i>[Person]</i>
    <i>Uses the web application to</i>
    <i>manage items and account</i>"]
    admin_user["Admin User
    <i>[Person]</i>
    <i>Manages users, configuration,</i>
    <i>and system administration</i>"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    fastapi_platform["Full Stack FastAPI Platform
    <i>[Software System]</i>
    <i>Web application providing user</i>
    <i>management and item CRUD operations</i>"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server
    <i>[External System]</i>
    <i>Email delivery service</i>"]
    sentry["Sentry
    <i>[External System]</i>
    <i>Error monitoring and tracing</i>"]
    letsencrypt["Let's Encrypt
    <i>[External System]</i>
    <i>TLS certificate authority</i>"]
  end

  end_user -->|"Uses web application\n[HTTPS]"| fastapi_platform
  admin_user -->|"Manages users and system\n[HTTPS]"| fastapi_platform
  fastapi_platform -->|"Sends emails\n[SMTP/TLS]"| smtp
  fastapi_platform -->|"Reports errors\n[HTTPS]"| sentry
  fastapi_platform -->|"Obtains TLS certificates\n[ACME]"| letsencrypt

  classDef person fill:#08427B,stroke:#052E56,color:#fff
  classDef system fill:#1168BD,stroke:#0B4884,color:#fff
  classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
  classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5

  class end_user,admin_user person
  class fastapi_platform system
  class smtp,sentry,letsencrypt ext
  class platform_boundary,users,external boundary
```

---

## L2: Container Diagram

```mermaid
---
title: "L2: Container – Full Stack FastAPI Platform"
---
%% SCOPE: urn:c4:system:fastapi_platform
flowchart TD

  subgraph users["Users"]
    end_user["End User
    <i>[Person]</i>"]
    admin_user["Admin User
    <i>[Person]</i>"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik
    <i>[Container: Traefik 3.x]</i>
    <i>Reverse proxy, TLS termination,</i>
    <i>load balancing, HTTP routing</i>"]
    spa["React SPA
    <i>[Container: React 19, TypeScript, Vite]</i>
    <i>Single-page application served by</i>
    <i>nginx; user dashboard with shadcn/ui</i>"]
    fastapi_api["FastAPI Backend
    <i>[Container: Python, FastAPI]</i>
    <i>REST API providing authentication,</i>
    <i>user management, and item CRUD</i>"]
    pg["PostgreSQL
    <i>[Container: PostgreSQL 18]</i>
    <i>Stores user accounts, items,</i>
    <i>and application data</i>"]
    adminer["Adminer
    <i>[Container: Adminer]</i>
    <i>Database administration web UI</i>"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server
    <i>[External System]</i>"]
    sentry["Sentry
    <i>[External System]</i>"]
    letsencrypt["Let's Encrypt
    <i>[External System]</i>"]
  end

  end_user -->|"Browses dashboard\n[HTTPS]"| traefik
  admin_user -->|"Administers system\n[HTTPS]"| traefik
  traefik -->|"Serves static assets\n[HTTP :80]"| spa
  traefik -->|"Proxies API requests\n[HTTP :8000]"| fastapi_api
  traefik -->|"Proxies DB admin\n[HTTP :8080]"| adminer
  spa -.->|"API calls from browser\n[JSON/HTTPS via Traefik]"| fastapi_api
  fastapi_api -->|"Reads/writes data\n[SQL, psycopg]"| pg
  adminer -->|"Administers database\n[SQL]"| pg
  fastapi_api -->|"Sends emails\n[SMTP/TLS]"| smtp
  fastapi_api -.->|"Reports errors\n[HTTPS]"| sentry
  traefik -->|"Obtains TLS certificates\n[ACME]"| letsencrypt

  classDef person fill:#08427B,stroke:#052E56,color:#fff
  classDef container fill:#438DD5,stroke:#2E6295,color:#fff
  classDef database fill:#438DD5,stroke:#2E6295,color:#fff
  classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
  classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5

  class end_user,admin_user person
  class traefik,spa,fastapi_api,adminer container
  class pg database
  class smtp,sentry,letsencrypt ext
  class platform_boundary,users,external boundary
```

---

## L3: FastAPI Backend

```mermaid
---
title: "L3: Component – FastAPI Backend"
---
%% SCOPE: urn:c4:container:fastapi_api
flowchart TD

  subgraph fastapi_api["FastAPI Backend"]

    subgraph mw["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware
      <i>[Component: Starlette CORSMiddleware]</i>
      <i>Handles cross-origin requests</i>
      <i>for configured origins</i>"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes
      <i>[Component: FastAPI Router]</i>
      <i>/login/*, /password-recovery/*,</i>
      <i>/reset-password/</i>"]
      %% KIND: router
      user_routes["User Routes
      <i>[Component: FastAPI Router]</i>
      <i>/users/* — CRUD, signup,</i>
      <i>profile, password change</i>"]
      %% KIND: router
      item_routes["Item Routes
      <i>[Component: FastAPI Router]</i>
      <i>/items/* — CRUD operations</i>
      <i>with ownership enforcement</i>"]
      %% KIND: router
      util_routes["Utils Routes
      <i>[Component: FastAPI Router]</i>
      <i>/utils/* — health check,</i>
      <i>test email</i>"]
    end

    subgraph services["Service Layer"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies
      <i>[Component: FastAPI Depends]</i>
      <i>OAuth2 bearer token validation,</i>
      <i>current user extraction, session DI</i>"]
      %% KIND: data_access
      crud_layer["CRUD Operations
      <i>[Component: Python module]</i>
      <i>User and item create/read/</i>
      <i>update/delete with SQLModel</i>"]
      %% KIND: integration
      email_utils["Email Utilities
      <i>[Component: Python module]</i>
      <i>Sends transactional emails via</i>
      <i>SMTP using Jinja2 templates</i>"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      security_mod["Security
      <i>[Component: Python module]</i>
      <i>JWT token creation/validation,</i>
      <i>Argon2/bcrypt password hashing</i>"]
      %% KIND: service_layer
      config_mod["Configuration
      <i>[Component: Pydantic Settings]</i>
      <i>Application settings loaded from</i>
      <i>environment variables and .env</i>"]
      %% KIND: storage
      db_engine["Database Engine
      <i>[Component: SQLAlchemy Engine]</i>
      <i>Connection pool and session</i>
      <i>factory for PostgreSQL</i>"]
    end

    %% KIND: data_access
    models_layer["SQLModel Models
    <i>[Component: SQLModel/Pydantic]</i>
    <i>User, Item table models and</i>
    <i>request/response schemas</i>"]

  end

  pg["PostgreSQL
  <i>[Container]</i>"]
  smtp["SMTP Server
  <i>[External System]</i>"]
  sentry["Sentry
  <i>[External System]</i>"]

  cors_mw --> routes
  login_routes --> auth_deps
  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_utils
  user_routes --> auth_deps
  user_routes --> crud_layer
  user_routes --> security_mod
  user_routes --> email_utils
  item_routes --> auth_deps
  item_routes --> models_layer
  util_routes --> email_utils
  auth_deps --> security_mod
  auth_deps --> db_engine
  auth_deps --> models_layer
  crud_layer --> models_layer
  crud_layer --> security_mod
  email_utils --> security_mod
  email_utils --> config_mod
  security_mod --> config_mod
  db_engine --> config_mod
  db_engine -->|"SQL\n[psycopg]"| pg
  email_utils -->|"Sends email\n[SMTP/TLS]"| smtp
  fastapi_api -.->|"Reports errors\n[HTTPS]"| sentry

  classDef component fill:#85BBF0,stroke:#5D82A8,color:#000
  classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
  classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5
  classDef grouping fill:none,stroke:#7B7B7B,stroke-width:1px,stroke-dasharray:3 3

  class cors_mw,login_routes,user_routes,item_routes,util_routes component
  class auth_deps,crud_layer,email_utils component
  class security_mod,config_mod,db_engine component
  class models_layer component
  class pg,smtp,sentry ext
  class fastapi_api boundary
  class mw,routes,services,core grouping
```

---

## L3: React SPA

```mermaid
---
title: "L3: Component – React SPA"
---
%% SCOPE: urn:c4:container:spa
flowchart TD

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router
      <i>[Component: @tanstack/react-router]</i>
      <i>File-based routing with</i>
      <i>type-safe navigation</i>"]
      %% KIND: boundary
      route_guard["Route Guard
      <i>[Component: beforeLoad hook]</i>
      <i>Redirects unauthenticated</i>
      <i>users to login page</i>"]
    end

    subgraph pages["Pages"]
      %% KIND: boundary
      auth_pages["Auth Pages
      <i>[Component: React pages]</i>
      <i>Login, Signup, Recover Password,</i>
      <i>Reset Password</i>"]
      %% KIND: boundary
      dashboard_pages["Dashboard Pages
      <i>[Component: React pages]</i>
      <i>Home, Admin, Items, Settings</i>
      <i>— protected by route guard</i>"]
    end

    subgraph features["Feature Components"]
      %% KIND: service_layer
      admin_features["Admin Features
      <i>[Component: React components]</i>
      <i>User table, add/edit/delete</i>
      <i>user dialogs and actions</i>"]
      %% KIND: service_layer
      item_features["Item Features
      <i>[Component: React components]</i>
      <i>Item table, add/edit/delete</i>
      <i>item dialogs and actions</i>"]
      %% KIND: service_layer
      settings_features["Settings Features
      <i>[Component: React components]</i>
      <i>User info, change password,</i>
      <i>delete account</i>"]
    end

    subgraph shared["Shared UI"]
      %% KIND: boundary
      ui_lib["shadcn/ui Components
      <i>[Component: Radix UI + Tailwind]</i>
      <i>Buttons, dialogs, tables, forms,</i>
      <i>sidebar, tooltips, etc.</i>"]
      %% KIND: boundary
      sidebar_comp["Sidebar
      <i>[Component: React components]</i>
      <i>Application sidebar with</i>
      <i>navigation and user menu</i>"]
      %% KIND: boundary
      common_comp["Common Components
      <i>[Component: React components]</i>
      <i>Layout shell, footer, logo,</i>
      <i>error boundary, not-found</i>"]
    end

    %% KIND: integration
    api_client["OpenAPI Client SDK
    <i>[Component: Auto-generated]</i>
    <i>Type-safe HTTP client generated</i>
    <i>from backend OpenAPI schema</i>"]
    %% KIND: service_layer
    auth_hooks["Auth Hooks
    <i>[Component: React hooks]</i>
    <i>useAuth — login, logout, signup</i>
    <i>mutations and user query</i>"]
    %% KIND: service_layer
    query_client["Query Client
    <i>[Component: @tanstack/react-query]</i>
    <i>Server state management, caching,</i>
    <i>and global error handling</i>"]
    %% KIND: service_layer
    theme_provider["Theme Provider
    <i>[Component: next-themes]</i>
    <i>Dark/light mode toggle</i>
    <i>with system preference support</i>"]

  end

  fastapi_api["FastAPI Backend
  <i>[Container]</i>"]

  tanstack_router --> route_guard
  tanstack_router --> auth_pages
  tanstack_router --> dashboard_pages
  route_guard --> auth_hooks
  auth_pages --> auth_hooks
  auth_pages --> api_client
  auth_pages --> ui_lib
  dashboard_pages --> admin_features
  dashboard_pages --> item_features
  dashboard_pages --> settings_features
  dashboard_pages --> sidebar_comp
  dashboard_pages --> common_comp
  admin_features --> api_client
  admin_features --> query_client
  admin_features --> ui_lib
  item_features --> api_client
  item_features --> query_client
  item_features --> ui_lib
  settings_features --> api_client
  settings_features --> query_client
  settings_features --> ui_lib
  auth_hooks --> api_client
  auth_hooks --> query_client
  api_client -->|"REST API calls\n[JSON/HTTPS]"| fastapi_api

  classDef component fill:#85BBF0,stroke:#5D82A8,color:#000
  classDef ext fill:#438DD5,stroke:#2E6295,color:#fff
  classDef boundary fill:none,stroke:#444,stroke-width:2px,stroke-dasharray:5 5
  classDef grouping fill:none,stroke:#7B7B7B,stroke-width:1px,stroke-dasharray:3 3

  class tanstack_router,route_guard component
  class auth_pages,dashboard_pages component
  class admin_features,item_features,settings_features component
  class ui_lib,sidebar_comp,common_comp component
  class api_client,auth_hooks,query_client,theme_provider component
  class fastapi_api ext
  class spa boundary
  class routing,pages,features,shared grouping
```

---

## Containment Map

```text
%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [end_user, admin_user]
platform_boundary  CONTAINS [traefik, spa, fastapi_api, pg, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2→L3 internal containment ─────────────────────────────────────
fastapi_api  CONTAINS [mw, routes, services, core, models_layer]
  mw         CONTAINS [cors_mw]
  routes     CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services   CONTAINS [auth_deps, crud_layer, email_utils]
  core       CONTAINS [security_mod, config_mod, db_engine]

spa          CONTAINS [routing, pages, features, shared, api_client, auth_hooks, query_client, theme_provider]
  routing    CONTAINS [tanstack_router, route_guard]
  pages      CONTAINS [auth_pages, dashboard_pages]
  features   CONTAINS [admin_features, item_features, settings_features]
  shared     CONTAINS [ui_lib, sidebar_comp, common_comp]
```
