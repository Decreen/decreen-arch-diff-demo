# C4 Architecture Model — Full Stack FastAPI Template

> Auto-generated C4 model covering L1 (System Context), L2 (Container),
> and L3 (Component) diagrams with a complete containment map.

---

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
graph TB

  subgraph users ["Users"]
    end_user["End User<br/><i>Regular application user</i>"]
    admin_user["Admin User<br/><i>Superuser / operator</i>"]
  end

  subgraph system_boundary ["Full Stack FastAPI Platform"]
    fullstack_system["Full Stack FastAPI<br/><i>Web application for managing<br/>users and items</i>"]
  end

  subgraph external ["External Services"]
    smtp["SMTP Server<br/><i>Email delivery</i>"]
    sentry["Sentry<br/><i>Error tracking &amp; monitoring</i>"]
  end

  end_user -- "Manages items,<br/>updates profile" --> fullstack_system
  admin_user -- "Manages users &amp; items,<br/>system administration" --> fullstack_system
  fullstack_system -- "Sends transactional<br/>emails via SMTP" --> smtp
  fullstack_system -- "Reports errors<br/>and traces" --> sentry
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
graph TB

  subgraph users ["Users"]
    end_user["End User"]
    admin_user["Admin User"]
  end

  subgraph system_boundary ["Full Stack FastAPI Platform"]
    traefik["Traefik<br/><i>Reverse proxy / TLS termination<br/>Docker: traefik:3.6</i>"]
    spa["React SPA<br/><i>Vite + React 19 + TanStack Router<br/>Served by Nginx</i>"]
    fastapi_backend["FastAPI Backend<br/><i>Python FastAPI + Uvicorn<br/>REST API on /api/v1</i>"]
    pg["PostgreSQL<br/><i>postgres:18<br/>Relational database</i>"]
  end

  subgraph external ["External Services"]
    smtp["SMTP Server<br/><i>Email delivery</i>"]
    sentry["Sentry<br/><i>Error tracking</i>"]
  end

  end_user -- "HTTPS" --> traefik
  admin_user -- "HTTPS" --> traefik

  traefik -- "dashboard.*" --> spa
  traefik -- "api.*" --> fastapi_backend

  spa -- "REST /api/v1/*<br/>JSON + JWT" --> fastapi_backend
  fastapi_backend -- "SQL via psycopg<br/>SQLModel ORM" --> pg
  fastapi_backend -- "SMTP" --> smtp
  fastapi_backend -- "Sentry SDK<br/>HTTPS" --> sentry
```

---

## L3: FastAPI Backend — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:fastapi-backend
graph TB

  subgraph fastapi_backend ["FastAPI Backend"]

    subgraph middleware ["Middleware"]
      %% KIND: boundary
      cors_mw["CORS Middleware<br/><i>Starlette CORSMiddleware</i>"]
    end

    subgraph auth ["Auth & Dependencies"]
      %% KIND: service_layer
      auth_deps["Auth Dependencies<br/><i>OAuth2 bearer, JWT decode,<br/>current-user injection</i>"]
    end

    subgraph routes ["API Routes"]
      %% KIND: router
      login_routes["Login Routes<br/><i>/login/*, /password-recovery/*,<br/>/reset-password/</i>"]
      %% KIND: router
      user_routes["User Routes<br/><i>/users/* CRUD,<br/>signup, profile</i>"]
      %% KIND: router
      item_routes["Item Routes<br/><i>/items/* CRUD,<br/>owner-scoped access</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>/utils/health-check/,<br/>test-email</i>"]
    end

    subgraph services ["Core Services"]
      %% KIND: service_layer
      security_mod["Security Module<br/><i>JWT creation, Argon2/bcrypt<br/>password hashing</i>"]
      %% KIND: service_layer
      email_utils["Email Utils<br/><i>Jinja2 templates,<br/>SMTP send, token gen</i>"]
      %% KIND: service_layer
      config_mod["Config Module<br/><i>Pydantic Settings,<br/>env-based configuration</i>"]
    end

    subgraph data_access ["Data Access"]
      %% KIND: data_access
      crud_layer["CRUD Layer<br/><i>create/read/update/delete<br/>User &amp; Item operations</i>"]
      %% KIND: data_access
      models_layer["Models Layer<br/><i>SQLModel table models,<br/>Pydantic schemas</i>"]
      %% KIND: storage
      db_engine["DB Engine<br/><i>SQLAlchemy engine,<br/>session factory</i>"]
    end

  end

  pg["PostgreSQL"]
  smtp["SMTP Server"]
  sentry["Sentry"]
  spa["React SPA"]

  spa -- "HTTP requests" --> cors_mw
  cors_mw --> auth_deps
  auth_deps --> login_routes
  auth_deps --> user_routes
  auth_deps --> item_routes
  auth_deps --> utils_routes

  login_routes --> security_mod
  login_routes --> email_utils
  login_routes --> crud_layer

  user_routes --> crud_layer
  user_routes --> security_mod
  user_routes --> email_utils

  item_routes --> crud_layer

  utils_routes --> email_utils

  crud_layer --> models_layer
  crud_layer --> db_engine
  models_layer --> db_engine

  db_engine -- "SQL via psycopg" --> pg
  email_utils -- "SMTP" --> smtp
  config_mod -. "reads Sentry DSN" .-> sentry
```

