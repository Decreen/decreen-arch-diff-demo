# C4 Architecture Model – Full Stack FastAPI Platform

---

## L1: System Context

```mermaid
flowchart TB
  %% SCOPE: urn:c4:system:fullstack_fastapi_platform

  subgraph users["Users"]
    user["End User\n<i>Person</i>\nManages personal items\nvia web application"]
    admin["Administrator\n<i>Person / Superuser</i>\nManages users and\nsystem configuration"]
  end

  subgraph system_boundary["Full Stack FastAPI Platform"]
    fullstack_platform["Full Stack FastAPI Platform\n<i>Software System</i>\nUser and item management\nweb application with REST API"]
  end

  subgraph external["External Systems"]
    smtp_ext["SMTP Provider\n<i>External System</i>\nTransactional email delivery\n(password reset, new account)"]
    sentry_ext["Sentry\n<i>External System</i>\nError monitoring\nand performance tracking"]
    letsencrypt["Let's Encrypt\n<i>External System</i>\nTLS certificate authority"]
  end

  user -->|"Uses [HTTPS]"| fullstack_platform
  admin -->|"Administers [HTTPS]"| fullstack_platform
  fullstack_platform -->|"Sends emails [SMTP/TLS]"| smtp_ext
  fullstack_platform -->|"Reports errors [HTTPS]"| sentry_ext
  fullstack_platform -->|"Obtains TLS certs [ACME]"| letsencrypt
```

---

## L2: Container Diagram

```mermaid
flowchart TB
  %% SCOPE: urn:c4:system:fullstack_fastapi_platform

  user["End User\n<i>Person</i>"]
  admin["Administrator\n<i>Person</i>"]

  subgraph system_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik\n<i>Container: Reverse Proxy</i>\nHTTPS termination, routing\nby subdomain"]
    spa["React SPA\n<i>Container: Nginx + React 19 / TypeScript</i>\nSingle-page application\nserved as static files"]
    fastapi_backend["FastAPI Backend\n<i>Container: Python 3.10 / FastAPI</i>\nREST API, business logic,\nJWT authentication"]
    pg["PostgreSQL 18\n<i>Container: Database</i>\nStores users, items,\nand application data"]
    adminer_ctr["Adminer\n<i>Container: Web UI</i>\nDatabase administration\ninterface"]
  end

  subgraph external["External Systems"]
    smtp_ext["SMTP Provider\n<i>External System</i>"]
    sentry_ext["Sentry\n<i>External System</i>"]
    letsencrypt["Let's Encrypt\n<i>External System</i>"]
  end

  user -->|"HTTPS"| traefik
  admin -->|"HTTPS"| traefik
  traefik -->|"dashboard.DOMAIN\n[HTTP proxy]"| spa
  traefik -->|"api.DOMAIN\n[HTTP proxy]"| fastapi_backend
  traefik -->|"adminer.DOMAIN\n[HTTP proxy]"| adminer_ctr
  spa -->|"REST API calls\n[HTTPS / JSON]"| fastapi_backend
  fastapi_backend -->|"SQL queries\n[psycopg / TCP:5432]"| pg
  adminer_ctr -->|"SQL\n[TCP:5432]"| pg
  fastapi_backend -->|"Sends emails\n[SMTP/TLS]"| smtp_ext
  fastapi_backend -->|"Reports errors\n[HTTPS]"| sentry_ext
  traefik -->|"ACME challenge\n[HTTPS]"| letsencrypt
```

---

## L3: FastAPI Backend

