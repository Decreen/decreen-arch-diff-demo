# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
flowchart TB
  %% L1 System Context
  %% SCOPE: urn:c4:system:fullstack_fastapi_platform

  subgraph users [" Users "]
    user(["End User<br/><i>Interacts with the web<br/>application via browser</i>"])
    admin_user(["Administrator<br/><i>Superuser who manages<br/>users and system</i>"])
  end

  subgraph system_boundary [" Full Stack FastAPI Platform "]
    platform["Full Stack FastAPI Platform<br/><i>Web application providing user<br/>management, item tracking,<br/>and admin dashboard</i>"]
  end

  subgraph external [" External Systems "]
    smtp["SMTP Server<br/><i>Email delivery service</i>"]
    sentry["Sentry<br/><i>Error monitoring and<br/>performance tracking</i>"]
  end

  user -- "Uses web application<br/>[HTTPS]" --> platform
  admin_user -- "Manages users & system<br/>[HTTPS]" --> platform
  platform -- "Sends emails<br/>[SMTP/TLS]" --> smtp
  platform -- "Reports errors<br/>[HTTPS]" --> sentry
```

---

## L2: Container

```mermaid
flowchart TB
  %% L2 Container
  %% SCOPE: urn:c4:system:fullstack_fastapi_platform

  subgraph users [" Users "]
    user(["End User"])
    admin_user(["Administrator"])
  end

  subgraph system_boundary [" Full Stack FastAPI Platform "]
    traefik["Traefik<br/><i>Reverse proxy &amp; TLS termination,<br/>routes requests to frontend<br/>and backend</i><br/>[Docker: Traefik]"]
    spa["React SPA<br/><i>Single-page application<br/>with TanStack Router,<br/>shadcn/ui components</i><br/>[Nginx / React 19 + Vite]"]
    fastapi_backend["FastAPI Backend<br/><i>REST API handling auth,<br/>user management, and<br/>item operations</i><br/>[Python / FastAPI + SQLModel]"]
    pg[("PostgreSQL<br/><i>Stores users, items,<br/>and application data</i><br/>[PostgreSQL 18]")]
    adminer["Adminer<br/><i>Database administration UI</i><br/>[Docker: Adminer]"]
  end

  subgraph external [" External Systems "]
    smtp["SMTP Server<br/><i>Email delivery</i>"]
    sentry["Sentry<br/><i>Error monitoring</i>"]
  end

  user -- "HTTPS" --> traefik
  admin_user -- "HTTPS" --> traefik
  traefik -- "Routes /" --> spa
  traefik -- "Routes /api" --> fastapi_backend
  spa -- "REST API calls<br/>[JSON/HTTP]" --> fastapi_backend
  fastapi_backend -- "SQL queries<br/>[psycopg]" --> pg
  fastapi_backend -- "Sends emails<br/>[SMTP]" --> smtp
  fastapi_backend -- "Error reports<br/>[HTTPS/SDK]" --> sentry
  admin_user -. "DB admin<br/>[HTTP]" .-> adminer
  adminer -- "SQL" --> pg
```

---

## L3: FastAPI Backend

```mermaid
flowchart TB
  %% L3 Component — FastAPI Backend
  %% SCOPE: urn:c4:container:fastapi_backend

  spa["React SPA"]
  pg[("PostgreSQL")]
  smtp["SMTP Server"]

  subgraph fastapi_backend [" FastAPI Backend "]

    subgraph mw [" Middleware & Dependencies "]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Handles cross-origin<br/>request policies</i>"]
      %% KIND: boundary
      auth_deps["Auth Dependencies<br/><i>JWT validation, user loading,<br/>superuser guards via DI</i>"]
    end

    subgraph routes [" API Routes "]
      %% KIND: router
      login_routes["Login Routes<br/><i>POST /login/access-token<br/>POST /password-recovery/&lbrace;email&rbrace;<br/>POST /reset-password</i>"]
      %% KIND: router
      users_routes["Users Routes<br/><i>GET/POST/PATCH/DELETE /users<br/>GET /users/me &middot; POST /signup</i>"]
      %% KIND: router
      items_routes["Items Routes<br/><i>GET/POST/PUT/DELETE /items</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>POST /utils/test-email<br/>GET /utils/health-check</i>"]
    end

    subgraph services [" Services "]
      %% KIND: data_access
      crud_layer["CRUD Layer<br/><i>create_user, authenticate,<br/>create_item, get_user_by_email</i>"]
      %% KIND: service_layer
      security_mod["Security Module<br/><i>JWT token creation,<br/>Argon2/Bcrypt password hashing</i>"]
      %% KIND: integration
      email_utils["Email Utilities<br/><i>SMTP email sending,<br/>HTML template rendering</i>"]
      %% KIND: service_layer
      config_mod["Configuration<br/><i>Pydantic Settings &mdash; DB, SMTP,<br/>auth, and app config from .env</i>"]
    end

    subgraph data_access [" Data Access "]
      %% KIND: storage
      db_session["DB Session &amp; Engine<br/><i>SQLAlchemy create_engine,<br/>session context manager</i>"]
    end

  end

  spa -- "HTTP requests" --> cors_mw
  cors_mw --> auth_deps

  auth_deps --> login_routes
  auth_deps --> users_routes
  auth_deps --> items_routes
  auth_deps --> utils_routes

  login_routes --> crud_layer
  login_routes --> security_mod
  login_routes --> email_utils

  users_routes --> crud_layer
  users_routes --> security_mod

  items_routes --> crud_layer

  utils_routes --> email_utils

  crud_layer --> db_session
  auth_deps --> security_mod
  auth_deps --> db_session

  db_session -- "SQL queries<br/>[psycopg]" --> pg
  email_utils -- "Sends emails<br/>[SMTP]" --> smtp

  security_mod --> config_mod
  email_utils --> config_mod
  db_session --> config_mod
