# C4 Architecture Model — Full Stack FastAPI Project

---

## L1: System Context

```mermaid
graph TD
  %% SCOPE: urn:c4:system:fullstack_fastapi

  subgraph users["Users"]
    user["👤 User\n[Person]\nUses the web application to\nmanage items and profile"]
    admin_user["👤 Admin\n[Person]\nManages users and\nsystem configuration"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    fullstack_system["Full Stack FastAPI Project\n[Software System]\nWeb application for user and item\nmanagement with auth and email"]
  end

  subgraph external["External Services"]
    smtp["SMTP Service\n[External System]\nEmail delivery provider"]
    sentry["Sentry\n[External System]\nError monitoring and tracing"]
    letsencrypt["Let's Encrypt\n[External System]\nAutomated TLS certificates"]
  end

  user -->|"Uses web app\n[HTTPS]"| fullstack_system
  admin_user -->|"Administers system\n[HTTPS]"| fullstack_system
  fullstack_system -->|"Sends emails\n[SMTP/TLS]"| smtp
  fullstack_system -->|"Reports errors\n[HTTPS]"| sentry
  fullstack_system -->|"Obtains TLS certs\n[ACME]"| letsencrypt
```

---

## L2: Container

```mermaid
graph TD
  %% SCOPE: urn:c4:system:fullstack_fastapi

  subgraph users["Users"]
    user["👤 User\n[Person]"]
    admin_user["👤 Admin\n[Person]"]
  end

  subgraph system_boundary["Full Stack FastAPI Project"]
    traefik["Traefik\n[Container: Traefik 3.6]\nReverse proxy, TLS termination,\nrequest routing"]
    spa["React SPA\n[Container: React 19 / Nginx]\nSingle-page application\nserved by Nginx"]
    fastapi["FastAPI Backend\n[Container: Python / FastAPI]\nREST API, business logic,\nauthentication"]
    pg["PostgreSQL\n[Container: PostgreSQL 18]\nPrimary relational data store"]
    adminer["Adminer\n[Container: Adminer]\nDatabase administration UI"]
  end

  subgraph external["External Services"]
    smtp["SMTP Service\n[External System]\nEmail delivery"]
    sentry["Sentry\n[External System]\nError monitoring"]
    letsencrypt["Let's Encrypt\n[External System]\nTLS certificates"]
  end

  user -->|"HTTPS"| traefik
  admin_user -->|"HTTPS"| traefik
  traefik -->|"Serves SPA\n[HTTP :80]"| spa
  traefik -->|"Routes /api\n[HTTP :8000]"| fastapi
  traefik -->|"Routes /adminer\n[HTTP :8080]"| adminer
  spa -.->|"API calls\n[JSON/HTTPS via Traefik]"| fastapi
  fastapi -->|"Reads/writes\n[SQL via psycopg]"| pg
  adminer -->|"Manages data\n[SQL]"| pg
  fastapi -->|"Sends emails\n[SMTP]"| smtp
  fastapi -.->|"Error reports\n[HTTPS]"| sentry
  traefik -->|"Certificate requests\n[ACME]"| letsencrypt
```

---

## L3: FastAPI Backend

