# C4 Architecture Model — Full Stack FastAPI Project

<!-- Auto-generated from source code and configuration files. -->

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:full-stack-fastapi-project
graph TD

  subgraph users["Users"]
    user["User<br/>[Person]<br/>Application end-user who manages items"]
    admin["Admin<br/>[Person]<br/>Superuser with elevated privileges"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    system["Full Stack FastAPI Project<br/>[Software System]<br/>Web application with item management<br/>and user authentication"]
  end

  subgraph external["External Systems"]
    smtp_provider["SMTP Provider<br/>[External System]<br/>Email delivery service"]
    sentry["Sentry<br/>[External System]<br/>Error monitoring and tracking"]
    letsencrypt["Let's Encrypt<br/>[External System]<br/>TLS certificate authority"]
  end

  user -->|"Uses web application"| system
  admin -->|"Manages users and content"| system
  system -->|"Sends transactional emails"| smtp_provider
  system -->|"Reports errors via SDK"| sentry
  system -->|"Obtains TLS certificates"| letsencrypt
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:full-stack-fastapi-project
graph TD

  subgraph users["Users"]
    user["User<br/>[Person]<br/>Application end-user"]
    admin["Admin<br/>[Person]<br/>Superuser"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    traefik["Traefik<br/>[Container: Reverse Proxy / Traefik 3.6]<br/>Routes traffic by subdomain, TLS termination"]
    spa["React SPA<br/>[Container: React 19 / TypeScript / Nginx]<br/>Single-page application with Tailwind UI"]
    fastapi_backend["FastAPI Backend<br/>[Container: Python / FastAPI]<br/>REST API, business logic, JWT auth"]
    pg["PostgreSQL<br/>[Container: PostgreSQL 18]<br/>Persists users and items"]
    adminer["Adminer<br/>[Container: Web UI]<br/>Database administration interface"]
  end

  subgraph external["External Systems"]
    smtp_provider["SMTP Provider<br/>[External System]<br/>Email delivery"]
    sentry["Sentry<br/>[External System]<br/>Error monitoring"]
    letsencrypt["Let's Encrypt<br/>[External System]<br/>TLS certificates"]
  end

  user -->|"HTTPS"| traefik
  admin -->|"HTTPS"| traefik
  traefik -->|"dashboard.DOMAIN :80"| spa
  traefik -->|"api.DOMAIN :8000"| fastapi_backend
  traefik -->|"adminer.DOMAIN :8080"| adminer
  traefik -->|"ACME TLS challenge"| letsencrypt
  spa -->|"REST /api/v1/*<br/>via Axios"| fastapi_backend
  fastapi_backend -->|"SQL via psycopg"| pg
  fastapi_backend -->|"SMTP"| smtp_provider
  fastapi_backend -->|"Sentry SDK"| sentry
  adminer -->|"SQL"| pg
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi_backend
graph TD

  subgraph fastapi_backend["FastAPI Backend"]

    subgraph mw["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/>[Component: Starlette CORSMiddleware]<br/>Allows cross-origin requests from frontend"]
    end

    subgraph routes["API Routes"]
      %% KIND: router
      api_router["API Router<br/>[Component: FastAPI APIRouter]<br/>Mounts all route modules under /api/v1"]
      %% KIND: router
      login_routes["Login Routes<br/>[Component: /api/v1/login]<br/>OAuth2 token issuance, password recovery"]
      %% KIND: router
      user_routes["User Routes<br/>[Component: /api/v1/users]<br/>User CRUD, registration, profile"]
      %% KIND: router
      item_routes["Item Routes<br/>[Component: /api/v1/items]<br/>Item CRUD operations"]
      %% KIND: router
      utils_routes["Utils Routes<br/>[Component: /api/v1/utils]<br/>Health check, test email"]
      %% KIND: router
      private_routes["Private Routes<br/>[Component: /api/v1/private]<br/>Dev-only user creation (local env)"]
    end

    subgraph deps["Dependency Injection"]
      %% KIND: boundary
      auth_deps["Auth Dependencies<br/>[Component: FastAPI Depends]<br/>OAuth2 bearer token validation,<br/>get_current_user, superuser guard"]
      %% KIND: data_access
      db_session["DB Session Provider<br/>[Component: FastAPI Depends]<br/>Yields SQLModel Session per request"]
    end

    subgraph services["Services"]
      %% KIND: service_layer
      crud_layer["CRUD Operations<br/>[Component: Python module]<br/>create_user, authenticate,<br/>create_item, get_user_by_email"]
      %% KIND: integration
      email_svc["Email Service<br/>[Component: Python module]<br/>Sends password-reset, new-account,<br/>and test emails via SMTP"]
    end

    subgraph core["Core"]
      %% KIND: service_layer
      config["Configuration<br/>[Component: Pydantic Settings]<br/>Loads env vars, computes CORS origins,<br/>DB URI, SMTP flags"]
      %% KIND: service_layer
      security["Security<br/>[Component: PyJWT / pwdlib]<br/>JWT creation and verification,<br/>Argon2/Bcrypt password hashing"]
      %% KIND: storage
      db_engine["DB Engine<br/>[Component: SQLModel create_engine]<br/>PostgreSQL connection pool via psycopg"]
    end

    subgraph data_layer["Data Layer"]
      %% KIND: data_access
      models_layer["Models and Schemas<br/>[Component: SQLModel]<br/>User, Item tables; request/response schemas"]
      %% KIND: data_access
      alembic_migrations["Alembic Migrations<br/>[Component: Alembic]<br/>Database schema versioning and upgrades"]
    end

  end

  pg["PostgreSQL<br/>[Container]"]
  smtp_provider["SMTP Provider<br/>[External System]"]
  sentry["Sentry<br/>[External System]"]
  spa["React SPA<br/>[Container]"]

  spa -->|"REST /api/v1/*"| cors_mw
  cors_mw --> api_router
  api_router --> login_routes
  api_router --> user_routes
  api_router --> item_routes
  api_router --> utils_routes
  api_router --> private_routes

  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  utils_routes --> auth_deps

  auth_deps --> security
  auth_deps --> db_session

  login_routes --> crud_layer
  user_routes --> crud_layer
  item_routes --> crud_layer
  login_routes --> email_svc
  user_routes --> email_svc
  utils_routes --> email_svc

  crud_layer --> models_layer
  crud_layer --> db_session
  crud_layer --> security

  db_session --> db_engine
  db_engine -->|"SQL via psycopg"| pg
  alembic_migrations -->|"Schema migrations"| pg

  email_svc -->|"SMTP"| smtp_provider

  config -.->|"provides settings"| security
  config -.->|"provides DB URI"| db_engine
  config -.->|"provides SMTP config"| email_svc
  config -.->|"provides CORS origins"| cors_mw
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TD

  subgraph spa["React SPA"]

    subgraph routing["Routing"]
      %% KIND: router
      router["TanStack Router<br/>[Component: File-based Router]<br/>Client-side routing with code splitting"]
      %% KIND: boundary
      auth_layout["Auth Layout<br/>[Component: React Layout]<br/>Split layout for login, signup,<br/>password recovery pages"]
      %% KIND: boundary
      app_layout["App Layout<br/>[Component: React Layout]<br/>Sidebar layout with header/footer<br/>for authenticated pages"]
    end

    subgraph auth_pages["Auth Pages"]
      %% KIND: boundary
      login_page["Login Page<br/>[Component: /login]<br/>Email and password authentication"]
      %% KIND: boundary
      signup_page["Sign Up Page<br/>[Component: /signup]<br/>New user registration"]
      %% KIND: boundary
      recover_page["Recover Password<br/>[Component: /recover-password]<br/>Request password reset email"]
      %% KIND: boundary
      reset_page["Reset Password<br/>[Component: /reset-password]<br/>Set new password with token"]
    end

    subgraph app_pages["App Pages"]
      %% KIND: boundary
      dashboard_page["Dashboard<br/>[Component: /]<br/>Welcome page with user greeting"]
      %% KIND: boundary
      items_feature["Items Management<br/>[Component: /items]<br/>CRUD with data table, add/edit/delete"]
      %% KIND: boundary
      admin_feature["Admin Panel<br/>[Component: /admin]<br/>User management, superuser only"]
      %% KIND: boundary
      settings_feature["User Settings<br/>[Component: /settings]<br/>Profile, password change, account deletion"]
    end

    subgraph services_layer["Services"]
      %% KIND: integration
      api_client["OpenAPI Client<br/>[Component: Generated SDK / Axios]<br/>Type-safe API calls to FastAPI backend"]
      %% KIND: service_layer
      auth_hook["useAuth Hook<br/>[Component: React Hook]<br/>Login, signup, logout, current user"]
      %% KIND: service_layer
      toast_hook["useCustomToast Hook<br/>[Component: React Hook]<br/>Success and error notifications via Sonner"]
    end

    subgraph state_mgmt["State Management"]
      %% KIND: data_access
      query_client["Query Client<br/>[Component: TanStack React Query]<br/>Server state caching, mutations,<br/>401/403 error handling"]
      %% KIND: service_layer
      theme_provider["Theme Provider<br/>[Component: React Context]<br/>Dark, light, and system theme switching"]
    end

    subgraph ui_layer["UI Components"]
      %% KIND: boundary
      ui_components["Shared UI Library<br/>[Component: Radix + Tailwind CSS v4]<br/>Buttons, forms, tables, dialogs,<br/>data-table, sidebar, etc."]
    end

  end

  fastapi_backend["FastAPI Backend<br/>[Container]"]

  router --> auth_layout
  router --> app_layout
  auth_layout --> login_page
  auth_layout --> signup_page
  auth_layout --> recover_page
  auth_layout --> reset_page
  app_layout --> dashboard_page
  app_layout --> items_feature
  app_layout --> admin_feature
  app_layout --> settings_feature

  login_page --> auth_hook
  signup_page --> auth_hook
  recover_page --> api_client
  reset_page --> api_client

  items_feature --> query_client
  admin_feature --> query_client
  settings_feature --> query_client
  dashboard_page --> auth_hook

  auth_hook --> api_client
  query_client --> api_client
  api_client -->|"REST /api/v1/*<br/>via Axios"| fastapi_backend

  items_feature --> ui_components
  admin_feature --> ui_components
  settings_feature --> ui_components
  login_page --> ui_components
  signup_page --> ui_components
  recover_page --> ui_components
  reset_page --> ui_components

  items_feature --> toast_hook
  admin_feature --> toast_hook
  settings_feature --> toast_hook
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers L1, L2, and all L3 diagrams
%% Every subgraph and its direct children are listed.
%% Indentation denotes nesting depth.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 / L2 top-level groups ───────────────────────────────────────
users              CONTAINS [user, admin]
system_boundary    CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external           CONTAINS [smtp_provider, sentry, letsencrypt]

%% ── L3: FastAPI Backend (urn:c4:container:fastapi_backend) ─────────
fastapi_backend    CONTAINS [mw, routes, deps, services, core, data_layer]
  mw               CONTAINS [cors_mw]
  routes           CONTAINS [api_router, login_routes, user_routes, item_routes, utils_routes, private_routes]
  deps             CONTAINS [auth_deps, db_session]
  services         CONTAINS [crud_layer, email_svc]
  core             CONTAINS [config, security, db_engine]
  data_layer       CONTAINS [models_layer, alembic_migrations]

%% ── L3: React SPA (urn:c4:container:spa) ───────────────────────────
spa                CONTAINS [routing, auth_pages, app_pages, services_layer, state_mgmt, ui_layer]
  routing          CONTAINS [router, auth_layout, app_layout]
  auth_pages       CONTAINS [login_page, signup_page, recover_page, reset_page]
  app_pages        CONTAINS [dashboard_page, items_feature, admin_feature, settings_feature]
  services_layer   CONTAINS [api_client, auth_hook, toast_hook]
  state_mgmt       CONTAINS [query_client, theme_provider]
  ui_layer         CONTAINS [ui_components]
```
