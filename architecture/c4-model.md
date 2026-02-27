# C4 Architecture Model — Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack_fastapi
graph TB

  subgraph users["Users"]
    user["End User<br/><i>Registers, logs in, manages items</i>"]
    admin_user["Administrator<br/><i>Manages users, system configuration</i>"]
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    fullstack_system["Full Stack FastAPI Platform<br/><i>Web application for user<br/>and item management</i>"]
  end

  subgraph external["External Services"]
    smtp_ext["SMTP Email Service<br/><i>Delivers transactional emails</i>"]
    sentry_ext["Sentry<br/><i>Error tracking & performance monitoring</i>"]
    letsencrypt["Let's Encrypt<br/><i>TLS certificate authority</i>"]
  end

  user -->|"Uses web application"| fullstack_system
  admin_user -->|"Administers users & system"| fullstack_system
  fullstack_system -->|"Sends emails via SMTP"| smtp_ext
  fullstack_system -->|"Reports errors"| sentry_ext
  fullstack_system -->|"Obtains TLS certificates"| letsencrypt
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fullstack_fastapi
graph TB

  user["End User"]
  admin_user["Administrator"]

  subgraph system_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik Reverse Proxy<br/><i>HTTPS termination,<br/>subdomain routing</i>"]
    spa["React SPA<br/><i>React 19 · TypeScript · Vite<br/>Served by Nginx</i>"]
    fastapi["FastAPI Backend<br/><i>Python · REST API<br/>4 Uvicorn workers</i>"]
    pg[("PostgreSQL 18<br/><i>Primary relational database</i>")]
    adminer["Adminer<br/><i>Database administration UI</i>"]
  end

  smtp_ext["SMTP Email Service"]
  sentry_ext["Sentry"]
  letsencrypt["Let's Encrypt"]

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.*"| spa
  traefik -->|"api.*"| fastapi
  traefik -->|"adminer.*"| adminer
  traefik -->|"ACME challenge"| letsencrypt
  spa -->|"REST /api/v1 (JSON)"| fastapi
  fastapi -->|"SQL via SQLModel"| pg
  fastapi -->|"SMTP"| smtp_ext
  fastapi -->|"DSN"| sentry_ext
  adminer -->|"SQL"| pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi
