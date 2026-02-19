# C4 Architecture Model — Full Stack FastAPI Template

---

## L1: System Context

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack_fastapi

  subgraph users["Users"]
    end_user["End User<br/><i>Regular application user</i>"]
    admin_user["Admin User<br/><i>Superuser / administrator</i>"]
  end

  subgraph fullstack_fastapi_boundary["Full Stack FastAPI Platform"]
    fullstack_fastapi["Full Stack FastAPI<br/><i>Web application for managing<br/>users and items with JWT auth</i>"]
  end

  subgraph external["External Systems"]
    smtp_server["SMTP Server<br/><i>Email delivery service</i>"]
    sentry["Sentry<br/><i>Error monitoring &amp; tracing</i>"]
  end

  end_user -->|"Uses web dashboard<br/>[HTTPS]"| fullstack_fastapi
  admin_user -->|"Manages users &amp; settings<br/>[HTTPS]"| fullstack_fastapi
  fullstack_fastapi -->|"Sends transactional emails<br/>[SMTP/TLS]"| smtp_server
  fullstack_fastapi -->|"Reports errors &amp; traces<br/>[HTTPS]"| sentry
```

---

## L2: Container

```mermaid
graph TB
  %% SCOPE: urn:c4:system:fullstack_fastapi

  end_user["End User"]
  admin_user["Admin User"]

  subgraph fullstack_fastapi_boundary["Full Stack FastAPI Platform"]
    traefik["Traefik<br/><i>Reverse proxy &amp; TLS termination</i>"]
    spa["React SPA<br/><i>Vite + TanStack Router + React Query<br/>Served via Nginx</i>"]
    fastapi_backend["FastAPI Backend<br/><i>Python REST API<br/>SQLModel ORM, JWT auth</i>"]
    pg["PostgreSQL<br/><i>Relational database<br/>Users &amp; Items tables</i>"]
    adminer["Adminer<br/><i>Database admin UI</i>"]
  end

  subgraph external["External Systems"]
    smtp_server["SMTP Server<br/><i>Email delivery</i>"]
    sentry["Sentry<br/><i>Error monitoring</i>"]
  end

  end_user -->|"Browses dashboard<br/>[HTTPS]"| traefik
  admin_user -->|"Manages users<br/>[HTTPS]"| traefik
  admin_user -->|"Inspects database<br/>[HTTPS]"| traefik

  traefik -->|"Routes dashboard.*<br/>[HTTP]"| spa
  traefik -->|"Routes api.*<br/>[HTTP]"| fastapi_backend
  traefik -->|"Routes adminer.*<br/>[HTTP]"| adminer

  spa -->|"Calls REST API<br/>[HTTP/JSON]"| fastapi_backend
  fastapi_backend -->|"Reads/writes data<br/>[psycopg / TCP 5432]"| pg
  fastapi_backend -->|"Sends emails<br/>[SMTP/TLS]"| smtp_server
  fastapi_backend -->|"Reports errors<br/>[HTTPS]"| sentry
  adminer -->|"Manages schema &amp; data<br/>[TCP 5432]"| pg
```

---

## L3: FastAPI Backend

```mermaid
graph TB
  %% SCOPE: urn:c4:container:fastapi_backend

  spa["React SPA"]
  pg["PostgreSQL"]
  smtp_server["SMTP Server"]
  sentry["Sentry"]

  subgraph fastapi_backend["FastAPI Backend"]

    %% KIND: boundary
    subgraph middleware["Middleware"]
      %% KIND: router
      cors_mw["CORS Middleware<br/><i>Handles cross-origin requests</i>"]
    end

    %% KIND: router
    subgraph api_router["API Router (/api/v1)"]
      %% KIND: router
      login_routes["Login Routes<br/><i>/login, /password-recovery,<br/>/reset-password</i>"]
      %% KIND: router
      user_routes["User Routes<br/><i>/users CRUD, /signup,<br/>/me, /me/password</i>"]
      %% KIND: router
      item_routes["Item Routes<br/><i>/items CRUD with<br/>owner-scoped access</i>"]
      %% KIND: router
      utils_routes["Utils Routes<br/><i>/health-check,<br/>/test-email</i>"]
    end

    %% KIND: service_layer
    subgraph auth_deps["Auth Dependencies"]
      %% KIND: service_layer
      oauth2_scheme["OAuth2 Password Bearer<br/><i>Token extraction</i>"]
      %% KIND: service_layer
      get_current_user["get_current_user<br/><i>JWT validation &amp; user lookup</i>"]
      %% KIND: service_layer
      get_superuser["get_current_active_superuser<br/><i>Admin privilege check</i>"]
      %% KIND: service_layer
      session_dep["SessionDep<br/><i>DB session injection</i>"]
    end

    %% KIND: data_access
    crud_layer["CRUD Layer<br/><i>create/read/update/delete<br/>User &amp; Item operations</i>"]

    %% KIND: storage
    models_layer["SQLModel Models<br/><i>User, Item tables<br/>Pydantic schemas</i>"]

    %% KIND: service_layer
    security_mod["Security Module<br/><i>JWT creation, password<br/>hashing (Argon2/Bcrypt)</i>"]

    %% KIND: service_layer
    core_config["Core Config<br/><i>Pydantic Settings<br/>env-based configuration</i>"]

    %% KIND: integration
    email_utils["Email Utilities<br/><i>Jinja2 templates, SMTP<br/>send &amp; token generation</i>"]

    %% KIND: data_access
    db_engine["Database Engine<br/><i>SQLAlchemy engine<br/>&amp; session factory</i>"]

    %% KIND: data_access
    alembic_mig["Alembic Migrations<br/><i>Schema versioning<br/>&amp; migration scripts</i>"]
  end

  spa -->|"HTTP requests"| cors_mw
  cors_mw --> api_router
  login_routes --> auth_deps
  user_routes --> auth_deps
  item_routes --> auth_deps
  utils_routes --> auth_deps

  login_routes --> crud_layer
  user_routes --> crud_layer
  item_routes --> models_layer
  login_routes --> email_utils

  auth_deps --> security_mod
  auth_deps --> db_engine
  crud_layer --> security_mod
  crud_layer --> models_layer
  crud_layer --> db_engine

  security_mod --> core_config
  email_utils --> core_config
  email_utils --> smtp_server
  db_engine --> core_config
  db_engine --> pg
  alembic_mig --> pg

  fastapi_backend -.->|"Sentry SDK"| sentry