```mermaid
graph TD
  %% SCOPE: urn:c4:container:fastapi

  spa["React SPA"]
  pg["PostgreSQL"]
  smtp["SMTP Service"]
  sentry["Sentry"]

  subgraph fastapi["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware"]
      cors_mw["CORS Middleware\n[Component: Starlette]\nEnforces cross-origin policy\nfrom BACKEND_CORS_ORIGINS"]
    end

    %% KIND: router
    subgraph routes["API Routes"]
      api_router["API Router\n[Component: FastAPI APIRouter]\nAggregates sub-routers\nunder /api/v1"]
      %% KIND: router
      login_routes["Login Routes\n[Component: FastAPI Router]\nOAuth2 token, password\nrecovery and reset"]
      %% KIND: router
      users_routes["Users Routes\n[Component: FastAPI Router]\nUser CRUD, signup,\nprofile management"]
      %% KIND: router
      items_routes["Items Routes\n[Component: FastAPI Router]\nItem CRUD operations"]
      %% KIND: router
      utils_routes["Utils Routes\n[Component: FastAPI Router]\nHealth check, test email"]
      %% KIND: router
      private_routes["Private Routes\n[Component: FastAPI Router]\nLocal-only dev endpoints"]
    end

    %% KIND: service_layer
    subgraph deps["Dependencies"]
      auth_dep["Auth Dependency\n[Component: FastAPI Depends]\nJWT validation via OAuth2\nreturns current user"]
      db_dep["DB Session\n[Component: FastAPI Depends]\nYields SQLModel Session\nfrom engine"]
      superuser_dep["Superuser Guard\n[Component: FastAPI Depends]\nVerifies is_superuser flag"]
    end

    %% KIND: data_access
    subgraph data_layer["Data Access"]
      crud["CRUD Operations\n[Component: Python Module]\nCreate/read/update/delete\nfor User and Item"]
      models["SQLModel Models\n[Component: SQLModel]\nUser, Item tables and\nPydantic request/response schemas"]
    end

    %% KIND: service_layer
    subgraph core["Core"]
      config["Settings\n[Component: Pydantic Settings]\nAll env-based configuration\nvalidated at startup"]
      db_engine["DB Engine\n[Component: SQLModel create_engine]\nPostgreSQL connection pool"]
      security["Security\n[Component: Python Module]\nJWT token creation,\nArgon2/Bcrypt password hashing"]
    end

    %% KIND: integration
    email_utils["Email Utilities\n[Component: emails library]\nHTML email rendering\nand SMTP dispatch"]

    %% KIND: data_access
    alembic["Alembic Migrations\n[Component: Alembic]\nDatabase schema versioning\nand migration runner"]
  end

  spa -->|"HTTP/JSON"| cors_mw
  cors_mw --> api_router
  api_router --> login_routes
  api_router --> users_routes
  api_router --> items_routes
  api_router --> utils_routes
  api_router --> private_routes

  login_routes --> auth_dep
  login_routes --> security
  login_routes --> email_utils
  login_routes --> crud
  users_routes --> auth_dep
  users_routes --> superuser_dep
  users_routes --> email_utils
  users_routes --> crud
  items_routes --> auth_dep
  items_routes --> crud
  utils_routes --> superuser_dep
  utils_routes --> email_utils
  private_routes --> db_dep

  auth_dep --> security
  auth_dep --> db_dep
  superuser_dep --> auth_dep
  db_dep --> db_engine

  crud --> models
  crud --> db_dep

  db_engine --> pg
  config --> db_engine
  alembic --> pg
  email_utils --> smtp
  fastapi -.->|"Sentry SDK init"| sentry
```

---

## L3: React SPA