```mermaid
flowchart TB
  %% SCOPE: urn:c4:container:fastapi_backend

  spa["React SPA\n<i>Container</i>"]
  pg["PostgreSQL 18\n<i>Container</i>"]
  smtp_ext["SMTP Provider\n<i>External</i>"]
  sentry_ext["Sentry\n<i>External</i>"]

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware"]
      cors_mw["CORS Middleware\n<i>Starlette CORSMiddleware</i>\nEnforces allowed origins,\nmethods, headers"]
      sentry_integration["Sentry Integration\n<i>sentry-sdk[fastapi]</i>\nCaptures exceptions\nand traces"]
    end

    %% KIND: boundary
    subgraph auth["Auth Dependencies (deps.py)"]
      oauth2_bearer["OAuth2PasswordBearer\n<i>FastAPI Security</i>\nExtracts JWT from\nAuthorization header"]
      session_dep["SessionDep\n<i>Dependency</i>\nProvides SQLModel Session\nper request"]
      get_current_user_dep["get_current_user\n<i>Dependency</i>\nDecodes JWT, loads User\nfrom database"]
      get_superuser_dep["get_current_active_superuser\n<i>Dependency</i>\nEnforces is_superuser\nprivilege"]
    end

    %% KIND: router
    subgraph routes["API Routes (/api/v1)"]
      login_routes["Login Routes\n<i>Router: /login</i>\naccess-token, test-token,\npassword-recovery, reset"]
      users_routes["Users Routes\n<i>Router: /users</i>\nCRUD, signup, /me,\npassword change"]
      items_routes["Items Routes\n<i>Router: /items</i>\nCRUD for user-owned items"]
      utils_routes["Utils Routes\n<i>Router: /utils</i>\nhealth-check, test-email"]
    end

    %% KIND: data_access
    subgraph data["Data Layer"]
      crud_layer["CRUD Module\n<i>crud.py</i>\ncreate_user, authenticate,\ncreate_item, update_user"]
      models_layer["Models\n<i>SQLModel + Pydantic</i>\nUser, Item, Token,\nrequest/response schemas"]
    end

    %% KIND: service_layer
    subgraph core["Core Services"]
      config_mod["Config\n<i>Pydantic Settings</i>\nEnvironment-based\nconfiguration"]
      db_engine["DB Engine\n<i>SQLAlchemy create_engine</i>\nPostgreSQL connection\nvia psycopg"]
      security_mod["Security\n<i>PyJWT + pwdlib</i>\nJWT creation/validation,\nArgon2/Bcrypt hashing"]
    end

    %% KIND: integration
    email_utils["Email Utils\n<i>utils.py</i>\nSMTP sending, Jinja2 templates,\npassword-reset token generation"]

    %% KIND: service_layer
    alembic_migrations["Alembic Migrations\n<i>Database Migrations</i>\nSchema versioning\nand upgrades"]

  end

  spa -->|"REST API calls"| cors_mw
  cors_mw --> routes
  sentry_integration -.->|"wraps"| routes

  routes --> auth
  login_routes --> security_mod
  login_routes --> email_utils
  users_routes --> crud_layer
  items_routes --> crud_layer

  auth --> session_dep
  get_current_user_dep --> oauth2_bearer
  get_current_user_dep --> security_mod
  get_superuser_dep --> get_current_user_dep

  crud_layer --> models_layer
  crud_layer --> security_mod
  session_dep --> db_engine
  db_engine -->|"SQL [psycopg]"| pg

  email_utils -->|"SMTP/TLS"| smtp_ext
  sentry_integration -->|"HTTPS"| sentry_ext
  config_mod -.->|"configures"| db_engine
  config_mod -.->|"configures"| security_mod
  config_mod -.->|"configures"| email_utils

  alembic_migrations -->|"ALTER/CREATE TABLE"| pg
```

---

## L3: React SPA

