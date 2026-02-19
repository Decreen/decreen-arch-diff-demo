# C4 Architecture Model — Full Stack FastAPI Project

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
C4Context
    title L1 – System Context: Full Stack FastAPI Project

    Person(user, "User", "Registered application user who manages items")
    Person(admin, "Admin", "Superuser who manages users and all items")

    System_Boundary(system_boundary, "Full Stack FastAPI Project") {
        System(fullstack_system, "Full Stack FastAPI Project", "Web application for user and item management with JWT auth, built on FastAPI + React")
    }

    System_Ext(smtp, "SMTP Server", "Sends transactional emails: password recovery, new account notifications")
    System_Ext(sentry, "Sentry", "Collects error reports and performance traces in staging/production")

    Rel(user, fullstack_system, "Manages items, updates profile", "HTTPS")
    Rel(admin, fullstack_system, "Manages users and all items", "HTTPS")
    Rel(fullstack_system, smtp, "Sends emails", "SMTP/TLS")
    Rel(fullstack_system, sentry, "Reports errors & traces", "HTTPS")
```

---

## L2: Container Diagram

```mermaid
%% SCOPE: urn:c4:system:fullstack-fastapi
C4Container
    title L2 – Container Diagram: Full Stack FastAPI Project

    Person(user, "User", "Registered application user")
    Person(admin, "Admin", "Superuser")

    System_Boundary(system_boundary, "Full Stack FastAPI Project") {
        Container(spa, "React SPA", "React, TypeScript, Vite, TanStack Router/Query, Tailwind, shadcn/ui", "Single-page dashboard served via Nginx; communicates with backend API")
        Container(traefik, "Traefik Proxy", "Traefik v3", "Reverse proxy and load balancer; handles TLS termination and routing")
        Container(fastapi_backend, "FastAPI Backend", "Python, FastAPI, Uvicorn, SQLModel, Pydantic", "REST API serving /api/v1/*; handles auth, CRUD, email dispatch")
        ContainerDb(pg, "PostgreSQL", "PostgreSQL 18", "Stores users, items, and Alembic migration state")
    }

    System_Ext(smtp, "SMTP Server", "Transactional email delivery")
    System_Ext(sentry, "Sentry", "Error and performance monitoring")

    Rel(user, traefik, "Browses dashboard", "HTTPS")
    Rel(admin, traefik, "Browses dashboard", "HTTPS")
    Rel(traefik, spa, "Routes dashboard.* requests", "HTTP")
    Rel(traefik, fastapi_backend, "Routes api.* requests", "HTTP")
    Rel(spa, fastapi_backend, "API calls", "HTTP/JSON via /api/v1")
    Rel(fastapi_backend, pg, "Reads/writes data", "psycopg (PostgreSQL wire protocol)")
    Rel(fastapi_backend, smtp, "Sends emails", "SMTP/TLS")
    Rel(fastapi_backend, sentry, "Reports errors", "HTTPS")
