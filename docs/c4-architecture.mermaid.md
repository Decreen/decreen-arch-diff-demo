# C4 Architecture Model — Full Stack FastAPI Project

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams for the Full Stack FastAPI Project.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:platform
graph TB

  subgraph users["Users"]
    user(["User<br/><i>End user who manages items<br/>via the web application</i>"])
    admin_user(["Administrator<br/><i>Superuser with elevated<br/>privileges for user mgmt</i>"])
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    platform["Full Stack FastAPI Project<br/><i>[Software System]</i><br/><i>Web application for item<br/>management with authentication</i>"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Provider<br/><i>[External System]</i><br/><i>Email delivery service</i>"]
    sentry["Sentry<br/><i>[External System]</i><br/><i>Error tracking and<br/>performance monitoring</i>"]
    letsencrypt["Let's Encrypt<br/><i>[External System]</i><br/><i>TLS certificate authority</i>"]
  end

  user -->|"Uses [HTTPS]"| platform
  admin_user -->|"Administers [HTTPS]"| platform
  platform -->|"Sends emails [SMTP/TLS]"| smtp
  platform -->|"Reports errors [HTTPS]"| sentry
  platform -->|"Obtains TLS certs [ACME]"| letsencrypt

  classDef person fill:#08427B,color:#fff,stroke:#073B6F
  classDef system fill:#1168BD,color:#fff,stroke:#0E5CA7
  classDef external fill:#999999,color:#fff,stroke:#8A8A8A

  class user,admin_user person
  class platform system
  class smtp,sentry,letsencrypt external
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:platform
graph TB

  subgraph users["Users"]
    user(["User"])
    admin_user(["Administrator"])
  end

  subgraph platform_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik<br/><i>[Container: Traefik 3.x]</i><br/><i>Reverse proxy, TLS<br/>termination, request routing</i>"]
    spa["Frontend SPA<br/><i>[Container: React 19 / Vite / Nginx]</i><br/><i>Single-page application<br/>serving the web UI</i>"]
    backend_api["Backend API<br/><i>[Container: Python / FastAPI]</i><br/><i>REST API handling auth,<br/>business logic, data access</i>"]
    pg["PostgreSQL<br/><i>[Container: PostgreSQL 18]</i><br/><i>Primary relational database</i>"]
    adminer["Adminer<br/><i>[Container: Adminer]</i><br/><i>Database administration UI</i>"]
  end

  subgraph external["External Systems"]
    smtp["SMTP Provider<br/><i>[External]</i>"]
    sentry["Sentry<br/><i>[External]</i>"]
    letsencrypt["Let's Encrypt<br/><i>[External]</i>"]
  end

  user -->|"Browses [HTTPS]"| traefik
  admin_user -->|"Administers [HTTPS]"| traefik
  traefik -->|"Serves static assets [HTTP]"| spa
  traefik -->|"Proxies API calls [HTTP]"| backend_api
  traefik -->|"Proxies admin UI [HTTP]"| adminer
  spa -->|"Calls REST API [HTTP/JSON]"| backend_api
  backend_api -->|"Reads/writes data [SQL/TCP]"| pg
  adminer -->|"Manages database [SQL/TCP]"| pg
  backend_api -->|"Sends emails [SMTP/TLS]"| smtp
  backend_api -->|"Reports errors [HTTPS]"| sentry
  traefik -->|"Obtains certs [ACME]"| letsencrypt

  classDef person fill:#08427B,color:#fff,stroke:#073B6F
  classDef container fill:#438DD5,color:#fff,stroke:#3C7FC0
  classDef external fill:#999999,color:#fff,stroke:#8A8A8A

  class user,admin_user person
  class traefik,spa,backend_api,pg,adminer container
  class smtp,sentry,letsencrypt external
```

---

## L3: Backend API

```mermaid
%% SCOPE: urn:c4:container:backend_api
graph TB

  subgraph backend_api["Backend API (FastAPI)"]

    %% KIND: boundary
    subgraph mw["Middleware"]
      %% KIND: router
      cors_mw["CORS Middleware<br/><i>[Starlette CORSMiddleware]</i><br/><i>Cross-origin request handling</i>"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies<br/><i>[FastAPI Depends]</i><br/><i>OAuth2 bearer extraction,<br/>JWT validation, user resolution</i>"]
    end

    %% KIND: boundary
    subgraph routes["Route Handlers"]
      %% KIND: router
      login_routes["Login Routes<br/><i>[APIRouter]</i><br/><i>/login/* — token issuance,<br/>password recovery and reset</i>"]
      %% KIND: router
      user_routes["User Routes<br/><i>[APIRouter]</i><br/><i>/users/* — registration,<br/>profile, user CRUD</i>"]
      %% KIND: router
      item_routes["Item Routes<br/><i>[APIRouter]</i><br/><i>/items/* — item CRUD</i>"]
      %% KIND: router
      util_routes["Utility Routes<br/><i>[APIRouter]</i><br/><i>/utils/* — health check,<br/>test email</i>"]
    end

    %% KIND: boundary
    subgraph services["Services"]
      %% KIND: service_layer
      security_mod["Security Module<br/><i>[PyJWT / pwdlib]</i><br/><i>JWT creation and validation,<br/>password hashing (Argon2/Bcrypt)</i>"]
      %% KIND: integration
      email_utils["Email Utilities<br/><i>[emails / Jinja2]</i><br/><i>Template rendering and<br/>transactional email sending</i>"]
      %% KIND: service_layer
      config_mod["Configuration<br/><i>[Pydantic Settings]</i><br/><i>Environment-based application<br/>configuration</i>"]
    end

    %% KIND: boundary
    subgraph data_access["Data Access"]
      %% KIND: data_access
      crud_layer["CRUD Layer<br/><i>[Python / SQLModel]</i><br/><i>Data access functions for<br/>User and Item entities</i>"]
      %% KIND: data_access
      models_layer["Models and Schemas<br/><i>[SQLModel / Pydantic]</i><br/><i>ORM entities, request/response<br/>schemas, validation</i>"]
      %% KIND: storage
      db_mod["Database Module<br/><i>[SQLAlchemy Engine]</i><br/><i>Connection pool, session<br/>management, DB initialization</i>"]
    end

  end

  pg["PostgreSQL<br/><i>[Container]</i>"]
  smtp["SMTP Provider<br/><i>[External]</i>"]

  auth_deps --> security_mod
  auth_deps --> db_mod
  login_routes --> security_mod
  login_routes --> crud_layer
  login_routes --> email_utils
  user_routes --> crud_layer
  user_routes --> security_mod
  item_routes --> crud_layer
  util_routes --> email_utils
  crud_layer --> models_layer
  crud_layer --> security_mod
  models_layer --> db_mod
  db_mod -->|"SQL / TCP"| pg
  email_utils -->|"SMTP / TLS"| smtp
  email_utils --> config_mod
  security_mod --> config_mod

  classDef component fill:#85BBF0,color:#000,stroke:#78A8D8
  classDef external fill:#999999,color:#fff,stroke:#8A8A8A

  class cors_mw,auth_deps component
  class login_routes,user_routes,item_routes,util_routes component
  class security_mod,email_utils,config_mod component
  class crud_layer,models_layer,db_mod component
  class pg,smtp external
```

---

## L3: Frontend SPA

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

  subgraph spa["Frontend SPA (React)"]

    %% KIND: boundary
    subgraph pages["Pages"]
      %% KIND: router
      auth_pages["Auth Pages<br/><i>[React Components]</i><br/><i>Login, signup, password<br/>recovery and reset</i>"]
      %% KIND: service_layer
      dashboard_page["Dashboard<br/><i>[React Component]</i><br/><i>Welcome landing page<br/>for logged-in users</i>"]
      %% KIND: service_layer
      items_page["Items Management<br/><i>[React Component]</i><br/><i>CRUD data table<br/>for item entities</i>"]
      %% KIND: service_layer
      settings_page["User Settings<br/><i>[React Component]</i><br/><i>Profile, password,<br/>account management</i>"]
      %% KIND: service_layer
      admin_page["Admin Panel<br/><i>[React Component]</i><br/><i>User management<br/>(superuser only)</i>"]
    end

    %% KIND: boundary
    subgraph services_layer["Services"]
      %% KIND: router
      routing["TanStack Router<br/><i>[@tanstack/router]</i><br/><i>File-based routing<br/>with auth guards</i>"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>[@hey-api/openapi-ts]</i><br/><i>Generated typed HTTP<br/>client via Axios</i>"]
      %% KIND: data_access
      query_layer["TanStack Query<br/><i>[@tanstack/react-query]</i><br/><i>Server state management,<br/>caching, mutations</i>"]
      %% KIND: service_layer
      auth_hook["Auth Hook<br/><i>[React Hook]</i><br/><i>Token storage,<br/>login/logout logic</i>"]
    end

    %% KIND: boundary
    subgraph ui_layer["UI Layer"]
      %% KIND: service_layer
      ui_lib["UI Components<br/><i>[Shadcn / Radix UI]</i><br/><i>Reusable design-system<br/>components</i>"]
      %% KIND: service_layer
      theme_provider["Theme Provider<br/><i>[next-themes]</i><br/><i>Dark and light mode</i>"]
    end

  end

  backend_api["Backend API<br/><i>[Container]</i>"]

  routing --> auth_pages
  routing --> dashboard_page
  routing --> items_page
  routing --> settings_page
  routing --> admin_page
  auth_pages --> auth_hook
  auth_pages --> api_client
  dashboard_page --> query_layer
  items_page --> query_layer
  settings_page --> query_layer
  admin_page --> query_layer
  auth_hook --> api_client
  query_layer --> api_client
  api_client -->|"HTTP / JSON"| backend_api
  auth_pages --> ui_lib
  dashboard_page --> ui_lib
  items_page --> ui_lib
  settings_page --> ui_lib
  admin_page --> ui_lib

  classDef component fill:#85BBF0,color:#000,stroke:#78A8D8
  classDef external fill:#438DD5,color:#fff,stroke:#3C7FC0

  class auth_pages,dashboard_page,items_page,settings_page,admin_page component
  class routing,api_client,query_layer,auth_hook component
  class ui_lib,theme_provider component
  class backend_api external
```

---

## Containment Map

```text
%% ── CONTAINMENT MAP ─────────────────────────────────────────────────────────

%% ── L1 top-level groups ─────────────────────────────────────────────────────
users              CONTAINS [user, admin_user]
platform_boundary  CONTAINS [traefik, spa, backend_api, pg, adminer]
external           CONTAINS [smtp, sentry, letsencrypt]

%% ── L2→L3 internal containment (Backend API) ───────────────────────────────
backend_api        CONTAINS [mw, routes, services, data_access]
  mw               CONTAINS [cors_mw, auth_deps]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes]
  services         CONTAINS [security_mod, email_utils, config_mod]
  data_access      CONTAINS [crud_layer, models_layer, db_mod]

%% ── L2→L3 internal containment (Frontend SPA) ──────────────────────────────
spa                CONTAINS [pages, services_layer, ui_layer]
  pages            CONTAINS [auth_pages, dashboard_page, items_page, settings_page, admin_page]
  services_layer   CONTAINS [routing, api_client, query_layer, auth_hook]
  ui_layer         CONTAINS [ui_lib, theme_provider]
```