```mermaid
graph TD
  %% SCOPE: urn:c4:container:spa

  fastapi["FastAPI Backend"]

  subgraph spa["React SPA"]

    %% KIND: router
    subgraph routing["Routing"]
      router["TanStack Router\n[Component: @tanstack/react-router]\nFile-based routing with\nauto code-splitting"]
      route_tree["Route Tree\n[Component: Generated]\nAuto-generated route\ndefinitions from /routes"]
    end

    %% KIND: integration
    subgraph client_layer["API Client Layer"]
      api_client["OpenAPI Client\n[Component: @hey-api/openapi-ts]\nGenerated SDK with typed\nservice methods"]
      axios_core["Axios Core\n[Component: Axios]\nHTTP request execution,\ntoken injection, error handling"]
    end

    %% KIND: service_layer
    subgraph hooks_layer["Hooks"]
      auth_hook["useAuth\n[Component: React Hook]\nCurrent user query,\nlogin/logout actions"]
      toast_hook["useCustomToast\n[Component: React Hook]\nStandardized success/error\nnotification helpers"]
    end

    %% KIND: service_layer
    subgraph state_layer["State Management"]
      query_client["React Query\n[Component: TanStack Query]\nServer state caching,\nrefetching, mutations"]
      theme_provider["Theme Provider\n[Component: React Context]\nDark/light mode toggle\nvia localStorage"]
    end

    %% KIND: boundary
    subgraph pages_public["Public Pages"]
      login_page["Login Page\n[Component: React Route]\nOAuth2 password login form"]
      signup_page["Signup Page\n[Component: React Route]\nNew user registration"]
      recover_page["Recover Password\n[Component: React Route]\nEmail-based password recovery"]
      reset_page["Reset Password\n[Component: React Route]\nToken-based password reset"]
    end

    %% KIND: boundary
    subgraph pages_protected["Protected Pages"]
      dashboard_page["Dashboard\n[Component: React Route]\nWelcome / overview page"]
      items_page["Items Page\n[Component: React Route]\nItem CRUD with data table"]
      admin_page["Admin Page\n[Component: React Route]\nUser management (superuser)"]
      settings_page["Settings Page\n[Component: React Route]\nProfile, password, account"]
    end

    %% KIND: boundary
    subgraph ui_components["UI Components"]
      common_components["Common\n[Component: React]\nAuthLayout, Footer,\nLogo, NotFound, DataTable"]
      sidebar_components["Sidebar\n[Component: React]\nAppSidebar, navigation,\nuser menu"]
      items_components["Items Components\n[Component: React]\nAddItem, EditItem,\nDeleteItem, columns"]
      admin_components["Admin Components\n[Component: React]\nAddUser, EditUser,\nDeleteUser, columns"]
      settings_components["Settings Components\n[Component: React]\nUserInformation, ChangePassword,\nDeleteAccount"]
      %% KIND: boundary
      ui_primitives["UI Primitives\n[Component: shadcn/Radix]\nButton, Dialog, Input, Select,\nTabs, Tooltip, etc."]
    end
  end

  router --> route_tree
  router --> pages_public
  router --> pages_protected

  pages_public --> api_client
  pages_public --> common_components
  pages_protected --> api_client
  pages_protected --> auth_hook
  pages_protected --> common_components
  pages_protected --> sidebar_components

  items_page --> items_components
  admin_page --> admin_components
  settings_page --> settings_components

  auth_hook --> query_client
  auth_hook --> api_client

  items_components --> ui_primitives
  admin_components --> ui_primitives
  settings_components --> ui_primitives
  common_components --> ui_primitives
  sidebar_components --> ui_primitives

  api_client --> axios_core
  query_client --> api_client
  axios_core -->|"HTTP/JSON"| fastapi
  toast_hook --> ui_primitives
```

---

## Containment Map

```text
%% ── CONTAINMENT MAP ──────────────────────────────────────────────────────────
%%
%% Every subgraph from every diagram is listed with its CONTAINS children.
%% Indentation expresses nesting depth for the parser's parent→child tree.
%%
%% ── L1 top-level groups ─────────────────────────────────────────────────────

users              CONTAINS [user, admin_user]
system_boundary    CONTAINS [fullstack_system, traefik, spa, fastapi, pg, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2 containers (same system_boundary, expanded) ──────────────────────────
%% system_boundary entries above include all L2 containers.

%% ── L3: FastAPI Backend ─────────────────────────────────────────────────────

fastapi            CONTAINS [middleware, routes, deps, data_layer, core, email_utils, alembic]
  middleware       CONTAINS [cors_mw]
  routes           CONTAINS [api_router, login_routes, users_routes, items_routes, utils_routes, private_routes]
  deps             CONTAINS [auth_dep, db_dep, superuser_dep]
  data_layer       CONTAINS [crud, models]
  core             CONTAINS [config, db_engine, security]

%% ── L3: React SPA ──────────────────────────────────────────────────────────

spa                CONTAINS [routing, client_layer, hooks_layer, state_layer, pages_public, pages_protected, ui_components]
  routing          CONTAINS [router, route_tree]
  client_layer     CONTAINS [api_client, axios_core]
  hooks_layer      CONTAINS [auth_hook, toast_hook]
  state_layer      CONTAINS [query_client, theme_provider]
  pages_public     CONTAINS [login_page, signup_page, recover_page, reset_page]
  pages_protected  CONTAINS [dashboard_page, items_page, admin_page, settings_page]
  ui_components    CONTAINS [common_components, sidebar_components, items_components, admin_components, settings_components, ui_primitives]
```