```

---

## L3: FastAPI Backend (Component Diagram)

```mermaid
%% SCOPE: urn:c4:container:fastapi-backend
C4Component
    title L3 – Component Diagram: FastAPI Backend

    Container_Boundary(fastapi_backend, "FastAPI Backend") {

        %% KIND: boundary
        Component(cors_mw, "CORS Middleware", "Starlette CORSMiddleware", "Enforces allowed origins, methods, headers for cross-origin requests")

        %% KIND: router
        Component(api_router, "API Router", "FastAPI APIRouter", "Top-level /api/v1 router; mounts all sub-routers")

        %% KIND: router
        Component(login_routes, "Login Routes", "FastAPI APIRouter", "POST /login/access-token, /password-recovery, /reset-password")

        %% KIND: router
        Component(users_routes, "Users Routes", "FastAPI APIRouter", "CRUD /users, /users/me, /users/signup, /users/{id}")

        %% KIND: router
        Component(items_routes, "Items Routes", "FastAPI APIRouter", "CRUD /items, /items/{id}")

        %% KIND: router
        Component(utils_routes, "Utils Routes", "FastAPI APIRouter", "GET /utils/health-check, POST /utils/test-email")

        %% KIND: service_layer
        Component(auth_deps, "Auth Dependencies", "FastAPI Depends", "OAuth2 bearer extraction, JWT validation, current-user/superuser resolution")

        %% KIND: data_access
        Component(crud_layer, "CRUD Layer", "SQLModel Session", "create_user, update_user, get_user_by_email, authenticate, create_item")

        %% KIND: data_access
        Component(models_layer, "SQLModel Models", "SQLModel, Pydantic", "User, Item, Token, schema models for request/response validation")

        %% KIND: service_layer
        Component(security, "Security Module", "PyJWT, pwdlib", "JWT token creation (HS256), Argon2/Bcrypt password hashing and verification")

        %% KIND: service_layer
        Component(config, "Settings / Config", "Pydantic Settings", "Loads .env, validates config, computes SQLALCHEMY_DATABASE_URI and CORS origins")

        %% KIND: storage
        Component(db_engine, "DB Engine", "SQLAlchemy Engine", "Creates engine from DATABASE_URI; provides Session factory; runs init_db seeding")

        %% KIND: integration
        Component(email_utils, "Email Utilities", "python-emails, Jinja2", "Renders templates, sends transactional emails, generates/verifies password-reset tokens")
    }

    ContainerDb(pg, "PostgreSQL", "PostgreSQL 18", "User and item storage")
    System_Ext(smtp, "SMTP Server", "Email delivery")

    Rel(cors_mw, api_router, "Passes requests after CORS check")
    Rel(api_router, login_routes, "Mounts /login, /password-recovery, /reset-password")
    Rel(api_router, users_routes, "Mounts /users")
    Rel(api_router, items_routes, "Mounts /items")
    Rel(api_router, utils_routes, "Mounts /utils")

    Rel(login_routes, auth_deps, "Uses for token test")
    Rel(login_routes, crud_layer, "authenticate()")
    Rel(login_routes, security, "create_access_token()")
    Rel(login_routes, email_utils, "Password recovery emails")

    Rel(users_routes, auth_deps, "Current user / superuser guard")
    Rel(users_routes, crud_layer, "User CRUD operations")
    Rel(users_routes, email_utils, "New account notification")

    Rel(items_routes, auth_deps, "Current user guard")
    Rel(items_routes, models_layer, "Item model validation")

    Rel(utils_routes, auth_deps, "Superuser guard for test-email")
    Rel(utils_routes, email_utils, "send_email()")

    Rel(auth_deps, security, "JWT decode + verify")
    Rel(auth_deps, db_engine, "get_db() session")
    Rel(auth_deps, models_layer, "User lookup")

    Rel(crud_layer, models_layer, "Validates and persists models")
    Rel(crud_layer, security, "verify_password, get_password_hash")
    Rel(crud_layer, db_engine, "Session operations")

    Rel(db_engine, pg, "SQL queries", "psycopg")
    Rel(db_engine, config, "Reads SQLALCHEMY_DATABASE_URI")

    Rel(security, config, "Reads SECRET_KEY")

    Rel(email_utils, smtp, "Sends email", "SMTP/TLS")
    Rel(email_utils, config, "Reads SMTP and email settings")
    Rel(email_utils, security, "JWT for password-reset tokens")