```

---

## L3: React SPA

```mermaid
flowchart TB
  %% L3 Component — React SPA
  %% SCOPE: urn:c4:container:spa

  fastapi_backend["FastAPI Backend"]

  subgraph spa [" React SPA "]

    subgraph routing [" Routing "]
      %% KIND: router
      tanstack_router["TanStack Router<br/><i>File-based routing with<br/>code splitting &amp; route guards</i>"]
    end

    subgraph features [" Feature Modules "]
      %% KIND: service_layer
      auth_pages["Auth Pages<br/><i>Login, Signup,<br/>Password Recovery &amp; Reset</i>"]
      %% KIND: service_layer
      dashboard_page["Dashboard<br/><i>Welcome page with<br/>current user info</i>"]
      %% KIND: service_layer
      admin_mod["Admin Module<br/><i>User management<br/>DataTable (superuser only)</i>"]
      %% KIND: service_layer
      items_mod["Items Module<br/><i>Item CRUD with<br/>DataTable, Add/Edit/Delete</i>"]
      %% KIND: service_layer
      settings_mod["Settings Module<br/><i>Profile editing, password<br/>change, account deletion</i>"]
    end

    subgraph services_layer [" Services "]
      %% KIND: service_layer
      auth_hook["Auth Hook<br/><i>useAuth — login, logout,<br/>signup, current user state</i>"]
      %% KIND: integration
      api_client["API Client<br/><i>OpenAPI-generated Axios client<br/>with token interceptor</i>"]
      %% KIND: service_layer
      theme_provider["Theme Provider<br/><i>Light / Dark / System<br/>theme management</i>"]
    end

    subgraph ui_layer [" UI Layer "]
      %% KIND: boundary
      sidebar_comp["Sidebar<br/><i>Navigation, user area,<br/>theme toggle</i>"]
      %% KIND: boundary
      ui_primitives["UI Primitives<br/><i>shadcn/ui + Radix UI<br/>component library</i>"]
    end

  end

  tanstack_router --> auth_pages
  tanstack_router --> dashboard_page
  tanstack_router --> admin_mod
  tanstack_router --> items_mod
  tanstack_router --> settings_mod

  auth_pages --> auth_hook
  auth_pages --> ui_primitives

  dashboard_page --> api_client
  dashboard_page --> ui_primitives

  admin_mod --> api_client
  admin_mod --> ui_primitives

  items_mod --> api_client
  items_mod --> ui_primitives

  settings_mod --> api_client
  settings_mod --> ui_primitives

  auth_hook --> api_client

  tanstack_router --> sidebar_comp
  sidebar_comp --> theme_provider
  sidebar_comp --> ui_primitives

  api_client -- "REST API<br/>[JSON/HTTP]" --> fastapi_backend
```

---

## Containment Map

```text
%% ── CONTAINMENT MAP ─────────────────────────────────────────────
%%
%% Every subgraph from every diagram appears as a CONTAINS entry.
%% Nesting is expressed by indentation: child groups are indented
%% under their parent.

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [user, admin_user]
system_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external         CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment (FastAPI Backend) ────────────────
fastapi_backend  CONTAINS [mw, routes, services, data_access]
  mw             CONTAINS [cors_mw, auth_deps]
  routes         CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services       CONTAINS [crud_layer, security_mod, email_utils, config_mod]
  data_access    CONTAINS [db_session]

%% ── L2→L3 internal containment (React SPA) ─────────────────────
spa              CONTAINS [routing, features, services_layer, ui_layer]
  routing        CONTAINS [tanstack_router]
  features       CONTAINS [auth_pages, dashboard_page, admin_mod, items_mod, settings_mod]
  services_layer CONTAINS [auth_hook, api_client, theme_provider]
  ui_layer       CONTAINS [sidebar_comp, ui_primitives]
```