---

## L3: React SPA — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:spa
graph TB

  subgraph spa ["React SPA"]

    subgraph routing ["Routing"]
      %% KIND: router
      tanstack_router["TanStack Router<br/><i>File-based routing,<br/>route guards</i>"]
      %% KIND: router
      root_route["Root Route<br/><i>Error boundary,<br/>devtools</i>"]
      %% KIND: router
      layout_route["Layout Route<br/><i>Auth guard, sidebar,<br/>main content area</i>"]
      %% KIND: router
      public_routes["Public Routes<br/><i>/login, /signup,<br/>/recover-password, /reset-password</i>"]
    end

    subgraph state ["State & Data"]
      %% KIND: service_layer
      query_client["React Query Client<br/><i>QueryClient, caches,<br/>mutations, error handling</i>"]
      %% KIND: service_layer
      auth_hook["useAuth Hook<br/><i>Login, logout, signup,<br/>current user query</i>"]
    end

    subgraph api_layer ["API Client"]
      %% KIND: integration
      api_client["OpenAPI Client<br/><i>Auto-generated SDK:<br/>ItemsService, UsersService,<br/>LoginService, UtilsService</i>"]
    end

    subgraph features ["Feature Components"]
      %% KIND: boundary
      admin_components["Admin Components<br/><i>AddUser, EditUser, DeleteUser,<br/>user table &amp; columns</i>"]
      %% KIND: boundary
      items_components["Items Components<br/><i>AddItem, EditItem, DeleteItem,<br/>item table &amp; columns</i>"]
      %% KIND: boundary
      settings_components["UserSettings Components<br/><i>ChangePassword, UserInformation,<br/>DeleteAccount</i>"]
    end

    subgraph ui ["UI Foundation"]
      %% KIND: boundary
      sidebar_nav["Sidebar Navigation<br/><i>AppSidebar, Main nav,<br/>User menu</i>"]
      %% KIND: boundary
      ui_primitives["UI Primitives<br/><i>shadcn/ui: Button, Dialog,<br/>Table, Form, etc.</i>"]
      %% KIND: service_layer
      theme_provider["Theme Provider<br/><i>Dark/light mode,<br/>next-themes</i>"]
    end

  end

  fastapi_backend["FastAPI Backend"]

  tanstack_router --> root_route
  tanstack_router --> layout_route
  tanstack_router --> public_routes

  layout_route --> sidebar_nav
  layout_route --> admin_components
  layout_route --> items_components
  layout_route --> settings_components

  auth_hook --> api_client
  admin_components --> query_client
  items_components --> query_client
  settings_components --> query_client
  query_client --> api_client

  admin_components --> ui_primitives
  items_components --> ui_primitives
  settings_components --> ui_primitives
  sidebar_nav --> ui_primitives

  api_client -- "REST /api/v1/*<br/>JSON + JWT" --> fastapi_backend