```

---

## L3: React SPA (Component Diagram)

```mermaid
%% SCOPE: urn:c4:container:spa
C4Component
    title L3 – Component Diagram: React SPA

    Container_Boundary(spa, "React SPA") {

        %% KIND: router
        Component(tanstack_router, "TanStack Router", "TanStack Router, file-based routes", "Client-side routing with route guards (isLoggedIn redirect)")

        %% KIND: boundary
        Component(auth_pages, "Auth Pages", "React components", "Login, Signup, Recover Password, Reset Password pages")

        %% KIND: boundary
        Component(dashboard_layout, "Dashboard Layout", "React component", "Authenticated shell with sidebar, header, footer; wraps child routes")

        %% KIND: boundary
        Component(admin_feature, "Admin Feature", "React components", "User list, add/edit/delete user dialogs (superuser only)")

        %% KIND: boundary
        Component(items_feature, "Items Feature", "React components", "Item list, add/edit/delete item dialogs with DataTable")

        %% KIND: boundary
        Component(settings_feature, "User Settings Feature", "React components", "User info update, change password, delete account")

        %% KIND: service_layer
        Component(hooks_layer, "Custom Hooks", "React hooks", "useAuth (login/signup/logout), useCustomToast, useCopyToClipboard, useMobile")

        %% KIND: integration
        Component(api_client, "Generated API Client", "openapi-ts SDK", "Type-safe HTTP client auto-generated from OpenAPI spec; ItemsService, LoginService, UsersService, UtilsService")

        %% KIND: service_layer
        Component(query_layer, "TanStack Query", "React Query", "QueryClient with QueryCache/MutationCache; handles caching, refetching, and 401/403 error interception")

        %% KIND: service_layer
        Component(theme_provider, "Theme Provider", "React context", "Dark/light mode toggle persisted to localStorage")

        %% KIND: boundary
        Component(ui_components, "UI Component Library", "shadcn/ui, Tailwind CSS", "Buttons, forms, dialogs, tables, sidebar, sheets, tooltips, sonner toasts")
    }

    Container(fastapi_backend, "FastAPI Backend", "Python, FastAPI", "REST API")

    Rel(tanstack_router, auth_pages, "Routes /login, /signup, /recover-password, /reset-password")
    Rel(tanstack_router, dashboard_layout, "Routes /_layout/* (authenticated)")
    Rel(dashboard_layout, admin_feature, "Renders /admin route")
    Rel(dashboard_layout, items_feature, "Renders /items route")
    Rel(dashboard_layout, settings_feature, "Renders /settings route")

    Rel(auth_pages, hooks_layer, "useAuth() for login/signup")
    Rel(auth_pages, ui_components, "Form, Input, LoadingButton, PasswordInput")
    Rel(admin_feature, hooks_layer, "useAuth() for current user")
    Rel(admin_feature, query_layer, "useQuery/useMutation for user CRUD")
    Rel(items_feature, query_layer, "useQuery/useMutation for item CRUD")
    Rel(settings_feature, hooks_layer, "useAuth() for current user")
    Rel(settings_feature, query_layer, "useMutation for profile/password update")

    Rel(hooks_layer, api_client, "Calls LoginService, UsersService")
    Rel(query_layer, api_client, "All API calls go through SDK")
    Rel(api_client, fastapi_backend, "HTTP/JSON calls to /api/v1/*", "HTTPS")

    Rel(dashboard_layout, ui_components, "Sidebar, SidebarInset, Footer")
    Rel(admin_feature, ui_components, "DataTable, Dialog, Form")
    Rel(items_feature, ui_components, "DataTable, Dialog, Form")
    Rel(settings_feature, ui_components, "Card, Form, Input, Dialog")
```

---

## L3: Traefik Proxy (Component Diagram)

```mermaid
%% SCOPE: urn:c4:container:traefik
C4Component
    title L3 – Component Diagram: Traefik Proxy

    Container_Boundary(traefik, "Traefik Proxy") {

        %% KIND: router
        Component(entrypoint_http, "HTTP Entrypoint", "Traefik entrypoint", "Listens on port 80; redirects to HTTPS in production")

        %% KIND: router
        Component(entrypoint_https, "HTTPS Entrypoint", "Traefik entrypoint", "Listens on port 443; terminates TLS via Let's Encrypt")

        %% KIND: service_layer
        Component(https_redirect_mw, "HTTPS Redirect Middleware", "Traefik middleware", "Redirects HTTP to HTTPS with 301 permanent redirect")

        %% KIND: service_layer
        Component(docker_provider, "Docker Provider", "Traefik provider", "Reads container labels to discover routes and services dynamically")

        %% KIND: service_layer
        Component(le_resolver, "Let's Encrypt Resolver", "ACME TLS challenge", "Obtains and renews TLS certificates automatically")

        %% KIND: router
        Component(frontend_router, "Frontend Router", "Traefik router rule", "Host(dashboard.*) -> forwards to SPA Nginx on port 80")

        %% KIND: router
        Component(backend_router, "Backend Router", "Traefik router rule", "Host(api.*) -> forwards to FastAPI on port 8000")

        %% KIND: router
        Component(adminer_router, "Adminer Router", "Traefik router rule", "Host(adminer.*) -> forwards to Adminer on port 8080")
    }

    Container(spa, "React SPA", "Nginx", "Frontend")
    Container(fastapi_backend, "FastAPI Backend", "Uvicorn", "Backend API")

    Rel(entrypoint_http, https_redirect_mw, "All HTTP traffic")
    Rel(https_redirect_mw, entrypoint_https, "Redirects to HTTPS")
    Rel(entrypoint_https, frontend_router, "dashboard.* requests")
    Rel(entrypoint_https, backend_router, "api.* requests")
    Rel(entrypoint_https, adminer_router, "adminer.* requests")
    Rel(frontend_router, spa, "Proxy to port 80")
    Rel(backend_router, fastapi_backend, "Proxy to port 8000")
    Rel(docker_provider, frontend_router, "Discovers route labels")
    Rel(docker_provider, backend_router, "Discovers route labels")
    Rel(docker_provider, adminer_router, "Discovers route labels")
    Rel(le_resolver, entrypoint_https, "Provisions TLS certificates")
