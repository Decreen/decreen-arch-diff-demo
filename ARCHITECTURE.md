# C4 Architecture Model – Full Stack FastAPI Platform

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI template.

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    user(["End User<br/>Browses dashboard, manages items"])
    admin_user(["Administrator<br/>Manages users, administers system"])
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    spa["React SPA<br/><i>TypeScript · React 19</i>"]
    fastapi_api["FastAPI Backend<br/><i>Python · FastAPI</i>"]
    pg[("PostgreSQL<br/><i>Relational Database</i>")]
  end

  subgraph external["External Systems"]
    sentry["Sentry<br/><i>Error Tracking</i>"]
    smtp["SMTP Server<br/><i>Email Delivery</i>"]
  end

  user -->|"HTTPS"| spa
  admin_user -->|"HTTPS"| spa
  spa -->|"REST API /api/v1"| fastapi_api
  fastapi_api -->|"SQL"| pg
  fastapi_api -.->|"Error reports"| sentry
  fastapi_api -->|"Transactional emails"| smtp
```

---

## L2: Container Diagram

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack-fastapi

  subgraph users["Users"]
    user(["End User<br/>Browses dashboard, manages items"])
    admin_user(["Administrator<br/>Manages users via admin panel"])
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik<br/><i>Reverse Proxy · TLS termination<br/>Routes by subdomain</i>"]
    spa["React SPA<br/><i>Nginx · React 19 · Vite 7<br/>TanStack Router & Query · Tailwind 4</i>"]
    fastapi_api["FastAPI Backend<br/><i>Python · FastAPI · SQLModel<br/>JWT Auth · CORS · Alembic</i>"]
    pg[("PostgreSQL 18<br/><i>Relational Database<br/>Users, Items tables</i>")]
    adminer["Adminer<br/><i>Database Admin UI</i>"]
  end

  subgraph external["External Systems"]
    sentry["Sentry<br/><i>Error Tracking SaaS</i>"]
    smtp["SMTP Server<br/><i>Email Delivery<br/>Mailcatcher in dev</i>"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"dashboard.DOMAIN"| spa
  traefik -->|"api.DOMAIN"| fastapi_api
  traefik -->|"adminer.DOMAIN"| adminer
  spa -->|"REST /api/v1/*<br/>JSON over HTTPS"| fastapi_api
  fastapi_api -->|"postgresql+psycopg"| pg
  adminer -->|"SQL"| pg
  fastapi_api -.->|"Sentry SDK"| sentry
  fastapi_api -->|"SMTP"| smtp
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_api

  subgraph fastapi_api["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Allows configured origins</i>"]
      %% KIND: integration
      sentry_mw["Sentry Integration<br/><i>Error tracking · non-local only</i>"]
    end

    %% KIND: router
    subgraph routes["API Routes · /api/v1"]
      %% KIND: router
      login_routes["Login Routes<br/><i>POST /login/access-token<br/>Password recovery & reset</i>"]
      %% KIND: router
      users_routes["Users Routes<br/><i>/users · CRUD, signup<br/>Profile, password update</i>"]
      %% KIND: router
      items_routes["Items Routes<br/><i>/items · Owner-scoped CRUD</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>/utils · Health check<br/>Test email endpoint</i>"]
    end

    %% KIND: service_layer
    subgraph auth_layer["Auth Layer"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies<br/><i>get_current_user · JWT validation<br/>OAuth2PasswordBearer</i>"]
      %% KIND: service_layer
      core_security["Security Module<br/><i>JWT creation · HS256<br/>Argon2 & Bcrypt hashing</i>"]
    end

    %% KIND: data_access
    subgraph data_layer["Data Access Layer"]
      %% KIND: data_access
      crud_layer["CRUD Operations<br/><i>create/read/update/authenticate<br/>Users & Items</i>"]
      %% KIND: storage
      models_layer["SQLModel Models<br/><i>User, Item, Token<br/>Pydantic schemas</i>"]
      %% KIND: data_access
      core_db["Database Engine<br/><i>SQLAlchemy engine<br/>Session management</i>"]
      %% KIND: data_access
      alembic["Alembic Migrations<br/><i>Schema versioning<br/>Prestart upgrade head</i>"]
    end

    %% KIND: integration
    subgraph integrations["Integrations"]
      %% KIND: integration
      email_utils["Email Utilities<br/><i>SMTP sending · Jinja2 templates<br/>Password reset tokens</i>"]
      %% KIND: integration
      core_config["Configuration<br/><i>Pydantic Settings<br/>Env-based config</i>"]
    end

  end

  spa["React SPA"]
  pg[("PostgreSQL")]
  sentry["Sentry"]
  smtp["SMTP Server"]

  spa -->|"REST /api/v1/*"| cors_mw
  cors_mw --> routes
  login_routes --> auth_deps
  users_routes --> auth_deps
  items_routes --> auth_deps
  auth_deps --> core_security
  login_routes --> crud_layer
  users_routes --> crud_layer
  items_routes --> crud_layer
  login_routes --> email_utils
  utils_routes --> email_utils
  crud_layer --> models_layer
  crud_layer --> core_db
  core_db -->|"postgresql+psycopg"| pg
  alembic -->|"DDL migrations"| pg
  email_utils -->|"SMTP"| smtp
  sentry_mw -.->|"Error reports"| sentry
  core_config -.-> core_security
  core_config -.-> core_db
  core_config -.-> email_utils
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  subgraph spa["React SPA"]

    %% KIND: router
    subgraph routing["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router<br/><i>File-based routing<br/>__root · _layout · public routes</i>"]
      %% KIND: boundary
      layout_guard["Layout Guard<br/><i>isLoggedIn check<br/>Redirects to /login</i>"]
    end

    %% KIND: service_layer
    subgraph state_mgmt["State Management"]
      %% KIND: service_layer
      query_client["TanStack Query Client<br/><i>Server state & caching<br/>401/403 error handling</i>"]
      %% KIND: service_layer
      auth_hooks["Auth Hooks · useAuth<br/><i>Login, logout, signup<br/>JWT in localStorage</i>"]
    end

    %% KIND: boundary
    subgraph features["Feature Modules"]
      %% KIND: boundary
      dashboard_feature["Dashboard<br/><i>Home page · / route</i>"]
      %% KIND: boundary
      items_feature["Items Management<br/><i>DataTable · Add/Edit/Delete<br/>/items route</i>"]
      %% KIND: boundary
      admin_feature["Admin Panel<br/><i>User management · superuser only<br/>/admin route</i>"]
      %% KIND: boundary
      settings_feature["User Settings<br/><i>Profile, password, delete account<br/>/settings route</i>"]
      %% KIND: boundary
      auth_pages["Auth Pages<br/><i>/login · /signup<br/>/recover-password · /reset-password</i>"]
    end

    %% KIND: integration
    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>Auto-generated via @hey-api<br/>Axios HTTP client</i>"]
      %% KIND: integration
      theme_provider["Theme Provider<br/><i>Dark / Light / System<br/>React Context</i>"]
    end

    %% KIND: boundary
    subgraph ui_layer["UI Layer"]
      %% KIND: boundary
      sidebar["Sidebar Navigation<br/><i>App nav · user menu<br/>Theme toggle</i>"]
      %% KIND: boundary
      ui_components["UI Component Library<br/><i>shadcn/ui · Radix primitives<br/>Tailwind CSS 4</i>"]
    end

  end

  fastapi_api["FastAPI Backend"]

  tanstack_router --> layout_guard
  layout_guard -->|"Protected routes"| features
  tanstack_router -->|"Public routes"| auth_pages
  dashboard_feature --> query_client
  items_feature --> query_client
  admin_feature --> query_client
  settings_feature --> query_client
  auth_pages --> auth_hooks
  auth_hooks --> api_client
  query_client --> api_client
  api_client -->|"REST /api/v1/*"| fastapi_api
  features --> ui_layer
  sidebar --> tanstack_router
  theme_provider -.-> ui_components
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [user, admin_user]
system_boundary    CONTAINS [traefik, spa, fastapi_api, pg, adminer]
external           CONTAINS [sentry, smtp]

%% ── L2 → L3 internal containment ───────────────────────────────
fastapi_api        CONTAINS [middleware, routes, auth_layer, data_layer, integrations]
  middleware       CONTAINS [cors_mw, sentry_mw]
  routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  auth_layer       CONTAINS [auth_deps, core_security]
  data_layer       CONTAINS [crud_layer, models_layer, core_db, alembic]
  integrations     CONTAINS [email_utils, core_config]

spa                CONTAINS [routing, state_mgmt, features, services_layer, ui_layer]
  routing          CONTAINS [tanstack_router, layout_guard]
  state_mgmt       CONTAINS [query_client, auth_hooks]
  features         CONTAINS [dashboard_feature, items_feature, admin_feature, settings_feature, auth_pages]
  services_layer   CONTAINS [api_client, theme_provider]
  ui_layer         CONTAINS [sidebar, ui_components]
```