graph TB

  spa["React SPA"]

  subgraph fastapi["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware & Dependencies"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Allows cross-origin requests</i>"]
      %% KIND: boundary
      auth_dep["Auth Dependency<br/><i>JWT token validation via<br/>OAuth2PasswordBearer</i>"]
      %% KIND: boundary
      superuser_dep["Superuser Dependency<br/><i>Admin-only authorization gate</i>"]
      %% KIND: data_access
      db_dep["DB Session Dependency<br/><i>Yields SQLModel Session</i>"]
    end

    %% KIND: router
    subgraph routes["API Routes /api/v1"]
      %% KIND: router
      login_routes["Login Routes<br/><i>access-token · password-recovery<br/>reset-password · test-token</i>"]
      %% KIND: router
      user_routes["User Routes<br/><i>CRUD · signup · profile<br/>password change</i>"]
      %% KIND: router
      item_routes["Item Routes<br/><i>CRUD operations<br/>owner-scoped queries</i>"]
      %% KIND: router
      util_routes["Utility Routes<br/><i>health-check · test-email</i>"]
    end

    %% KIND: service_layer
    subgraph core_services["Core Services"]
      %% KIND: service_layer
      security["Security Module<br/><i>JWT creation · Argon2/Bcrypt<br/>password hashing</i>"]
      %% KIND: integration
      email_utils["Email Utilities<br/><i>Jinja2 templates · SMTP sending<br/>password-reset & welcome emails</i>"]
    end

    %% KIND: data_access
    crud_layer["CRUD Layer<br/><i>create/read/update/delete<br/>User & Item operations</i>"]
    %% KIND: data_access
    models_layer["Models & Schemas<br/><i>SQLModel table definitions<br/>Pydantic request/response schemas</i>"]
    %% KIND: storage
    db_engine["Database Engine<br/><i>SQLAlchemy engine<br/>connection pool · init_db</i>"]
    %% KIND: storage
    alembic["Alembic Migrations<br/><i>Schema versioning<br/>UUID migration · cascades</i>"]
  end

  pg[("PostgreSQL 18")]
  smtp_ext["SMTP Email Service"]
  sentry_ext["Sentry"]

  spa -->|"REST API calls"| cors_mw
  cors_mw --> routes
  login_routes --> auth_dep
  user_routes --> auth_dep
  item_routes --> auth_dep
  util_routes --> superuser_dep
  auth_dep --> security
  superuser_dep --> auth_dep
  login_routes --> crud_layer
  login_routes --> security
  login_routes --> email_utils
  user_routes --> crud_layer
  item_routes --> crud_layer
  util_routes --> email_utils
  crud_layer --> models_layer
  crud_layer --> db_dep
  db_dep --> db_engine
  db_engine -->|"SQL via psycopg"| pg
  alembic --> db_engine
  email_utils -->|"SMTP"| smtp_ext
  fastapi -.->|"Sentry SDK"| sentry_ext
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

  fastapi["FastAPI Backend"]

  subgraph spa["React SPA"]

    %% KIND: router
    subgraph routing["Routing Layer · TanStack Router"]
      %% KIND: router
      public_routes["Public Routes<br/><i>Login · Signup<br/>Recover Password · Reset Password</i>"]
      %% KIND: router
      protected_routes["Protected Routes<br/><i>Dashboard · Items · Settings</i>"]
      %% KIND: router
      admin_routes["Admin Routes<br/><i>User management (superuser)</i>"]
    end

    %% KIND: boundary
    subgraph components["UI Components"]
      %% KIND: boundary
      ui_primitives["UI Primitives<br/><i>Radix UI · shadcn/ui<br/>Button · Dialog · Table · Form</i>"]
      %% KIND: boundary
      feature_components["Feature Components<br/><i>AddItem · EditItem · DeleteItem<br/>AddUser · EditUser · UserSettings</i>"]
      %% KIND: boundary
      layout_components["Layout Components<br/><i>AppSidebar · AuthLayout<br/>Footer · Logo · Appearance</i>"]
    end

    %% KIND: service_layer
    subgraph state_layer["State Management"]
      %% KIND: service_layer
      query_cache["TanStack Query<br/><i>Server state cache<br/>query invalidation</i>"]
      %% KIND: service_layer
      form_state["React Hook Form<br/><i>Form state · Zod validation</i>"]
    end

    %% KIND: integration
    api_client["API Client<br/><i>Auto-generated from OpenAPI<br/>Axios · token interceptors</i>"]

    %% KIND: service_layer
    subgraph hooks_layer["Custom Hooks"]
      %% KIND: service_layer
      use_auth["useAuth<br/><i>Login · logout · signup<br/>current user</i>"]
      %% KIND: service_layer
      use_toast["useCustomToast<br/><i>Success & error notifications</i>"]
    end

  end

  public_routes --> layout_components
  protected_routes --> feature_components
  admin_routes --> feature_components
  feature_components --> ui_primitives
  feature_components --> form_state
  protected_routes --> query_cache
  admin_routes --> query_cache
  query_cache --> api_client
  use_auth --> api_client
  api_client -->|"REST /api/v1"| fastapi
  public_routes --> use_auth
  feature_components --> use_toast
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — every subgraph and its children across all levels
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [user, admin_user]
system_boundary  CONTAINS [fullstack_system, traefik, spa, fastapi, pg, adminer]
external         CONTAINS [smtp_ext, sentry_ext, letsencrypt]

%% ── L2 system internals (inside system_boundary) ────────────────
system_boundary  CONTAINS [traefik, spa, fastapi, pg, adminer]

%% ── L3: FastAPI Backend (fastapi) ───────────────────────────────
fastapi          CONTAINS [middleware, routes, core_services, crud_layer, models_layer, db_engine, alembic]
  middleware     CONTAINS [cors_mw, auth_dep, superuser_dep, db_dep]
  routes         CONTAINS [login_routes, user_routes, item_routes, util_routes]
  core_services  CONTAINS [security, email_utils]

%% ── L3: React SPA (spa) ────────────────────────────────────────
spa              CONTAINS [routing, components, state_layer, api_client, hooks_layer]
  routing        CONTAINS [public_routes, protected_routes, admin_routes]
  components     CONTAINS [ui_primitives, feature_components, layout_components]
  state_layer    CONTAINS [query_cache, form_state]
  hooks_layer    CONTAINS [use_auth, use_toast]
```