```

---

## Containment Map

```text
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Project
%% Every subgraph and node from every diagram level.
%% Nesting expressed by indentation.
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users            CONTAINS [user, admin]
system_boundary  CONTAINS [fullstack_system]
external         CONTAINS [smtp, sentry]

%% ── L2 system boundary → containers ─────────────────────────────
system_boundary  CONTAINS [spa, traefik, fastapi_backend, pg]

%% ── L3: FastAPI Backend → components ────────────────────────────
fastapi_backend  CONTAINS [cors_mw, api_router, login_routes, users_routes, items_routes, utils_routes, auth_deps, crud_layer, models_layer, security, config, db_engine, email_utils]
  api_router       CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  auth_deps        CONTAINS []
  crud_layer       CONTAINS []
  models_layer     CONTAINS []
  security         CONTAINS []
  config           CONTAINS []
  db_engine        CONTAINS []
  email_utils      CONTAINS []
  cors_mw          CONTAINS []

%% ── L3: React SPA → components ──────────────────────────────────
spa              CONTAINS [tanstack_router, auth_pages, dashboard_layout, admin_feature, items_feature, settings_feature, hooks_layer, api_client, query_layer, theme_provider, ui_components]
  tanstack_router  CONTAINS [auth_pages, dashboard_layout]
  dashboard_layout CONTAINS [admin_feature, items_feature, settings_feature]
  auth_pages       CONTAINS []
  admin_feature    CONTAINS []
  items_feature    CONTAINS []
  settings_feature CONTAINS []
  hooks_layer      CONTAINS []
  api_client       CONTAINS []
  query_layer      CONTAINS []
  theme_provider   CONTAINS []
  ui_components    CONTAINS []

%% ── L3: Traefik Proxy → components ──────────────────────────────
traefik          CONTAINS [entrypoint_http, entrypoint_https, https_redirect_mw, docker_provider, le_resolver, frontend_router, backend_router, adminer_router]
  entrypoint_http    CONTAINS []
  entrypoint_https   CONTAINS []
  https_redirect_mw  CONTAINS []
  docker_provider    CONTAINS []
  le_resolver        CONTAINS []
  frontend_router    CONTAINS []
  backend_router     CONTAINS []
  adminer_router     CONTAINS []

%% ── Cross-level ID consistency table ────────────────────────────
%% ID                 | L1              | L2                | L3 (scope boundary)
%% ─────────────────────────────────────────────────────────────────
%% user               | Person          | Person            | —
%% admin              | Person          | Person            | —
%% fullstack_system   | System          | (expanded)        | —
%% spa                | —               | Container         | Container_Boundary (L3: React SPA)
%% fastapi_backend    | —               | Container         | Container_Boundary (L3: FastAPI Backend)
%% traefik            | —               | Container         | Container_Boundary (L3: Traefik Proxy)
%% pg                 | —               | ContainerDb       | External in L3: FastAPI Backend
%% smtp               | System_Ext      | System_Ext        | External in L3: FastAPI Backend
%% sentry             | System_Ext      | System_Ext        | —
```
