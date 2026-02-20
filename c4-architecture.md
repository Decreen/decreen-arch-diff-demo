# C4 Architecture Model — Full Stack FastAPI Template

---

## L1: System Context

```mermaid
C4Context
    title L1 System Context — Full Stack FastAPI Platform

    %% SCOPE: urn:c4:system:fastapi_platform

    subgraph users ["Users"]
        Person(end_user, "End User", "Manages personal items via the dashboard")
        Person(admin_user, "Admin User", "Manages users and system configuration")
    end

    subgraph fastapi_boundary ["Full Stack FastAPI Platform"]
        System(fastapi_platform, "Full Stack FastAPI Platform", "Web application for user and item management with JWT-based auth")
    end

    subgraph external ["External Services"]
        System_Ext(smtp_server, "SMTP Server", "Delivers transactional emails: password recovery, account creation")
        System_Ext(sentry, "Sentry", "Error tracking and performance monitoring")
    end

    Rel(end_user, fastapi_platform, "Browses dashboard, manages items", "HTTPS")
    Rel(admin_user, fastapi_platform, "Manages users, sends test emails", "HTTPS")
    Rel(fastapi_platform, smtp_server, "Sends transactional emails", "SMTP/TLS")
    Rel(fastapi_platform, sentry, "Reports errors and traces", "HTTPS")
```

---

## L2: Container Diagram

```mermaid
C4Container
    title L2 Container Diagram — Full Stack FastAPI Platform

    %% SCOPE: urn:c4:system:fastapi_platform

    Person(end_user, "End User", "Manages personal items")
    Person(admin_user, "Admin User", "Manages users and config")

    subgraph fastapi_boundary ["Full Stack FastAPI Platform"]
        Container(react_spa, "React SPA", "React, TypeScript, Vite, TanStack Router, TanStack Query, shadcn/ui", "Single-page dashboard for users and admins")
        Container(fastapi_backend, "FastAPI Backend", "Python, FastAPI, SQLModel, Pydantic", "REST API serving /api/v1 endpoints with JWT auth")
        Container(traefik, "Traefik Reverse Proxy", "Traefik 3.x", "Routes traffic, terminates TLS, load balances")
        ContainerDb(pg, "PostgreSQL", "PostgreSQL 18", "Stores users, items, and application data")
        Container(prestart, "Prestart Job", "Bash, Alembic, Python", "Runs DB migrations and seeds initial data on deploy")
        Container(adminer, "Adminer", "Adminer", "Web-based database administration UI")
    end

    System_Ext(smtp_server, "SMTP Server", "Delivers transactional emails")
    System_Ext(sentry, "Sentry", "Error tracking")

    Rel(end_user, traefik, "HTTPS requests", "HTTPS")
    Rel(admin_user, traefik, "HTTPS requests", "HTTPS")
    Rel(traefik, react_spa, "Serves frontend", "HTTP :80")
    Rel(traefik, fastapi_backend, "Proxies API calls", "HTTP :8000")
    Rel(traefik, adminer, "Proxies admin UI", "HTTP :8080")
    Rel(react_spa, fastapi_backend, "API calls", "HTTP /api/v1")
    Rel(fastapi_backend, pg, "Reads/writes data", "psycopg PostgreSQL protocol")
    Rel(fastapi_backend, smtp_server, "Sends emails", "SMTP/TLS")
    Rel(fastapi_backend, sentry, "Reports errors", "HTTPS")
    Rel(prestart, pg, "Runs Alembic migrations, seeds data", "psycopg")
    Rel(adminer, pg, "Direct DB access", "PostgreSQL protocol")
```

---

## L3: FastAPI Backend (Component Diagram)

