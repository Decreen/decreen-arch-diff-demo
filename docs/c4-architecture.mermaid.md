# C4 Architecture Model — Full Stack FastAPI Project

## L1: System Context

```mermaid
%% SCOPE: urn:c4:system:fullstack-platform
flowchart TD

    subgraph users["Users"]
        end_user["👤 End User\n<i>Person</i>\nBrowses dashboard, manages items"]
        admin_user["👤 Administrator\n<i>Person</i>\nManages users and platform"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Project"]
        fullstack_platform["Full Stack FastAPI Project\n<i>Software System</i>\nWeb application for managing\nusers and items"]
    end

    subgraph external["External Systems"]
        smtp_service["📧 SMTP Service\n<i>External System</i>\nEmail delivery"]
        sentry["🔍 Sentry\n<i>External System</i>\nError monitoring & tracing"]
    end

    end_user -->|"Uses via HTTPS"| fullstack_platform
    admin_user -->|"Administers via HTTPS"| fullstack_platform
    fullstack_platform -->|"Sends transactional\nemails via SMTP"| smtp_service
    fullstack_platform -->|"Reports errors\nand traces via HTTP"| sentry

    classDef person fill:#08427B,color:#fff,stroke:#073B6F
    classDef system fill:#1168BD,color:#fff,stroke:#0E5AA7
    classDef external_sys fill:#999999,color:#fff,stroke:#6B6B6B

    class end_user,admin_user person
    class fullstack_platform system
    class smtp_service,sentry external_sys
```

---

## L2: Container

```mermaid
%% SCOPE: urn:c4:system:fullstack-platform
flowchart TD

    subgraph users["Users"]
        end_user["👤 End User\n<i>Person</i>"]
        admin_user["👤 Administrator\n<i>Person</i>"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Project"]
        traefik["Traefik\n<i>Container: Traefik 3.6</i>\nReverse proxy, TLS termination,\nroute-based dispatch"]
        spa["React SPA\n<i>Container: React 19 / Vite / Nginx</i>\nSingle-page application\nserved as static files"]
        fastapi_backend["FastAPI Backend\n<i>Container: Python 3.10 / FastAPI</i>\nREST API with JWT auth\n4 Uvicorn workers"]
        pg["PostgreSQL\n<i>Database: PostgreSQL 18</i>\nRelational data store\nUsers, Items"]
        adminer["Adminer\n<i>Container: PHP</i>\nDatabase administration UI"]
    end

    subgraph external["External Systems"]
        smtp_service["📧 SMTP Service\n<i>External System</i>"]
        sentry["🔍 Sentry\n<i>External System</i>"]
    end

    end_user -->|"HTTPS\ndashboard.domain"| traefik
    admin_user -->|"HTTPS\nadminer.domain"| traefik
    admin_user -->|"HTTPS\ndashboard.domain"| traefik
    traefik -->|"HTTP :80"| spa
    traefik -->|"HTTP :8000"| fastapi_backend
    traefik -->|"HTTP :8080"| adminer
    spa -.->|"REST /api/v1\n(browser-side calls\nvia api.domain)"| fastapi_backend
    fastapi_backend -->|"SQL\npsycopg driver"| pg
    adminer -->|"SQL"| pg
    fastapi_backend -->|"SMTP\nTLS/SSL"| smtp_service
    fastapi_backend -->|"HTTP\nSentry SDK"| sentry

    classDef person fill:#08427B,color:#fff,stroke:#073B6F
    classDef container fill:#1168BD,color:#fff,stroke:#0E5AA7
    classDef database fill:#2D882D,color:#fff,stroke:#226622
    classDef external_sys fill:#999999,color:#fff,stroke:#6B6B6B
    classDef infra fill:#6B6B6B,color:#fff,stroke:#555555

    class end_user,admin_user person
    class spa,fastapi_backend container
    class pg database
    class adminer,traefik infra
    class smtp_service,sentry external_sys
```

---

## L3: FastAPI Backend

