# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fastapi-fullstack

  subgraph users["Users"]
    user["User\n[Person]\nInteracts with the web application"]
    admin_user["Admin\n[Person]\nSuperuser managing platform"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    platform["Full Stack FastAPI Platform\n[Software System]\nWeb application with REST API"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Service\n[External System]\nEmail delivery"]
    sentry["Sentry\n[External System]\nError monitoring"]
    letsencrypt["Let's Encrypt\n[External System]\nTLS certificate authority"]
  end

  user -->|"Uses web application via HTTPS"| platform
  admin_user -->|"Manages users and system via HTTPS"| platform
  platform -->|"Sends transactional emails via SMTP"| smtp
  platform -->|"Reports errors and traces"| sentry
  platform -->|"Obtains TLS certificates via ACME"| letsencrypt
```

---

## L2: Container Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fastapi-fullstack

  subgraph users["Users"]
    user["User\n[Person]"]
    admin_user["Admin\n[Person]"]
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik\n[Container: Reverse Proxy]\nRoutes traffic, TLS termination"]
    spa["React SPA\n[Container: React 19 / Vite / Nginx]\nSingle-page application"]
    fastapi["FastAPI Backend\n[Container: Python / FastAPI]\nREST API, business logic"]
    pg["PostgreSQL\n[Container: PostgreSQL 18]\nRelational data store"]
    adminer["Adminer\n[Container: PHP]\nDatabase admin UI"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Service\n[External System]\nEmail delivery"]
    sentry["Sentry\n[External System]\nError monitoring"]
    letsencrypt["Let's Encrypt\n[External System]\nTLS certificates"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"Routes dashboard.*"| spa
  traefik -->|"Routes api.*"| fastapi
  traefik -->|"Routes adminer.*"| adminer
  spa -->|"REST API calls /api/v1/*"| fastapi
  fastapi -->|"SQL via psycopg"| pg
  adminer -->|"SQL"| pg
  fastapi -->|"Sends emails via SMTP"| smtp
  fastapi -->|"Reports errors"| sentry
  traefik -->|"ACME protocol"| letsencrypt
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi

  spa["React SPA\n[Container]"]
  pg["PostgreSQL\n[Container]"]
  smtp["SMTP Service\n[External System]"]
  sentry["Sentry\n[External System]"]

  subgraph fastapi["FastAPI Backend"]

    subgraph mw["Middleware & Dependencies"]
      %% KIND: boundary
      cors_mw["CORS Middleware\n[Component]\nCross-origin request handling"]
      %% KIND: boundary
      auth_deps["Auth Dependencies\n[Component]\nOAuth2 bearer, JWT validation,\ncurrent-user injection"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      api_router["API Router\n[Component]\nAggregates all route modules\nunder /api/v1"]
      %% KIND: router
      login_routes["Login Routes\n[Component]\nToken auth, password recovery/reset"]
      %% KIND: router
      users_routes["Users Routes\n[Component]\nUser CRUD, signup, profile"]
      %% KIND: router
      items_routes["Items Routes\n[Component]\nItem CRUD operations"]
      %% KIND: router
      utils_routes["Utils Routes\n[Component]\nHealth check, test email"]
      %% KIND: router
      private_routes["Private Routes\n[Component]\nLocal-only user creation"]
    end

    subgraph services["Service Layer"]
      %% KIND: service_layer
      crud_layer["CRUD Module\n[Component]\nUser & item data operations,\nauthentication logic"]
      %% KIND: integration
      email_utils["Email & Token Utils\n[Component]\nEmail rendering, sending,\npassword-reset tokens"]
    end

    subgraph data_layer["Data & Core"]
      %% KIND: data_access
      models_layer["SQLModel Models\n[Component]\nUser, Item tables and\nPydantic schemas"]
      %% KIND: storage
      db_layer["DB Engine\n[Component]\nSQLAlchemy engine, session factory,\ndatabase initialization"]
      %% KIND: service_layer
      security["Security\n[Component]\nJWT creation, password hashing\n(Argon2/Bcrypt)"]
      %% KIND: service_layer
      config["Settings\n[Component]\nPydantic-based configuration\nfrom environment variables"]
    end

  end

  spa -->|"HTTP requests"| cors_mw
  cors_mw --> auth_deps
  auth_deps --> api_router
  api_router --> login_routes
  api_router --> users_routes
  api_router --> items_routes
  api_router --> utils_routes
  api_router --> private_routes
  login_routes --> crud_layer
  login_routes --> email_utils
  users_routes --> crud_layer
  items_routes --> crud_layer
  utils_routes --> email_utils
  crud_layer --> models_layer
  crud_layer --> security
  crud_layer --> db_layer
  email_utils --> smtp
  email_utils --> security
  db_layer --> pg
  auth_deps --> security
  auth_deps --> db_layer
  config -.->|"provides settings"| security
  config -.->|"provides settings"| db_layer
  config -.->|"provides settings"| email_utils
  fastapi -.->|"init on startup"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  user["User\n[Person]"]
  admin_user["Admin\n[Person]"]
  fastapi["FastAPI Backend\n[Container]"]

  subgraph spa["React SPA"]

    subgraph routing_layer["Routing"]
      %% KIND: router
      routing["TanStack Router\n[Component]\nFile-based routing with\nauth guards and code-splitting"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["OpenAPI Client\n[Component]\nAuto-generated API client\nusing Axios"]
      %% KIND: service_layer
      auth_hooks["Auth Hooks\n[Component]\nuseAuth: login, logout, signup,\ncurrent-user management"]
      %% KIND: service_layer
      theme_provider["Theme Provider\n[Component]\nDark/light/system theme\nvia React Context"]
    end

    subgraph features["Feature Modules"]
      %% KIND: boundary
      auth_pages["Auth Pages\n[Component]\nLogin, Signup, Password\nRecovery, Password Reset"]
      %% KIND: boundary
      dashboard_feature["Dashboard\n[Component]\nAuthenticated landing page"]
      %% KIND: boundary
      items_feature["Items Module\n[Component]\nItem CRUD with DataTable,\npagination, actions"]
      %% KIND: boundary
      admin_feature["Admin Module\n[Component]\nUser management\n(superuser only)"]
      %% KIND: boundary
      settings_feature["Settings Module\n[Component]\nProfile, password change,\naccount deletion"]
    end

    subgraph ui_layer["UI Layer"]
      %% KIND: boundary
      ui_primitives["UI Primitives\n[Component]\nshadcn/Radix components:\nButton, Dialog, Form, Table, etc."]
      %% KIND: boundary
      common_components["Common Components\n[Component]\nSidebar, DataTable, Layout,\nError/NotFound pages"]
    end

  end

  user -->|"HTTPS"| routing
  admin_user -->|"HTTPS"| routing
  routing --> auth_pages
  routing --> dashboard_feature
  routing --> items_feature
  routing --> admin_feature
  routing --> settings_feature
  auth_pages --> auth_hooks
  dashboard_feature --> auth_hooks
  items_feature --> api_client
  admin_feature --> api_client
  settings_feature --> api_client
  auth_hooks --> api_client
  api_client -->|"REST /api/v1/*"| fastapi
  auth_pages --> ui_primitives
  items_feature --> ui_primitives
  admin_feature --> ui_primitives
  settings_feature --> ui_primitives
  dashboard_feature --> common_components
  items_feature --> common_components
  admin_feature --> common_components
  settings_feature --> common_components
```

---

## L3: Traefik

```mermaid
graph TB
  %% SCOPE: urn:c4:container:traefik

  user["User\n[Person]"]
  admin_user["Admin\n[Person]"]
  spa["React SPA\n[Container]"]
  fastapi["FastAPI Backend\n[Container]"]
  adminer["Adminer\n[Container]"]
  letsencrypt["Let's Encrypt\n[External System]"]

  subgraph traefik["Traefik"]
    %% KIND: boundary
    entrypoints["Entrypoints\n[Component]\nHTTP (:80) and HTTPS (:443)\nlisteners"]
    %% KIND: router
    http_routers["HTTP Routers\n[Component]\nHost-based routing rules:\ndashboard.*, api.*, adminer.*"]
    %% KIND: service_layer
    https_redirect["HTTPS Redirect\n[Component]\nMiddleware redirecting\nHTTP to HTTPS"]
    %% KIND: integration
    cert_resolver["Cert Resolver\n[Component]\nACME / Let's Encrypt\ncertificate management"]
    %% KIND: service_layer
    lb_services["Load Balancer Services\n[Component]\nBackend service discovery\nand health checks"]
  end

  user -->|"HTTP/HTTPS"| entrypoints
  admin_user -->|"HTTP/HTTPS"| entrypoints
  entrypoints --> https_redirect
  https_redirect --> http_routers
  entrypoints -->|"HTTPS"| http_routers
  http_routers -->|"dashboard.*"| lb_services
  http_routers -->|"api.*"| lb_services
  http_routers -->|"adminer.*"| lb_services
  lb_services --> spa
  lb_services --> fastapi
  lb_services --> adminer
  cert_resolver -->|"ACME"| letsencrypt
  http_routers -.-> cert_resolver
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% Every subgraph and its children across L1, L2, and L3.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [spa, fastapi, pg, traefik, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 containers (same as platform_boundary at L1) ───────────────
%% platform_boundary is zoomed into at L2 revealing:
%%   spa, fastapi, pg, traefik, adminer

%% ── L3: FastAPI Backend (urn:c4:container:fastapi) ─────────────────
fastapi            CONTAINS [mw, routes, services, data_layer]
  mw               CONTAINS [cors_mw, auth_deps]
  routes            CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes, private_routes]
  services          CONTAINS [crud_layer, email_utils]
  data_layer        CONTAINS [models_layer, db_layer, security, config]

%% ── L3: React SPA (urn:c4:container:spa) ───────────────────────────
spa                CONTAINS [routing_layer, services_layer, features, ui_layer]
  routing_layer    CONTAINS [routing]
  services_layer   CONTAINS [api_client, auth_hooks, theme_provider]
  features         CONTAINS [auth_pages, dashboard_feature, items_feature, admin_feature, settings_feature]
  ui_layer         CONTAINS [ui_primitives, common_components]

%% ── L3: Traefik (urn:c4:container:traefik) ─────────────────────────
traefik            CONTAINS [entrypoints, http_routers, https_redirect, cert_resolver, lb_services]
```
