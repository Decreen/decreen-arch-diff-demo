# C4 Architecture Model — Full Stack FastAPI Template

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  %% ── Actors ────────────────────────────────────────────────────────
  subgraph users["Users"]
    end_user["End User<br/><i>Regular application user</i>"]
    admin_user["Admin User<br/><i>Superuser with elevated privileges</i>"]
  end

  %% ── System Boundary ───────────────────────────────────────────────
  subgraph system_boundary["Full Stack FastAPI Platform"]
    fastapi_system["Full Stack FastAPI Platform<br/><i>Web application for managing<br/>users and items with JWT auth</i>"]
  end

  %% ── External Systems ──────────────────────────────────────────────
  subgraph external["External Systems"]
    smtp["SMTP Server<br/><i>Email delivery service</i>"]
    sentry["Sentry<br/><i>Error monitoring &amp; tracing</i>"]
  end

  %% ── Relationships ─────────────────────────────────────────────────
  end_user -->|"Manages items,<br/>views dashboard<br/>[HTTPS]"| fastapi_system
  admin_user -->|"Manages users &amp; items,<br/>admin operations<br/>[HTTPS]"| fastapi_system
  fastapi_system -->|"Sends transactional<br/>emails<br/>[SMTP/TLS]"| smtp
  fastapi_system -->|"Reports errors<br/>&amp; traces<br/>[HTTPS]"| sentry