```mermaid
C4Component
    title L3 Component Diagram — FastAPI Backend

    %% SCOPE: urn:c4:container:fastapi_backend

    subgraph fastapi_backend ["FastAPI Backend"]

        %% KIND: boundary
        subgraph middleware ["Middleware"]
            %% KIND: router
            Component(cors_mw, "CORS Middleware", "Starlette CORSMiddleware", "Enforces allowed origins, methods, headers")
        end

        %% KIND: router
        subgraph auth_deps ["Auth Dependencies"]
            Component(oauth2_scheme, "OAuth2 Bearer", "FastAPI OAuth2PasswordBearer", "Extracts JWT from Authorization header")
            Component(get_current_user, "get_current_user", "FastAPI Dependency", "Validates JWT, resolves User from DB")
            Component(get_superuser, "get_current_active_superuser", "FastAPI Dependency", "Asserts superuser privilege")
        end

        %% KIND: router
        subgraph routes ["API Routes"]
            Component(login_routes, "Login Routes", "/login/*, /password-recovery/*, /reset-password/", "OAuth2 token login, password recovery and reset")
            Component(user_routes, "User Routes", "/users/*", "CRUD for users, self-service profile, signup")
            Component(item_routes, "Item Routes", "/items/*", "CRUD for items, ownership-scoped")
            Component(util_routes, "Utility Routes", "/utils/*", "Health check, test email")
            Component(private_routes, "Private Routes", "/private/* (local only)", "Internal user creation for dev/testing")
        end

        %% KIND: service_layer
        subgraph services ["Service / CRUD Layer"]
            Component(crud_layer, "CRUD Module", "app.crud", "create_user, update_user, authenticate, create_item")
        end

        %% KIND: service_layer
        subgraph email_service ["Email Service"]
            Component(email_utils, "Email Utilities", "app.utils", "Renders Jinja2 templates, sends via SMTP, generates/verifies reset tokens")
        end

        %% KIND: data_access
        subgraph data_layer ["Data Layer"]
            Component(models_layer, "SQLModel Models", "app.models", "User, Item, Token, Pydantic schemas")
            Component(db_engine, "Database Engine", "SQLAlchemy create_engine", "Connection pool to PostgreSQL")
        end

        %% KIND: service_layer
        subgraph security_layer ["Security"]
            Component(security_mod, "Security Module", "app.core.security", "JWT creation, Argon2/Bcrypt password hashing")
            Component(config_mod, "Configuration", "app.core.config (Pydantic Settings)", "Loads env vars, validates settings, computes DB URI")
        end

    end

    ContainerDb(pg, "PostgreSQL", "PostgreSQL 18", "Stores users and items")
    System_Ext(smtp_server, "SMTP Server", "Delivers emails")
    System_Ext(sentry, "Sentry", "Error tracking")
    Container(react_spa, "React SPA", "React", "Frontend client")

    Rel(react_spa, cors_mw, "API requests", "HTTP")
    Rel(cors_mw, login_routes, "Passes through", "")
    Rel(cors_mw, user_routes, "Passes through", "")
    Rel(cors_mw, item_routes, "Passes through", "")
    Rel(cors_mw, util_routes, "Passes through", "")
    Rel(login_routes, crud_layer, "authenticate, get_user_by_email", "")
    Rel(login_routes, security_mod, "create_access_token", "")
    Rel(login_routes, email_utils, "send password recovery email", "")
    Rel(user_routes, crud_layer, "create_user, update_user", "")
    Rel(user_routes, email_utils, "send new account email", "")
    Rel(user_routes, get_current_user, "Validates auth", "")
    Rel(item_routes, get_current_user, "Validates auth", "")
    Rel(item_routes, models_layer, "Item queries", "")
    Rel(util_routes, get_superuser, "Superuser only", "")
    Rel(util_routes, email_utils, "send test email", "")
    Rel(get_current_user, oauth2_scheme, "Extracts token", "")
    Rel(get_current_user, security_mod, "Decodes JWT", "")
    Rel(get_current_user, db_engine, "Session query", "")
    Rel(get_superuser, get_current_user, "Extends check", "")
    Rel(crud_layer, models_layer, "Uses ORM models", "")
    Rel(crud_layer, security_mod, "Hash/verify passwords", "")
    Rel(crud_layer, db_engine, "Session operations", "")
    Rel(db_engine, pg, "SQL queries", "psycopg")
    Rel(email_utils, smtp_server, "Sends email", "SMTP")
    Rel(email_utils, security_mod, "Generate/verify reset tokens", "")
    Rel(config_mod, sentry, "Initializes Sentry SDK", "HTTPS")
```

---

## L3: React SPA (Component Diagram)