```mermaid
flowchart TB
  %% SCOPE: urn:c4:container:spa

  fastapi_backend["FastAPI Backend\n<i>Container</i>"]

  subgraph spa["React SPA"]

    %% KIND: router
    router["TanStack Router\n<i>File-based Routing</i>\nRoute tree generation,\nauth guards in beforeLoad"]

    %% KIND: boundary
    subgraph pages["Pages / Routes"]
      login_page["Login Page\n<i>Route: /login</i>\nOAuth2 form login"]
      signup_page["Signup Page\n<i>Route: /signup</i>\nPublic registration"]
      recover_page["Recover Password\n<i>Route: /recover-password</i>\nSend reset email"]
      reset_page["Reset Password\n<i>Route: /reset-password</i>\nToken-based reset"]
      dashboard_page["Dashboard\n<i>Route: / (protected)</i>\nWelcome view"]
      items_page["Items Page\n<i>Route: /items (protected)</i>\nItem CRUD interface"]
      admin_page["Admin Page\n<i>Route: /admin (superuser)</i>\nUser management"]
      settings_page["Settings Page\n<i>Route: /settings (protected)</i>\nProfile, password, account"]
    end

    %% KIND: boundary
    subgraph components["UI Components"]
      sidebar_comp["Sidebar\n<i>AppSidebar</i>\nNavigation, user menu,\nappearance toggle"]
      items_components["Items Components\n<i>AddItem, EditItem, DeleteItem</i>\nItem CRUD dialogs"]
      admin_components["Admin Components\n<i>AddUser, EditUser, DeleteUser</i>\nUser management dialogs"]
      settings_components["UserSettings Components\n<i>UserInfo, ChangePassword,\nDeleteAccount</i>"]
      ui_lib["UI Primitives\n<i>shadcn/ui + Radix</i>\nButton, Card, Dialog,\nTable, Form, etc."]
    end

    %% KIND: service_layer
    subgraph hooks_layer["Hooks"]
      use_auth["useAuth\n<i>Custom Hook</i>\nLogin, signup, logout,\ncurrent user query"]
      use_toast["useCustomToast\n<i>Custom Hook</i>\nToast notification\nhelpers"]
    end

    %% KIND: integration
    subgraph api_layer["API Client Layer"]
      api_client["OpenAPI SDK\n<i>@hey-api/openapi-ts</i>\nGenerated Axios client:\nItemsService, LoginService,\nUsersService, UtilsService"]
      query_client["React Query Client\n<i>TanStack Query v5</i>\nQuery/mutation caching,\n401/403 error handling"]
    end

    %% KIND: service_layer
    theme_provider["Theme Provider\n<i>Dark/Light Mode</i>\nPersists preference\nin localStorage"]

  end

  router --> pages
  pages --> components
  pages --> hooks_layer

  dashboard_page --> sidebar_comp
  items_page --> items_components
  admin_page --> admin_components
  settings_page --> settings_components

  items_components --> ui_lib
  admin_components --> ui_lib
  settings_components --> ui_lib
  sidebar_comp --> ui_lib

  hooks_layer --> api_layer
  use_auth --> api_client
  use_auth --> query_client

  query_client --> api_client
  api_client -->|"REST API [HTTPS / JSON]"| fastapi_backend
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — covers ALL levels (L1 → L2 → L3)
%% Every subgraph and its children are listed.
%% Nesting is expressed by indentation.
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users              CONTAINS [user, admin]
system_boundary    CONTAINS [fullstack_platform, traefik, spa, fastapi_backend, pg, adminer_ctr]
external           CONTAINS [smtp_ext, sentry_ext, letsencrypt]

%% ── L2 system boundary (containers) ───────────────────────────────
system_boundary    CONTAINS [traefik, spa, fastapi_backend, pg, adminer_ctr]

%% ── L3: FastAPI Backend ───────────────────────────────────────────
fastapi_backend    CONTAINS [middleware, auth, routes, data, core, email_utils, alembic_migrations]
  middleware       CONTAINS [cors_mw, sentry_integration]
  auth             CONTAINS [oauth2_bearer, session_dep, get_current_user_dep, get_superuser_dep]
  routes           CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  data             CONTAINS [crud_layer, models_layer]
  core             CONTAINS [config_mod, db_engine, security_mod]

%% ── L3: React SPA ────────────────────────────────────────────────
spa                CONTAINS [router, pages, components, hooks_layer, api_layer, theme_provider]
  pages            CONTAINS [login_page, signup_page, recover_page, reset_page, dashboard_page, items_page, admin_page, settings_page]
  components       CONTAINS [sidebar_comp, items_components, admin_components, settings_components, ui_lib]
  hooks_layer      CONTAINS [use_auth, use_toast]
  api_layer        CONTAINS [api_client, query_client]
```