```mermaid
%% SCOPE: urn:c4:container:fastapi-backend
flowchart TD

    spa["React SPA"]
    pg["PostgreSQL"]
    smtp_service["SMTP Service"]
    sentry["Sentry"]

    subgraph fastapi_backend["FastAPI Backend"]

        %% KIND: boundary
        subgraph mw["Middleware"]
            cors_mw["CORS Middleware\n<i>Component</i>\nAllows cross-origin requests\nfrom frontend host"]
            %% KIND: service_layer
            auth_mw["JWT Auth Middleware\n<i>Component</i>\nOAuth2PasswordBearer\nValidates JWT tokens"]
            %% KIND: service_layer
            superuser_guard["Superuser Guard\n<i>Component</i>\nEnforces is_superuser\nprivilege check"]
        end

        %% KIND: router
        subgraph routes["API Routes"]
            %% KIND: router
            login_routes["Login Routes\n<i>Component</i>\nPOST /login/access-token\nPassword recovery & reset"]
            %% KIND: router
            user_routes["User Routes\n<i>Component</i>\nGET/POST/PATCH/DELETE /users\nSignup, profile, admin CRUD"]
            %% KIND: router
            item_routes["Item Routes\n<i>Component</i>\nGET/POST/PUT/DELETE /items\nOwner-scoped item management"]
            %% KIND: router
            utils_routes["Utils Routes\n<i>Component</i>\nGET /health-check\nPOST /test-email"]
        end

        %% KIND: service_layer
        subgraph services["Services"]
            %% KIND: service_layer
            crud_service["CRUD Service\n<i>Component</i>\ncreate/update/get User & Item\nauthenticate with timing-safe check"]
            %% KIND: service_layer
            email_service["Email Service\n<i>Component</i>\nsend_email, generate templates\nPassword reset, new account"]
        end

        %% KIND: boundary
        subgraph core["Core"]
            %% KIND: service_layer
            config_module["Configuration\n<i>Component: Pydantic Settings</i>\nLoads .env, validates config\nDatabase, SMTP, JWT settings"]
            %% KIND: service_layer
            security_module["Security\n<i>Component</i>\nJWT creation (HS256)\nArgon2 + Bcrypt password hashing"]
            %% KIND: data_access
            db_engine["DB Engine\n<i>Component: SQLAlchemy</i>\ncreate_engine, Session factory\npostgresql+psycopg"]
        end

        %% KIND: data_access
        subgraph models_layer["Models"]
            %% KIND: data_access
            user_model["User Model\n<i>Component: SQLModel</i>\nemail, hashed_password,\nis_active, is_superuser"]
            %% KIND: data_access
            item_model["Item Model\n<i>Component: SQLModel</i>\ntitle, description,\nowner_id FK → User"]
        end

    end

    spa -.->|"REST /api/v1"| routes
    routes --> mw
    login_routes --> crud_service
    login_routes --> email_service
    login_routes --> security_module
    user_routes --> crud_service
    user_routes --> email_service
    item_routes --> crud_service
    utils_routes --> email_service
    crud_service --> security_module
    crud_service --> models_layer
    crud_service --> db_engine
    email_service --> config_module
    security_module --> config_module
    db_engine --> config_module
    models_layer --> db_engine
    auth_mw --> security_module
    auth_mw --> db_engine
    db_engine -->|"SQL"| pg
    email_service -->|"SMTP"| smtp_service
    fastapi_backend -.->|"Sentry SDK"| sentry

    classDef component fill:#438DD5,color:#fff,stroke:#3A7BBD
    classDef mw_style fill:#85BBF0,color:#000,stroke:#6BA4D9
    classDef data fill:#2D882D,color:#fff,stroke:#226622
    classDef external_ref fill:#999999,color:#fff,stroke:#6B6B6B,stroke-dasharray: 5 5

    class cors_mw,auth_mw,superuser_guard mw_style
    class login_routes,user_routes,item_routes,utils_routes component
    class crud_service,email_service,config_module,security_module component
    class db_engine,user_model,item_model data
    class spa,pg,smtp_service,sentry external_ref
```

---

## L3: React SPA

```mermaid
%% SCOPE: urn:c4:container:spa
flowchart TD

    fastapi_backend["FastAPI Backend"]
    end_user["End User"]

    subgraph spa["React SPA"]

        %% KIND: router
        subgraph routing["Routing"]
            %% KIND: router
            tanstack_router["TanStack Router\n<i>Component</i>\nFile-based routing\nAuto code-splitting"]
        end

        %% KIND: service_layer
        subgraph state_mgmt["State Management"]
            %% KIND: service_layer
            query_client["React Query Client\n<i>Component</i>\nServer state caching\n401/403 error handling"]
            %% KIND: service_layer
            theme_context["Theme Provider\n<i>Component: next-themes</i>\nDark / light / system mode\nlocalStorage persistence"]
        end

        %% KIND: boundary
        subgraph features["Features"]
            %% KIND: router
            dashboard_page["Dashboard\n<i>Component</i>\nWelcome, current user info"]
            %% KIND: router
            items_feature["Items Management\n<i>Component</i>\nDataTable, Add/Edit/Delete Item\nOwner-scoped CRUD"]
            %% KIND: router
            admin_feature["Admin Panel\n<i>Component</i>\nDataTable, Add/Edit/Delete User\nSuperuser-only"]
            %% KIND: router
            settings_feature["User Settings\n<i>Component</i>\nProfile, change password\ndelete account"]
            %% KIND: router
            auth_feature["Auth Pages\n<i>Component</i>\nLogin, signup, password\nrecovery & reset"]
        end

        %% KIND: integration
        subgraph services_layer["Services"]
            %% KIND: integration
            api_client["OpenAPI Client\n<i>Component: @hey-api/openapi-ts</i>\nGenerated SDK, Axios transport\nBearer token injection"]
            %% KIND: service_layer
            auth_hook["useAuth Hook\n<i>Component</i>\nLogin/logout/signup\nlocalStorage token mgmt"]
        end

        %% KIND: boundary
        subgraph shared_ui["Shared UI"]
            sidebar_component["App Sidebar\n<i>Component</i>\nNavigation, user menu\ncontext-aware links"]
            data_table["Data Table\n<i>Component: TanStack Table</i>\nPaginated table with\nsort, search, actions"]
            ui_primitives["UI Primitives\n<i>Component: shadcn/ui + Radix</i>\nForm, Input, Button, Dialog\nTooltip, Sheet, Sonner"]
        end

    end

    end_user -->|"Interacts via browser"| tanstack_router
    tanstack_router --> features
    tanstack_router --> state_mgmt
    dashboard_page --> query_client
    items_feature --> query_client
    items_feature --> data_table
    admin_feature --> query_client
    admin_feature --> data_table
    settings_feature --> query_client
    auth_feature --> auth_hook
    query_client --> api_client
    auth_hook --> api_client
    features --> shared_ui
    api_client -->|"REST /api/v1\nAxios + Bearer JWT"| fastapi_backend

    classDef component fill:#438DD5,color:#fff,stroke:#3A7BBD
    classDef feature fill:#85BBF0,color:#000,stroke:#6BA4D9
    classDef ui fill:#B8D4F0,color:#000,stroke:#9BBDD9
    classDef external_ref fill:#999999,color:#fff,stroke:#6B6B6B,stroke-dasharray: 5 5

    class tanstack_router,query_client,theme_context,api_client,auth_hook component
    class dashboard_page,items_feature,admin_feature,settings_feature,auth_feature feature
    class sidebar_component,data_table,ui_primitives ui
    class fastapi_backend,end_user external_ref
```