```mermaid
C4Component
    title L3 Component Diagram — React SPA

    %% SCOPE: urn:c4:container:react_spa

    subgraph react_spa ["React SPA"]

        %% KIND: router
        subgraph routing ["Routing Layer"]
            Component(tanstack_router, "TanStack Router", "File-based routing", "Root route, layout, auth guards, code-split pages")
            Component(query_client, "TanStack Query Client", "QueryClient, QueryCache, MutationCache", "Data fetching, caching, auto-logout on 401/403")
        end

        %% KIND: boundary
        subgraph pages ["Page Routes"]
            Component(login_page, "Login Page", "/login", "Email/password login form")
            Component(signup_page, "Signup Page", "/signup", "Self-service registration")
            Component(recover_page, "Recover Password", "/recover-password", "Request password reset email")
            Component(reset_page, "Reset Password", "/reset-password", "Set new password with token")
            Component(dashboard_page, "Dashboard", "/", "Welcome page for authenticated users")
            Component(items_page, "Items Page", "/items", "List, create, edit, delete items")
            Component(admin_page, "Admin Page", "/admin", "User management (superuser only)")
            Component(settings_page, "Settings Page", "/settings", "Profile edit, password change, account deletion")
        end

        %% KIND: service_layer
        subgraph hooks_layer ["Hooks"]
            Component(use_auth, "useAuth", "Custom hook", "Login/logout/signup mutations, current user query")
            Component(use_toast, "useCustomToast", "Custom hook", "Error/success toast notifications")
        end

        %% KIND: integration
        subgraph api_layer ["API Client"]
            Component(api_client, "OpenAPI Client", "Auto-generated @hey-api/openapi-ts", "ItemsService, LoginService, UsersService, UtilsService")
            Component(openapi_config, "OpenAPI Config", "OpenAPI.BASE, OpenAPI.TOKEN", "Base URL and token injection from localStorage")
        end

        %% KIND: boundary
        subgraph ui_layer ["UI Framework"]
            Component(shadcn_ui, "shadcn/ui Components", "Tailwind CSS + Radix UI", "Button, Dialog, Form, Table, Sidebar, etc.")
            Component(theme_provider, "Theme Provider", "ThemeProvider context", "Dark/light mode toggle and persistence")
        end

    end

    Container(fastapi_backend, "FastAPI Backend", "FastAPI", "REST API")

    Rel(tanstack_router, login_page, "Routes to", "")
    Rel(tanstack_router, signup_page, "Routes to", "")
    Rel(tanstack_router, dashboard_page, "Routes to (auth guard)", "")
    Rel(tanstack_router, items_page, "Routes to (auth guard)", "")
    Rel(tanstack_router, admin_page, "Routes to (auth guard)", "")
    Rel(tanstack_router, settings_page, "Routes to (auth guard)", "")
    Rel(tanstack_router, recover_page, "Routes to", "")
    Rel(tanstack_router, reset_page, "Routes to", "")
    Rel(login_page, use_auth, "loginMutation", "")
    Rel(signup_page, use_auth, "signUpMutation", "")
    Rel(dashboard_page, use_auth, "current user", "")
    Rel(items_page, api_client, "ItemsService CRUD", "")
    Rel(admin_page, api_client, "UsersService CRUD", "")
    Rel(settings_page, api_client, "UsersService update/delete", "")
    Rel(use_auth, api_client, "LoginService, UsersService", "")
    Rel(api_client, openapi_config, "Reads BASE URL and TOKEN", "")
    Rel(api_client, fastapi_backend, "HTTP requests to /api/v1", "HTTP/JSON")
    Rel(query_client, api_client, "Manages request lifecycle", "")
    Rel(login_page, shadcn_ui, "Uses form components", "")
    Rel(items_page, shadcn_ui, "Uses table, dialog components", "")
    Rel(admin_page, shadcn_ui, "Uses table, dialog components", "")
    Rel(settings_page, shadcn_ui, "Uses form components", "")
    Rel(theme_provider, shadcn_ui, "Provides theme context", "")
```

---

## L3: Traefik Reverse Proxy (Component Diagram)

