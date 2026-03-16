# C4 Architecture — Full Stack FastAPI Project

> C4 model for the **Full Stack FastAPI Project**, a web application for user
> and item management with JWT authentication, admin panel, and email
> notifications. Three levels of zoom: System Context, Container, Component.

---

## L1: System Context

Actors, the software system, and external dependencies.

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
graph TB

  subgraph users["Users"]
    end_user["End User\n[Person]\nAuthenticated user who manages\nitems and personal settings"]
    admin_user["Admin User\n[Person]\nSuperuser who manages\nall users and the system"]
    devops["DevOps Engineer\n[Person]\nDeveloper / Operator"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    fullstack_system["Full Stack FastAPI Project\n[Software System]\nWeb app for user and item management\nwith admin panel and JWT auth"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n[External System]\nEmail delivery service"]
    sentry["Sentry\n[External System]\nError monitoring and tracing"]
  end

  end_user -->|"Uses\n[HTTPS]"| fullstack_system
  admin_user -->|"Manages\n[HTTPS]"| fullstack_system
  devops -->|"Monitors\n[HTTPS]"| fullstack_system
  fullstack_system -->|"Sends emails\n[SMTP/TLS]"| smtp
  fullstack_system -->|"Reports errors\n[HTTPS]"| sentry
```

---

## L2: Container

All deployable units inside the system boundary.

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
graph TB

  subgraph users["Users"]
    end_user["End User\n[Person]"]
    admin_user["Admin User\n[Person]"]
    devops["DevOps Engineer\n[Person]"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    traefik["Traefik\n[Container: Traefik 3.6]\nReverse proxy, TLS termination,\ndomain-based routing"]
    spa["React SPA\n[Container: React 19 / Nginx]\nDashboard UI for users and admins"]
    fastapi_backend["FastAPI Backend\n[Container: Python / FastAPI]\nREST API, authentication,\nbusiness logic"]
    pg["PostgreSQL\n[Container: PostgreSQL 18]\nPrimary relational data store"]
    adminer["Adminer\n[Container: Adminer]\nDatabase administration UI"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Server\n[External System]"]
    sentry["Sentry\n[External System]"]
  end

  end_user -->|"Browses dashboard\n[HTTPS]"| traefik
  admin_user -->|"Manages users\n[HTTPS]"| traefik
  devops -->|"Administers DB\n[HTTPS]"| traefik

  traefik -->|"dashboard.*\n[HTTP]"| spa
  traefik -->|"api.*\n[HTTP]"| fastapi_backend
  traefik -->|"adminer.*\n[HTTP]"| adminer

  spa -->|"API calls\n[JSON/HTTP]"| fastapi_backend
  fastapi_backend -->|"Reads/Writes\n[SQL/TCP]"| pg
  adminer -->|"Queries\n[SQL/TCP]"| pg

  fastapi_backend -->|"Sends emails\n[SMTP/TLS]"| smtp
  fastapi_backend -->|"Reports errors\n[HTTPS]"| sentry
```

---

## L3: FastAPI Backend

Components inside the FastAPI Backend container.

```mermaid
%% SCOPE: urn:c4:container:fastapi-backend
graph TB

  spa["React SPA\n[Container]"]
  pg["PostgreSQL\n[Container]"]
  smtp["SMTP Server\n[External System]"]

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    cors_mw["CORS Middleware\n[Component]\nCross-origin request handling"]

    %% KIND: boundary
    auth_deps["Auth Dependencies\n[Component]\nJWT validation, session injection,\nsuperuser gate"]

    subgraph routes["API Routes"]
      %% KIND: router
      login_routes["Login Routes\n[Component]\nOAuth2 token, password\nrecovery and reset"]
      %% KIND: router
      user_routes["User Routes\n[Component]\nUser CRUD, registration,\nprofile management"]
      %% KIND: router
      item_routes["Item Routes\n[Component]\nItem CRUD operations"]
      %% KIND: router
      utils_routes["Utils Routes\n[Component]\nHealth check, test email"]
    end

    subgraph services["Service Layer"]
      %% KIND: data_access
      crud_layer["CRUD Layer\n[Component]\nUser and item data operations,\nauthentication logic"]
      %% KIND: integration
      email_utils["Email Utilities\n[Component]\nTemplate rendering, email sending,\npassword reset tokens"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      security_mod["Security Module\n[Component]\nJWT creation, password hashing\nArgon2 and Bcrypt"]
      %% KIND: service_layer
      config_mod["Config Module\n[Component]\nApp settings via pydantic-settings"]
      %% KIND: storage
      db_engine["DB Engine\n[Component]\nSQLModel and SQLAlchemy engine"]
    end

    %% KIND: data_access
    models_layer["Models Layer\n[Component]\nSQLModel ORM models and\nPydantic request-response schemas"]

  end

  spa -->|"JSON/HTTP"| cors_mw

  cors_mw --> login_routes
  cors_mw --> user_routes
  cors_mw --> item_routes
  cors_mw --> utils_routes

  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  utils_routes --> auth_deps

  login_routes --> crud_layer
  login_routes --> email_utils
  user_routes --> crud_layer
  user_routes --> email_utils
  item_routes --> crud_layer
  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> security_mod
  auth_deps --> security_mod
  auth_deps --> db_engine

  email_utils --> security_mod
  email_utils --> config_mod

  models_layer --> db_engine
  db_engine --> config_mod
  db_engine -->|"SQL/TCP"| pg
  email_utils -->|"SMTP/TLS"| smtp
```

---

## L3: React SPA

Components inside the React SPA container.

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

  fastapi_backend["FastAPI Backend\n[Container]"]

  subgraph spa["React SPA"]

    subgraph spa_routing["Routing"]
      %% KIND: router
      router["TanStack Router\n[Component]\nFile-based route definitions\nand navigation"]
      %% KIND: boundary
      auth_guard["Auth Guard\n[Component]\nRoute protection via beforeLoad\nredirects unauthenticated users"]
    end

    subgraph spa_features["Features"]
      %% KIND: boundary
      dashboard_view["Dashboard\n[Component]\nHome overview page"]
      %% KIND: boundary
      items_view["Items Management\n[Component]\nItem CRUD interface"]
      %% KIND: boundary
      admin_view["Admin Panel\n[Component]\nUser management\nsuperuser only"]
      %% KIND: boundary
      settings_view["User Settings\n[Component]\nProfile, password,\naccount deletion"]
      %% KIND: boundary
      auth_views["Auth Pages\n[Component]\nLogin, signup,\npassword recovery"]
    end

    subgraph spa_services["Services"]
      %% KIND: integration
      api_client["API Client\n[Component]\nGenerated OpenAPI SDK\nAxios-based HTTP client"]
      %% KIND: service_layer
      auth_hook["Auth Hook\n[Component]\nuseAuth: login, signup,\nlogout, current user"]
      %% KIND: service_layer
      query_client["Query Client\n[Component]\nTanStack React Query\nserver-state cache"]
    end

    subgraph spa_ui["UI Framework"]
      %% KIND: boundary
      ui_kit["UI Kit\n[Component]\nshadcn/ui components\nRadix UI primitives"]
      %% KIND: service_layer
      theme_provider["Theme Provider\n[Component]\nDark, light, system\ntheme switching"]
    end

  end

  router --> auth_guard
  auth_guard --> dashboard_view
  auth_guard --> items_view
  auth_guard --> admin_view
  auth_guard --> settings_view
  router --> auth_views

  dashboard_view --> query_client
  items_view --> query_client
  admin_view --> query_client
  settings_view --> query_client
  auth_views --> auth_hook

  auth_hook --> api_client
  query_client --> api_client

  dashboard_view --> ui_kit
  items_view --> ui_kit
  admin_view --> ui_kit
  settings_view --> ui_kit
  auth_views --> ui_kit

  api_client -->|"JSON/HTTP"| fastapi_backend
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [end_user, admin_user, devops]
system_boundary  CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external         CONTAINS [smtp, sentry]

%% ── L3: FastAPI Backend ─────────────────────────────────────────
fastapi_backend  CONTAINS [cors_mw, auth_deps, routes, services, core, models_layer]
  routes         CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services       CONTAINS [crud_layer, email_utils]
  core           CONTAINS [security_mod, config_mod, db_engine]

%% ── L3: React SPA ──────────────────────────────────────────────
spa              CONTAINS [spa_routing, spa_features, spa_services, spa_ui]
  spa_routing    CONTAINS [router, auth_guard]
  spa_features   CONTAINS [dashboard_view, items_view, admin_view, settings_view, auth_views]
  spa_services   CONTAINS [api_client, auth_hook, query_client]
  spa_ui         CONTAINS [ui_kit, theme_provider]
```