```

---

## L3: PostgreSQL — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:pg
graph TB

  subgraph pg ["PostgreSQL"]

    subgraph schemas ["Database Schema"]
      %% KIND: storage
      user_table["user Table<br/><i>id (UUID PK), email, full_name,<br/>hashed_password, is_active,<br/>is_superuser, created_at</i>"]
      %% KIND: storage
      item_table["item Table<br/><i>id (UUID PK), title, description,<br/>owner_id (FK → user), created_at</i>"]
    end

    subgraph migrations ["Schema Management"]
      %% KIND: data_access
      alembic["Alembic Migrations<br/><i>Version-controlled schema<br/>evolution</i>"]
    end

  end

  fastapi_backend["FastAPI Backend"]

  fastapi_backend -- "SQL via psycopg" --> user_table
  fastapi_backend -- "SQL via psycopg" --> item_table
  item_table -- "FK owner_id<br/>CASCADE DELETE" --> user_table
  alembic -. "manages schema" .-> user_table
  alembic -. "manages schema" .-> item_table
```

---

## L3: Traefik — Component Diagram

```mermaid
%% SCOPE: urn:c4:container:traefik
graph TB

  subgraph traefik ["Traefik Reverse Proxy"]

    subgraph entrypoints ["Entrypoints"]
      %% KIND: router
      http_ep["HTTP Entrypoint<br/><i>Port 80, redirects to HTTPS</i>"]
      %% KIND: router
      https_ep["HTTPS Entrypoint<br/><i>Port 443, TLS termination</i>"]
    end

    subgraph routers ["Routers"]
      %% KIND: router
      frontend_router["Frontend Router<br/><i>Host: dashboard.*</i>"]
      %% KIND: router
      backend_router["Backend Router<br/><i>Host: api.*</i>"]
      %% KIND: router
      adminer_router["Adminer Router<br/><i>Host: adminer.*</i>"]
    end

    subgraph tls ["TLS"]
      %% KIND: service_layer
      le_resolver["Let's Encrypt Resolver<br/><i>ACME TLS challenge,<br/>auto certificate renewal</i>"]
    end

    subgraph mw ["Middleware"]
      %% KIND: boundary
      https_redirect["HTTPS Redirect<br/><i>HTTP → HTTPS permanent redirect</i>"]
    end

  end

  spa["React SPA"]
  fastapi_backend["FastAPI Backend"]
  adminer["Adminer"]

  http_ep --> https_redirect
  https_redirect --> https_ep
  https_ep --> le_resolver
  le_resolver --> frontend_router
  le_resolver --> backend_router
  le_resolver --> adminer_router
  frontend_router --> spa
  backend_router --> fastapi_backend
  adminer_router --> adminer
```

---

## Containment Map

```text
%% ── L1 top-level groups ──────────────────────────────────────────────
users              CONTAINS [end_user, admin_user]
system_boundary    CONTAINS [traefik, spa, fastapi_backend, pg]
external           CONTAINS [smtp, sentry]

%% ── L2 → L3 internal containment ────────────────────────────────────

%% FastAPI Backend (L3)
fastapi_backend    CONTAINS [middleware, auth, routes, services, data_access]
  middleware       CONTAINS [cors_mw]
  auth             CONTAINS [auth_deps]
  routes           CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services         CONTAINS [security_mod, email_utils, config_mod]
  data_access      CONTAINS [crud_layer, models_layer, db_engine]

%% React SPA (L3)
spa                CONTAINS [routing, state, api_layer, features, ui]
  routing          CONTAINS [tanstack_router, root_route, layout_route, public_routes]
  state            CONTAINS [query_client, auth_hook]
  api_layer        CONTAINS [api_client]
  features         CONTAINS [admin_components, items_components, settings_components]
  ui               CONTAINS [sidebar_nav, ui_primitives, theme_provider]

%% PostgreSQL (L3)
pg                 CONTAINS [schemas, migrations]
  schemas          CONTAINS [user_table, item_table]
  migrations       CONTAINS [alembic]

%% Traefik (L3)
traefik            CONTAINS [entrypoints, routers, tls, mw]
  entrypoints      CONTAINS [http_ep, https_ep]
  routers          CONTAINS [frontend_router, backend_router, adminer_router]
  tls              CONTAINS [le_resolver]
  mw               CONTAINS [https_redirect]
```