```

---

## L2: Container Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  %% ── Actors ────────────────────────────────────────────────────────
  subgraph users["Users"]
    end_user["End User<br/><i>Regular application user</i>"]
    admin_user["Admin User<br/><i>Superuser with elevated privileges</i>"]
  end

  %% ── System Boundary ───────────────────────────────────────────────
  subgraph system_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik<br/><i>Reverse proxy, TLS termination,<br/>load balancer</i><br/>[Docker container]"]
    react_spa["React SPA<br/><i>Single-page application<br/>React + TypeScript + Vite</i><br/>[Nginx / Docker container]"]
    fastapi_backend["FastAPI Backend<br/><i>REST API server<br/>Python FastAPI + SQLModel</i><br/>[Docker container]"]
    pg["PostgreSQL<br/><i>Relational database<br/>Users, Items storage</i><br/>[Docker container]"]
  end

  %% ── External Systems ──────────────────────────────────────────────
  subgraph external["External Systems"]
    smtp["SMTP Server<br/><i>Email delivery service</i>"]
    sentry["Sentry<br/><i>Error monitoring &amp; tracing</i>"]
  end

  %% ── Relationships ─────────────────────────────────────────────────
  end_user -->|"Browses application<br/>[HTTPS]"| traefik
  admin_user -->|"Browses application<br/>[HTTPS]"| traefik

  traefik -->|"Routes to dashboard.*<br/>[HTTP]"| react_spa
  traefik -->|"Routes to api.*<br/>[HTTP]"| fastapi_backend

  react_spa -->|"API calls<br/>[JSON/HTTPS]"| fastapi_backend

  fastapi_backend -->|"Reads/writes data<br/>[SQL via psycopg]"| pg
  fastapi_backend -->|"Sends emails<br/>[SMTP/TLS]"| smtp
  fastapi_backend -->|"Reports errors<br/>[HTTPS]"| sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi-backend

  %% ── External actors that touch this container ─────────────────────
  react_spa["React SPA<br/><i>Frontend client</i>"]
  pg["PostgreSQL<br/><i>Database</i>"]
  smtp["SMTP Server"]
  sentry["Sentry"]

  %% ── Container boundary ────────────────────────────────────────────
  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware"]
      cors_mw["CORS Middleware<br/><i>Handles cross-origin requests</i>"]
    end

    %% KIND: boundary
    subgraph auth_deps["Auth Dependencies"]
      %% KIND: service_layer
      oauth2_scheme["OAuth2 Bearer Scheme<br/><i>Token extraction</i>"]
      %% KIND: service_layer
      get_current_user["get_current_user<br/><i>JWT validation &amp; user lookup</i>"]
      %% KIND: service_layer
      get_superuser["get_current_active_superuser<br/><i>Admin privilege check</i>"]
    end

    %% KIND: boundary
    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes<br/><i>/login/access-token<br/>/password-recovery<br/>/reset-password</i>"]
      %% KIND: router
      users_routes["Users Routes<br/><i>/users CRUD<br/>/users/signup<br/>/users/me</i>"]
      %% KIND: router
      items_routes["Items Routes<br/><i>/items CRUD</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>/utils/health-check<br/>/utils/test-email</i>"]
    end

    %% KIND: service_layer
    subgraph core["Core Services"]
      %% KIND: service_layer
      security_mod["Security Module<br/><i>JWT creation, password<br/>hashing (Argon2/Bcrypt)</i>"]
      %% KIND: service_layer
      config_mod["Config Module<br/><i>Pydantic Settings<br/>Environment management</i>"]
    end

    %% KIND: service_layer
    email_utils["Email Utilities<br/><i>Template rendering,<br/>email sending, token generation</i>"]

    %% KIND: data_access
    crud_layer["CRUD Layer<br/><i>Database operations:<br/>create/read/update/delete<br/>users &amp; items</i>"]

    %% KIND: data_access
    models_layer["SQLModel Models<br/><i>User, Item, Token<br/>ORM &amp; Pydantic schemas</i>"]

    %% KIND: data_access
    db_engine["DB Engine<br/><i>SQLAlchemy engine<br/>&amp; session management</i>"]
  end

  %% ── Relationships ─────────────────────────────────────────────────
  react_spa -->|"HTTP requests<br/>[JSON]"| cors_mw
  cors_mw --> routes

  login_routes --> auth_deps
  users_routes --> auth_deps
  items_routes --> auth_deps
  utils_routes --> auth_deps

  login_routes --> security_mod
  login_routes --> crud_layer
  login_routes --> email_utils

  users_routes --> crud_layer
  users_routes --> security_mod
  users_routes --> email_utils

  items_routes --> crud_layer

  utils_routes --> email_utils

  auth_deps --> security_mod
  auth_deps --> db_engine
  auth_deps --> models_layer

  crud_layer --> models_layer
  crud_layer --> security_mod
  crud_layer --> db_engine

  email_utils --> security_mod
  email_utils --> config_mod
  email_utils --> smtp

  db_engine --> pg
  db_engine --> config_mod

  security_mod --> config_mod

  fastapi_backend -.->|"Sentry SDK init"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:react-spa

  %% ── External actors ───────────────────────────────────────────────
  end_user["End User"]
  admin_user["Admin User"]
  fastapi_backend["FastAPI Backend<br/><i>REST API</i>"]

  %% ── Container boundary ────────────────────────────────────────────
  subgraph react_spa["React SPA"]

    %% KIND: router
    subgraph routing["TanStack Router"]
      root_route["Root Route<br/><i>Error &amp; NotFound handling</i>"]
      auth_guard["Layout Route<br/><i>Auth guard (isLoggedIn)</i>"]
    end

    %% KIND: boundary
    subgraph auth_pages["Auth Pages"]
      login_page["Login Page<br/><i>/login</i>"]
      signup_page["Signup Page<br/><i>/signup</i>"]
      recover_pw_page["Recover Password Page<br/><i>/recover-password</i>"]
      reset_pw_page["Reset Password Page<br/><i>/reset-password</i>"]
    end

    %% KIND: boundary
    subgraph app_pages["Application Pages"]
      dashboard_page["Dashboard Page<br/><i>/ (home)</i>"]
      items_page["Items Page<br/><i>/items — CRUD items</i>"]
      admin_page["Admin Page<br/><i>/admin — User management</i>"]
      settings_page["Settings Page<br/><i>/settings — Profile &amp; password</i>"]
    end

    %% KIND: integration
    subgraph api_client["API Client Layer"]
      openapi_client["OpenAPI Client<br/><i>Auto-generated SDK<br/>(hey-api/openapi-ts)</i>"]
      query_client["TanStack React Query<br/><i>Data fetching, caching,<br/>mutation management</i>"]
    end

    %% KIND: service_layer
    subgraph hooks["Custom Hooks"]
      use_auth["useAuth<br/><i>Login, logout, signup,<br/>current user state</i>"]
      use_toast["useCustomToast<br/><i>Error/success notifications</i>"]
    end

    %% KIND: boundary
    subgraph ui_layer["UI Framework"]
      theme_provider["Theme Provider<br/><i>Dark/light mode</i>"]
      sidebar_nav["Sidebar Navigation<br/><i>App navigation menu</i>"]
      shadcn_ui["shadcn/ui Components<br/><i>Buttons, Forms, Dialogs,<br/>Tables, etc.</i>"]
    end
  end

  %% ── Relationships ─────────────────────────────────────────────────
  end_user -->|"Interacts with UI<br/>[Browser]"| routing
  admin_user -->|"Interacts with UI<br/>[Browser]"| routing

  root_route --> auth_pages
  auth_guard --> app_pages
  auth_guard --> sidebar_nav

  login_page --> use_auth
  signup_page --> use_auth
  recover_pw_page --> openapi_client
  reset_pw_page --> openapi_client

  dashboard_page --> use_auth
  items_page --> query_client
  admin_page --> query_client
  settings_page --> query_client
  settings_page --> use_auth

  use_auth --> openapi_client
  use_auth --> query_client
  query_client --> openapi_client

  app_pages --> shadcn_ui
  auth_pages --> shadcn_ui
  app_pages --> use_toast

  openapi_client -->|"REST API calls<br/>[JSON/HTTPS]"| fastapi_backend
```

