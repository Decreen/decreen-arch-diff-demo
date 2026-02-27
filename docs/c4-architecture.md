# C4 Architecture Model — Full Stack FastAPI Application

> C4 model for the [Full Stack FastAPI Template](https://github.com/fastapi/full-stack-fastapi-template).
> Covers L1 (System Context), L2 (Container), and L3 (Component) views.

---

## L1: System Context

```mermaid
graph TD
  %% SCOPE: urn:c4:system-context:fullstack_fastapi

  subgraph users["Users"]
    user["End User
    <i>Person</i>
    Browses dashboard, manages own items"]
    admin["Administrator
    <i>Person</i>
    Manages users &amp; system configuration"]
  end

  subgraph system_boundary["Full Stack FastAPI Application"]
    system_node["FastAPI Full Stack App
    <i>Software System</i>
    Web application with authentication,
    user management, and item CRUD"]
  end

  subgraph external["External Services"]
    smtp["SMTP Email Service
    <i>External System</i>
    Delivers password-recovery
    and notification emails"]
    sentry["Sentry
    <i>External System</i>
    Error tracking &amp; performance monitoring"]
  end

  user -->|"Uses web dashboard"| system_node
  admin -->|"Manages users &amp; config"| system_node
  system_node -->|"Sends transactional emails"| smtp
  system_node -->|"Reports errors"| sentry
```

---

## L2: Container Diagram

```mermaid
graph TD
  %% SCOPE: urn:c4:system:fullstack_fastapi

  subgraph users["Users"]
    user["End User
    <i>Person</i>"]
    admin["Administrator
    <i>Person</i>"]
  end

  subgraph system_boundary["Full Stack FastAPI Application"]
    traefik["Traefik
    <i>Reverse Proxy · Traefik 3.6</i>
    HTTPS termination, routing,
    Let's Encrypt certificates"]
    spa["React SPA
    <i>Container · React 19 · TypeScript · Vite</i>
    Dashboard UI served by Nginx"]
    fastapi["FastAPI Backend
    <i>Container · Python 3.10 · FastAPI</i>
    REST API: authentication, users, items"]
    pg[("PostgreSQL
    <i>Database · PostgreSQL 18</i>
    Stores users, items, relationships")]
    adminer["Adminer
    <i>Container · Adminer</i>
    Database administration UI"]
  end

  subgraph external["External Services"]
    smtp["SMTP Email Service
    <i>External System</i>"]
    sentry["Sentry
    <i>External System</i>"]
  end

  user -->|"HTTPS"| traefik
  admin -->|"HTTPS"| traefik
  traefik -->|"dashboard.* → port 80"| spa
  traefik -->|"api.* → port 8000"| fastapi
  traefik -->|"adminer.* → port 8080"| adminer
  spa -->|"JSON / HTTPS"| fastapi
  fastapi -->|"SQL via psycopg"| pg
  adminer -->|"SQL"| pg
  fastapi -->|"SMTP"| smtp
  fastapi -->|"Sentry SDK"| sentry
```

---

## L3: FastAPI Backend

```mermaid
graph TD
  %% SCOPE: urn:c4:container:fastapi

  subgraph fastapi["FastAPI Backend"]

    subgraph middleware_group["Middleware &amp; Dependencies"]
      %% KIND: boundary
      auth_deps["Auth Dependencies
      <i>Component</i>
      OAuth2PasswordBearer, JWT validation,
      get_current_user, get_current_active_superuser"]
    end

    subgraph routes_group["API Routes"]
      %% KIND: router
      login_routes["Login Routes
      <i>FastAPI Router</i>
      POST /login/access-token
      Password recovery &amp; reset"]
      %% KIND: router
      user_routes["User Routes
      <i>FastAPI Router</i>
      CRUD users, signup,
      GET/PATCH /users/me"]
      %% KIND: router
      item_routes["Item Routes
      <i>FastAPI Router</i>
      CRUD items,
      owner-scoped queries"]
      %% KIND: router
      utils_routes["Utils Routes
      <i>FastAPI Router</i>
      GET /health-check
      POST /test-email"]
    end

    subgraph services_group["Services"]
      %% KIND: data_access
      crud["CRUD Operations
      <i>Component</i>
      create_user, authenticate,
      create_item, update_user"]
      %% KIND: integration
      email_util["Email Utilities
      <i>Component</i>
      SMTP sending via emails lib,
      Jinja2 templates from MJML"]
    end

    subgraph data_access_group["Data Access"]
      %% KIND: data_access
      models["Models &amp; Schemas
      <i>Component · SQLModel</i>
      User, Item ORM models,
      Pydantic request/response schemas"]
      %% KIND: storage
      db_engine["Database Engine
      <i>Component · SQLAlchemy</i>
      Engine, session factory,
      init_db, connection pooling"]
    end

    subgraph core_group["Core"]
      %% KIND: service_layer
      security["Security Module
      <i>Component</i>
      JWT creation HS256,
      Argon2 &amp; Bcrypt password hashing"]
      %% KIND: service_layer
      config["Configuration
      <i>Component · Pydantic Settings</i>
      DB, SMTP, JWT, CORS,
      Sentry, environment settings"]
    end

  end

  pg[("PostgreSQL
  <i>Database</i>")]
  smtp["SMTP Email Service
  <i>External</i>"]

  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  utils_routes --> auth_deps

  login_routes --> crud
  login_routes --> security
  user_routes --> crud
  item_routes --> crud
  utils_routes --> email_util

  crud --> models
  crud --> db_engine
  crud --> security

  db_engine --> pg
  email_util --> smtp

  config -.->|"settings"| security
  config -.->|"settings"| db_engine
  config -.->|"settings"| email_util
```

---

## L3: React SPA

```mermaid
graph TD
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    subgraph routing_group["Routing"]
      %% KIND: router
      router["TanStack Router
      <i>Component</i>
      File-based routing, protected
      layout wrapper, code splitting"]
    end

    subgraph features_group["Features"]
      %% KIND: boundary
      auth_pages["Auth Pages
      <i>Component</i>
      Login, Signup,
      Recover Password, Reset Password"]
      %% KIND: boundary
      dashboard["Dashboard
      <i>Component</i>
      Main dashboard view
      after authentication"]
      %% KIND: boundary
      items_feat["Items Management
      <i>Component</i>
      Add, Edit, Delete items
      with DataTable"]
      %% KIND: boundary
      admin_feat["Admin Panel
      <i>Component</i>
      User management
      superuser only"]
      %% KIND: boundary
      settings_feat["User Settings
      <i>Component</i>
      Profile editing, password
      change, account deletion"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["API Client
      <i>Component · @hey-api/openapi-ts</i>
      Auto-generated TypeScript SDK
      from OpenAPI specification"]
      %% KIND: service_layer
      query_mgmt["State Management
      <i>Component · TanStack Query</i>
      Server state caching,
      mutations, query invalidation"]
    end

    subgraph ui_layer["UI Layer"]
      %% KIND: boundary
      common_ui["Common Components
      <i>Component</i>
      DataTable, Sidebar,
      AuthLayout, ThemeProvider, Logo"]
      %% KIND: boundary
      ui_lib["UI Primitives
      <i>Component · shadcn/ui</i>
      Button, Dialog, Form, Input,
      Card, Tabs via Radix UI + Tailwind"]
    end

  end

  fastapi["FastAPI Backend
  <i>Container</i>"]

  router --> auth_pages
  router --> dashboard
  router --> items_feat
  router --> admin_feat
  router --> settings_feat

  auth_pages --> api_client
  dashboard --> api_client
  items_feat --> api_client
  admin_feat --> api_client
  settings_feat --> api_client

  api_client --> query_mgmt
  query_mgmt -->|"HTTP / JSON"| fastapi

  auth_pages --> common_ui
  dashboard --> common_ui
  items_feat --> common_ui
  admin_feat --> common_ui
  settings_feat --> common_ui

  common_ui --> ui_lib
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Application
%% Every subgraph from every diagram appears as a CONTAINS entry.
%% Indentation denotes nesting depth.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ──────────────────────────────────────────
users            CONTAINS [user, admin]
system_boundary  CONTAINS [traefik, spa, fastapi, pg, adminer]
external         CONTAINS [smtp, sentry]

%% ── L2 → L3 internal containment ────────────────────────────────
fastapi CONTAINS [middleware_group, routes_group, services_group, data_access_group, core_group]
  middleware_group  CONTAINS [auth_deps]
  routes_group      CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services_group    CONTAINS [crud, email_util]
  data_access_group CONTAINS [models, db_engine]
  core_group        CONTAINS [security, config]

spa CONTAINS [routing_group, features_group, services_layer, ui_layer]
  routing_group  CONTAINS [router]
  features_group CONTAINS [auth_pages, dashboard, items_feat, admin_feat, settings_feat]
  services_layer CONTAINS [api_client, query_mgmt]
  ui_layer       CONTAINS [common_ui, ui_lib]
```

---

## Stable ID Reference

| ID | Name | Level(s) | Type |
|---|---|---|---|
| `user` | End User | L1, L2 | Person |
| `admin` | Administrator | L1, L2 | Person |
| `system_node` | FastAPI Full Stack App | L1 | System |
| `traefik` | Traefik | L2 | Reverse Proxy |
| `spa` | React SPA | L2, L3 scope | Container |
| `fastapi` | FastAPI Backend | L2, L3 scope | Container |
| `pg` | PostgreSQL | L2, L3-FastAPI | Database |
| `adminer` | Adminer | L2 | Container |
| `smtp` | SMTP Email Service | L1, L2, L3-FastAPI | External System |
| `sentry` | Sentry | L1, L2 | External System |
| `auth_deps` | Auth Dependencies | L3-FastAPI | Component |
| `login_routes` | Login Routes | L3-FastAPI | Component |
| `user_routes` | User Routes | L3-FastAPI | Component |
| `item_routes` | Item Routes | L3-FastAPI | Component |
| `utils_routes` | Utils Routes | L3-FastAPI | Component |
| `crud` | CRUD Operations | L3-FastAPI | Component |
| `email_util` | Email Utilities | L3-FastAPI | Component |
| `models` | Models & Schemas | L3-FastAPI | Component |
| `db_engine` | Database Engine | L3-FastAPI | Component |
| `security` | Security Module | L3-FastAPI | Component |
| `config` | Configuration | L3-FastAPI | Component |
| `router` | TanStack Router | L3-SPA | Component |
| `auth_pages` | Auth Pages | L3-SPA | Component |
| `dashboard` | Dashboard | L3-SPA | Component |
| `items_feat` | Items Management | L3-SPA | Component |
| `admin_feat` | Admin Panel | L3-SPA | Component |
| `settings_feat` | User Settings | L3-SPA | Component |
| `api_client` | API Client | L3-SPA | Component |
| `query_mgmt` | State Management | L3-SPA | Component |
| `common_ui` | Common Components | L3-SPA | Component |
| `ui_lib` | UI Primitives | L3-SPA | Component |