---

## L3: Traefik Reverse Proxy

```mermaid
%% SCOPE: urn:c4:container:traefik
flowchart TD

    end_user["End User"]
    admin_user["Administrator"]
    spa["React SPA"]
    fastapi_backend["FastAPI Backend"]
    adminer["Adminer"]

    subgraph traefik["Traefik Reverse Proxy"]

        %% KIND: router
        entrypoints["Entrypoints\n<i>Component</i>\nHTTP :80, HTTPS :443\nTLS termination"]

        %% KIND: router
        https_redirect["HTTPS Redirect\n<i>Component: Middleware</i>\nRedirects HTTP → HTTPS"]

        %% KIND: router
        frontend_router["Frontend Router\n<i>Component</i>\nHost: dashboard.domain\n→ SPA container :80"]

        %% KIND: router
        backend_router["Backend Router\n<i>Component</i>\nHost: api.domain\n→ Backend container :8000"]

        %% KIND: router
        adminer_router["Adminer Router\n<i>Component</i>\nHost: adminer.domain\n→ Adminer container :8080"]

        %% KIND: integration
        cert_resolver["Let's Encrypt Resolver\n<i>Component</i>\nAutomatic TLS certificates\nvia ACME"]

    end

    end_user -->|"HTTPS"| entrypoints
    admin_user -->|"HTTPS"| entrypoints
    entrypoints --> https_redirect
    https_redirect --> frontend_router
    https_redirect --> backend_router
    https_redirect --> adminer_router
    entrypoints --> cert_resolver
    frontend_router -->|"HTTP :80"| spa
    backend_router -->|"HTTP :8000"| fastapi_backend
    adminer_router -->|"HTTP :8080"| adminer

    classDef component fill:#438DD5,color:#fff,stroke:#3A7BBD
    classDef external_ref fill:#999999,color:#fff,stroke:#6B6B6B,stroke-dasharray: 5 5

    class entrypoints,https_redirect,frontend_router,backend_router,adminer_router,cert_resolver component
    class end_user,admin_user,spa,fastapi_backend,adminer external_ref
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Project
%% Covers L1, L2, and all L3 diagrams
%% ═══════════════════════════════════════════════════════════════════

%% ── L1 top-level groups ─────────────────────────────────────────
users                CONTAINS [end_user, admin_user]
fullstack_boundary   CONTAINS [traefik, spa, fastapi_backend, pg, adminer]
external             CONTAINS [smtp_service, sentry]

%% ── L2→L3 internal containment ──────────────────────────────────

%% FastAPI Backend (L3: fastapi-backend)
fastapi_backend CONTAINS [mw, routes, services, core, models_layer]
  mw             CONTAINS [cors_mw, auth_mw, superuser_guard]
  routes         CONTAINS [login_routes, user_routes, item_routes, utils_routes]
  services       CONTAINS [crud_service, email_service]
  core           CONTAINS [config_module, security_module, db_engine]
  models_layer   CONTAINS [user_model, item_model]

%% React SPA (L3: spa)
spa CONTAINS [routing, state_mgmt, features, services_layer, shared_ui]
  routing        CONTAINS [tanstack_router]
  state_mgmt     CONTAINS [query_client, theme_context]
  features       CONTAINS [dashboard_page, items_feature, admin_feature, settings_feature, auth_feature]
  services_layer CONTAINS [api_client, auth_hook]
  shared_ui      CONTAINS [sidebar_component, data_table, ui_primitives]

%% Traefik Reverse Proxy (L3: traefik)
traefik CONTAINS [entrypoints, https_redirect, frontend_router, backend_router, adminer_router, cert_resolver]
```