```mermaid
C4Component
    title L3 Component Diagram — Traefik Reverse Proxy

    %% SCOPE: urn:c4:container:traefik

    subgraph traefik ["Traefik Reverse Proxy"]

        %% KIND: router
        Component(entrypoint_http, "HTTP Entrypoint", "Port 80", "Receives HTTP traffic, redirects to HTTPS in production")
        %% KIND: router
        Component(entrypoint_https, "HTTPS Entrypoint", "Port 443", "Terminates TLS with Let's Encrypt certificates")
        %% KIND: router
        Component(docker_provider, "Docker Provider", "Traefik Docker provider", "Discovers services via container labels")
        %% KIND: router
        Component(https_redirect_mw, "HTTPS Redirect Middleware", "redirectscheme", "Redirects HTTP to HTTPS")
        %% KIND: router
        Component(cert_resolver, "Let's Encrypt Resolver", "ACME TLS Challenge", "Automatic TLS certificate provisioning")
        %% KIND: router
        Component(backend_router, "Backend Router", "Host: api.DOMAIN", "Routes to FastAPI backend on port 8000")
        %% KIND: router
        Component(frontend_router, "Frontend Router", "Host: dashboard.DOMAIN", "Routes to React SPA on port 80")
        %% KIND: router
        Component(adminer_router, "Adminer Router", "Host: adminer.DOMAIN", "Routes to Adminer on port 8080")

    end

    Person(end_user, "End User", "")
    Container(react_spa, "React SPA", "React", "")
    Container(fastapi_backend, "FastAPI Backend", "FastAPI", "")
    Container(adminer, "Adminer", "Adminer", "")

    Rel(end_user, entrypoint_http, "HTTP request", "TCP :80")
    Rel(end_user, entrypoint_https, "HTTPS request", "TCP :443")
    Rel(entrypoint_http, https_redirect_mw, "Applies redirect", "")
    Rel(entrypoint_https, cert_resolver, "TLS termination", "")
    Rel(entrypoint_https, docker_provider, "Discovers routes", "")
    Rel(docker_provider, backend_router, "Matches Host rule", "")
    Rel(docker_provider, frontend_router, "Matches Host rule", "")
    Rel(docker_provider, adminer_router, "Matches Host rule", "")
    Rel(backend_router, fastapi_backend, "Proxies to", "HTTP :8000")
    Rel(frontend_router, react_spa, "Proxies to", "HTTP :80")
    Rel(adminer_router, adminer, "Proxies to", "HTTP :8080")
```

---

## Containment Map

```
%% ══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Platform
%% ══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users              CONTAINS [end_user, admin_user]
fastapi_boundary   CONTAINS [fastapi_platform]
external           CONTAINS [smtp_server, sentry]

%% ── L2 system → containers ──────────────────────────────────────
fastapi_boundary   CONTAINS [react_spa, fastapi_backend, traefik, pg, prestart, adminer]

%% ── L3: fastapi_backend internals ───────────────────────────────
fastapi_backend    CONTAINS [middleware, auth_deps, routes, services, email_service, data_layer, security_layer]
  middleware       CONTAINS [cors_mw]
  auth_deps        CONTAINS [oauth2_scheme, get_current_user, get_superuser]
  routes           CONTAINS [login_routes, user_routes, item_routes, util_routes, private_routes]
  services         CONTAINS [crud_layer]
  email_service    CONTAINS [email_utils]
  data_layer       CONTAINS [models_layer, db_engine]
  security_layer   CONTAINS [security_mod, config_mod]

%% ── L3: react_spa internals ─────────────────────────────────────
react_spa          CONTAINS [routing, pages, hooks_layer, api_layer, ui_layer]
  routing          CONTAINS [tanstack_router, query_client]
  pages            CONTAINS [login_page, signup_page, recover_page, reset_page, dashboard_page, items_page, admin_page, settings_page]
  hooks_layer      CONTAINS [use_auth, use_toast]
  api_layer        CONTAINS [api_client, openapi_config]
  ui_layer         CONTAINS [shadcn_ui, theme_provider]

%% ── L3: traefik internals ───────────────────────────────────────
traefik            CONTAINS [entrypoint_http, entrypoint_https, docker_provider, https_redirect_mw, cert_resolver, backend_router, frontend_router, adminer_router]
```