---

## L3: Traefik

```mermaid
graph TB
  %% SCOPE: urn:c4:container:traefik

  %% ── External actors ───────────────────────────────────────────────
  end_user["End User"]
  admin_user["Admin User"]
  react_spa["React SPA"]
  fastapi_backend["FastAPI Backend"]

  %% ── Container boundary ────────────────────────────────────────────
  subgraph traefik["Traefik"]

    %% KIND: router
    entrypoints["Entrypoints<br/><i>HTTP (:80) &amp; HTTPS (:443)</i>"]

    %% KIND: service_layer
    https_redirect["HTTPS Redirect Middleware<br/><i>HTTP → HTTPS redirection</i>"]

    %% KIND: service_layer
    tls_resolver["TLS Certificate Resolver<br/><i>Let's Encrypt ACME<br/>automatic certificates</i>"]

    %% KIND: router
    frontend_router["Frontend Router<br/><i>Host: dashboard.*</i>"]

    %% KIND: router
    backend_router["Backend Router<br/><i>Host: api.*</i>"]
  end

  %% ── Relationships ─────────────────────────────────────────────────
  end_user -->|"HTTPS"| entrypoints
  admin_user -->|"HTTPS"| entrypoints

  entrypoints --> https_redirect
  https_redirect --> tls_resolver
  tls_resolver --> frontend_router
  tls_resolver --> backend_router

  frontend_router -->|"Proxy to :80"| react_spa
  backend_router -->|"Proxy to :8000"| fastapi_backend
```

---

## L3: PostgreSQL

```mermaid
graph TB
  %% SCOPE: urn:c4:container:pg

  %% ── External actors ───────────────────────────────────────────────
  fastapi_backend["FastAPI Backend"]

  %% ── Container boundary ────────────────────────────────────────────
  subgraph pg["PostgreSQL"]

    %% KIND: storage
    user_table["user Table<br/><i>id, email, hashed_password,<br/>full_name, is_active,<br/>is_superuser, created_at</i>"]

    %% KIND: storage
    item_table["item Table<br/><i>id, title, description,<br/>owner_id (FK → user),<br/>created_at</i>"]

    %% KIND: storage
    alembic_version["alembic_version Table<br/><i>Migration version tracking</i>"]
  end

  %% ── Relationships ─────────────────────────────────────────────────
  fastapi_backend -->|"SQL queries<br/>[psycopg]"| user_table
  fastapi_backend -->|"SQL queries<br/>[psycopg]"| item_table
  item_table -->|"FK: owner_id<br/>CASCADE DELETE"| user_table
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ══════════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────────
users            CONTAINS [end_user, admin_user]
system_boundary  CONTAINS [traefik, react_spa, fastapi_backend, pg]
external         CONTAINS [smtp, sentry]

%% ── L2→L3 internal containment ─────────────────────────────────────

%% FastAPI Backend (L3)
fastapi_backend  CONTAINS [middleware, auth_deps, routes, core, email_utils, crud_layer, models_layer, db_engine]
  middleware     CONTAINS [cors_mw]
  auth_deps      CONTAINS [oauth2_scheme, get_current_user, get_superuser]
  routes         CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  core           CONTAINS [security_mod, config_mod]

%% React SPA (L3)
react_spa        CONTAINS [routing, auth_pages, app_pages, api_client, hooks, ui_layer]
  routing        CONTAINS [root_route, auth_guard]
  auth_pages     CONTAINS [login_page, signup_page, recover_pw_page, reset_pw_page]
  app_pages      CONTAINS [dashboard_page, items_page, admin_page, settings_page]
  api_client     CONTAINS [openapi_client, query_client]
  hooks          CONTAINS [use_auth, use_toast]
  ui_layer       CONTAINS [theme_provider, sidebar_nav, shadcn_ui]

%% Traefik (L3)
traefik          CONTAINS [entrypoints, https_redirect, tls_resolver, frontend_router, backend_router]

%% PostgreSQL (L3)
pg               CONTAINS [user_table, item_table, alembic_version]
```