```

---

## L3: React SPA

```mermaid
graph TB
  %% SCOPE: urn:c4:container:spa

  end_user["End User"]
  admin_user["Admin User"]
  fastapi_backend["FastAPI Backend"]

  subgraph spa["React SPA"]

    %% KIND: boundary
    subgraph routing["Routing Layer"]
      %% KIND: router
      tanstack_router["TanStack Router<br/><i>File-based routing<br/>with auth guards</i>"]
      %% KIND: router
      root_route["Root Route<br/><i>Error boundary &amp;<br/>devtools</i>"]
      %% KIND: router
      layout_route["Layout Route<br/><i>Sidebar + auth redirect</i>"]
      %% KIND: router
      auth_routes["Auth Pages<br/><i>Login, Signup,<br/>Recover/Reset Password</i>"]
    end

    %% KIND: service_layer
    subgraph hooks_layer["Hooks"]
      %% KIND: service_layer
      auth_hooks["useAuth<br/><i>Login, logout, signup<br/>mutations &amp; user query</i>"]
      %% KIND: service_layer
      toast_hooks["useCustomToast<br/><i>Success/error notifications</i>"]
    end

    %% KIND: integration
    subgraph api_client["API Client (auto-generated)"]
      %% KIND: integration
      login_service["LoginService<br/><i>Auth token operations</i>"]
      %% KIND: integration
      users_service["UsersService<br/><i>User CRUD calls</i>"]
      %% KIND: integration
      items_service["ItemsService<br/><i>Item CRUD calls</i>"]
      %% KIND: integration
      utils_service["UtilsService<br/><i>Health check, test email</i>"]
    end

    %% KIND: service_layer
    query_client["React Query Client<br/><i>Cache, mutations,<br/>global error handling</i>"]

    %% KIND: boundary
    subgraph features["Feature Components"]
      %% KIND: service_layer
      admin_features["Admin Components<br/><i>AddUser, EditUser,<br/>DeleteUser, UserTable</i>"]
      %% KIND: service_layer
      item_features["Item Components<br/><i>AddItem, EditItem,<br/>DeleteItem, ItemTable</i>"]
      %% KIND: service_layer
      user_settings_feat["User Settings<br/><i>UserInfo, ChangePassword,<br/>DeleteAccount</i>"]
    end

    %% KIND: boundary
    subgraph common_ui["Common Components"]
      %% KIND: service_layer
      sidebar_comp["Sidebar<br/><i>Navigation &amp; user menu</i>"]
      %% KIND: service_layer
      data_table["DataTable<br/><i>Generic table with pagination</i>"]
      %% KIND: service_layer
      auth_layout["AuthLayout<br/><i>Login/signup page wrapper</i>"]
    end

    %% KIND: boundary
    ui_lib["UI Library<br/><i>shadcn/ui components<br/>Tailwind CSS</i>"]

    %% KIND: service_layer
    theme_prov["Theme Provider<br/><i>Dark/light mode toggle</i>"]
  end

  end_user -->|"Interacts"| tanstack_router
  admin_user -->|"Interacts"| tanstack_router

  tanstack_router --> root_route
  root_route --> layout_route
  root_route --> auth_routes

  layout_route --> admin_features
  layout_route --> item_features
  layout_route --> user_settings_feat

  auth_routes --> auth_hooks
  admin_features --> query_client
  item_features --> query_client
  user_settings_feat --> query_client
  auth_hooks --> query_client

  query_client --> api_client
  login_service --> fastapi_backend
  users_service --> fastapi_backend
  items_service --> fastapi_backend
  utils_service --> fastapi_backend

  admin_features --> common_ui
  item_features --> common_ui
  common_ui --> ui_lib
  features --> ui_lib
  theme_prov --> ui_lib
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Template
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ────────────────────────────────────────────
users                CONTAINS [end_user, admin_user]
fullstack_fastapi_boundary CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external             CONTAINS [smtp_server, sentry]

%% ── L2→L3 containment: fastapi_backend ─────────────────────────────
fastapi_backend      CONTAINS [middleware, api_router, auth_deps, crud_layer, models_layer, security_mod, core_config, email_utils, db_engine, alembic_mig]
  middleware         CONTAINS [cors_mw]
  api_router         CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  auth_deps          CONTAINS [oauth2_scheme, get_current_user, get_superuser, session_dep]

%% ── L2→L3 containment: spa ─────────────────────────────────────────
spa                  CONTAINS [routing, hooks_layer, api_client, query_client, features, common_ui, ui_lib, theme_prov]
  routing            CONTAINS [tanstack_router, root_route, layout_route, auth_routes]
  hooks_layer        CONTAINS [auth_hooks, toast_hooks]
  api_client         CONTAINS [login_service, users_service, items_service, utils_service]
  features           CONTAINS [admin_features, item_features, user_settings_feat]
  common_ui          CONTAINS [sidebar_comp, data_table, auth_layout]
```
