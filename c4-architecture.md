# C4 Architecture Model — Full Stack FastAPI Project

> Auto-generated C4 model for the
> [Full Stack FastAPI Project](https://github.com/fastapi/full-stack-fastapi-template).

---

## L1: System Context

```mermaid
graph TB
    %% SCOPE: urn:c4:system:fullstack_platform

    subgraph users["Users"]
        user["End User\n[Person]\nManages personal items\nvia the web application"]
        admin_user["Admin User\n[Person]\nManages users and\nsystem configuration"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Project"]
        fullstack_platform["Full Stack FastAPI Project\n[Software System]\nWeb application for user\nand item management"]
    end

    subgraph external["External Services"]
        smtp["SMTP Service\n[External System]\nTransactional email delivery"]
        sentry["Sentry\n[External System]\nError monitoring and alerting"]
    end

    user -->|"Browses and manages items"| fullstack_platform
    admin_user -->|"Manages users and settings"| fullstack_platform
    fullstack_platform -->|"Sends emails via SMTP"| smtp
    fullstack_platform -->|"Reports errors"| sentry

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff

    class user,admin_user person
    class fullstack_platform system
    class smtp,sentry ext
```

---

## L2: Container Diagram

```mermaid
graph TB
    %% SCOPE: urn:c4:system:fullstack_platform

    subgraph users["Users"]
        user["End User\n[Person]"]
        admin_user["Admin User\n[Person]"]
    end

    subgraph fullstack_boundary["Full Stack FastAPI Project"]
        react_spa["React SPA\n[Container: React 19 / TypeScript / Vite]\nSingle-page application\nserved by Nginx"]
        fastapi_api["FastAPI API\n[Container: Python / FastAPI]\nREST API at /api/v1\nJWT auth · CRUD operations"]
        pg[("PostgreSQL\n[Container: PostgreSQL 18]\nStores users and items")]
    end

    subgraph external["External Services"]
        smtp["SMTP Service\n[External System]"]
        sentry["Sentry\n[External System]"]
    end

    user -->|"Interacts with UI"| react_spa
    admin_user -->|"Manages users via UI"| react_spa
    react_spa -->|"REST API calls\nJSON / HTTPS"| fastapi_api
    fastapi_api -->|"SQL queries\nSQLModel / SQLAlchemy"| pg
    fastapi_api -->|"Sends emails\nSMTP protocol"| smtp
    fastapi_api -->|"Reports errors\nSentry SDK"| sentry

    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef container fill:#1168bd,stroke:#0b4884,color:#fff
    classDef db fill:#2b78c4,stroke:#1a5276,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff

    class user,admin_user person
    class react_spa,fastapi_api container
    class pg db
    class smtp,sentry ext
```

---

## L3: FastAPI API

```mermaid
graph TB
    %% SCOPE: urn:c4:container:fastapi_api

    react_spa["React SPA\n[Container]"]

    subgraph fastapi_api["FastAPI API"]

        %% KIND: boundary
        subgraph middleware["Middleware & Dependencies"]
            %% KIND: router
            cors_mw["CORS Middleware\n[Component: CORSMiddleware]\nCross-origin request handling"]
            %% KIND: service_layer
            auth_dep["Auth Dependency\n[Component: OAuth2 / JWT]\nToken validation · user extraction"]
            %% KIND: data_access
            db_dep["DB Session\n[Component: get_db]\nSQLAlchemy session per request"]
            %% KIND: service_layer
            superuser_dep["Superuser Guard\n[Component: Dependency]\nEnforces superuser privilege"]
        end

        %% KIND: boundary
        subgraph routes["API Routes (/api/v1)"]
            %% KIND: router
            login_routes["Login Routes\n[Component: APIRouter]\nPOST /login/access-token\nPassword recovery & reset"]
            %% KIND: router
            users_routes["Users Routes\n[Component: APIRouter]\nGET·POST·PATCH·DELETE /users\nSignup · profile · admin ops"]
            %% KIND: router
            items_routes["Items Routes\n[Component: APIRouter]\nGET·POST·PUT·DELETE /items\nOwner-scoped item management"]
            %% KIND: router
            utils_routes["Utils Routes\n[Component: APIRouter]\nGET /health-check\nPOST /test-email"]
        end

        %% KIND: boundary
        subgraph services["Service Layer"]
            %% KIND: service_layer
            crud_layer["CRUD Operations\n[Component: crud.py]\ncreate_user · authenticate\ncreate_item · update_user"]
            %% KIND: service_layer
            email_service["Email Service\n[Component: utils.py]\nSMTP send · Jinja2 templates\nReset & welcome emails"]
            %% KIND: service_layer
            token_utils["Token Utilities\n[Component: utils.py]\nJWT generation & verification\nfor password-reset flow"]
        end

        %% KIND: boundary
        subgraph data_access["Data Access Layer"]
            %% KIND: data_access
            models_layer["SQLModel Models\n[Component: models.py]\nUser & Item tables\nPydantic request/response schemas"]
            %% KIND: data_access
            db_engine["DB Engine\n[Component: core/db.py]\nSQLAlchemy engine\nConnection pooling"]
            %% KIND: storage
            alembic_mig["Alembic Migrations\n[Component: alembic/]\nSchema version control\nUUID · cascade · timestamps"]
        end

        %% KIND: service_layer
        core_config["Core Config\n[Component: core/config.py]\nPydantic Settings\nEnvironment-based configuration"]

    end

    pg[("PostgreSQL\n[Container]")]
    smtp["SMTP Service\n[External]"]
    sentry["Sentry\n[External]"]

    react_spa -->|"HTTP requests"| cors_mw
    cors_mw --> routes

    login_routes --> auth_dep
    login_routes --> crud_layer
    login_routes --> email_service
    login_routes --> token_utils
    users_routes --> auth_dep
    users_routes --> superuser_dep
    users_routes --> crud_layer
    items_routes --> auth_dep
    items_routes --> crud_layer
    utils_routes --> superuser_dep
    utils_routes --> email_service

    auth_dep --> db_dep
    crud_layer --> db_dep
    crud_layer --> models_layer
    db_dep --> db_engine
    db_engine -->|"SQL / psycopg"| pg
    alembic_mig -->|"DDL migrations"| pg
    email_service -->|"SMTP"| smtp
    core_config -.->|"configures"| sentry

    classDef component fill:#4b8bc8,stroke:#2b6ca3,color:#fff
    classDef container fill:#1168bd,stroke:#0b4884,color:#fff
    classDef db fill:#2b78c4,stroke:#1a5276,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff

    class cors_mw,auth_dep,db_dep,superuser_dep component
    class login_routes,users_routes,items_routes,utils_routes component
    class crud_layer,email_service,token_utils component
    class models_layer,db_engine,alembic_mig component
    class core_config component
    class react_spa container
    class pg db
    class smtp,sentry ext
```

---

## L3: React SPA

```mermaid
graph TB
    %% SCOPE: urn:c4:container:react_spa

    user["End User\n[Person]"]
    admin_user["Admin User\n[Person]"]

    subgraph react_spa["React SPA"]

        %% KIND: router
        routing["TanStack Router\n[Component: @tanstack/react-router]\nFile-based routing\nLayout nesting & route guards"]

        %% KIND: boundary
        subgraph pages["Pages"]
            %% KIND: router
            dashboard_page["Dashboard\n[Component: routes/_layout/index.tsx]\nWelcome view"]
            %% KIND: router
            login_page["Login\n[Component: routes/login.tsx]\nEmail & password authentication"]
            %% KIND: router
            signup_page["Signup\n[Component: routes/signup.tsx]\nNew user registration"]
            %% KIND: router
            items_page["Items\n[Component: routes/_layout/items.tsx]\nItem listing & CRUD"]
            %% KIND: router
            admin_page["Admin\n[Component: routes/_layout/admin.tsx]\nUser management (superuser)"]
            %% KIND: router
            settings_page["Settings\n[Component: routes/_layout/settings.tsx]\nProfile & password management"]
            %% KIND: router
            recovery_pages["Password Recovery\n[Component: routes/recover-password.tsx]\nRecover & reset flows"]
        end

        %% KIND: boundary
        subgraph features["Feature Components"]
            %% KIND: service_layer
            items_feat["Items Components\n[Component: components/Items/]\nAddItem · EditItem · DeleteItem\nItemActionsMenu · columns"]
            %% KIND: service_layer
            admin_feat["Admin Components\n[Component: components/Admin/]\nAddUser · EditUser · DeleteUser\nUserActionsMenu · columns"]
            %% KIND: service_layer
            settings_feat["Settings Components\n[Component: components/UserSettings/]\nUserInformation · ChangePassword\nDeleteAccount · DeleteConfirmation"]
        end

        %% KIND: boundary
        subgraph common_ui["Common UI"]
            %% KIND: service_layer
            sidebar_ui["Sidebar\n[Component: components/Sidebar/]\nAppSidebar · navigation\nUser menu"]
            %% KIND: service_layer
            data_table["Data Table\n[Component: components/Common/DataTable.tsx]\nReusable table\nTanStack Table integration"]
            %% KIND: service_layer
            ui_primitives["UI Primitives\n[Component: components/ui/]\nshadcn/ui + Radix primitives\nButtons · dialogs · forms · etc."]
        end

        %% KIND: boundary
        subgraph services_layer["Services & State"]
            %% KIND: integration
            api_client["OpenAPI Client\n[Component: client/]\nAuto-generated TypeScript client\nfor all FastAPI endpoints"]
            %% KIND: service_layer
            auth_hook["Auth Hook\n[Component: hooks/useAuth.ts]\nLogin · logout · token mgmt\nlocalStorage persistence"]
            %% KIND: service_layer
            theme_provider["Theme Provider\n[Component: theme-provider.tsx]\nDark / light mode\nvia next-themes"]
        end

    end

    fastapi_api["FastAPI API\n[Container]"]

    user -->|"Browses application"| routing
    admin_user -->|"Manages users"| routing
    routing --> pages

    items_page --> items_feat
    admin_page --> admin_feat
    settings_page --> settings_feat

    items_feat --> data_table
    admin_feat --> data_table
    items_feat --> ui_primitives
    admin_feat --> ui_primitives
    settings_feat --> ui_primitives
    pages --> sidebar_ui

    items_feat --> api_client
    admin_feat --> api_client
    settings_feat --> api_client
    login_page --> auth_hook
    signup_page --> api_client
    recovery_pages --> api_client
    auth_hook --> api_client

    api_client -->|"REST API / JSON"| fastapi_api

    classDef component fill:#4b8bc8,stroke:#2b6ca3,color:#fff
    classDef container fill:#1168bd,stroke:#0b4884,color:#fff
    classDef person fill:#08427b,stroke:#052e56,color:#fff

    class routing component
    class dashboard_page,login_page,signup_page,items_page,admin_page,settings_page,recovery_pages component
    class items_feat,admin_feat,settings_feat component
    class sidebar_ui,data_table,ui_primitives component
    class api_client,auth_hook,theme_provider component
    class fastapi_api container
    class user,admin_user person
```

---

## Containment Map

```text
%% ═══════════════════════════════════════════════════════════════
%% CONTAINMENT MAP — Full Stack FastAPI Project
%% Covers L1, L2, and all L3 diagrams.
%% Every subgraph and its children are listed below.
%% ═══════════════════════════════════════════════════════════════

%% ── L1 top-level groups ──────────────────────────────────────────
users              CONTAINS [user, admin_user]
fullstack_boundary CONTAINS [fullstack_platform, react_spa, fastapi_api, pg]
external           CONTAINS [smtp, sentry]

%% ── L2→L3 FastAPI API ───────────────────────────────────────────
fastapi_api CONTAINS [middleware, routes, services, data_access, core_config]
  middleware   CONTAINS [cors_mw, auth_dep, db_dep, superuser_dep]
  routes       CONTAINS [login_routes, users_routes, items_routes, utils_routes]
  services     CONTAINS [crud_layer, email_service, token_utils]
  data_access  CONTAINS [models_layer, db_engine, alembic_mig]

%% ── L2→L3 React SPA ─────────────────────────────────────────────
react_spa CONTAINS [routing, pages, features, common_ui, services_layer]
  pages          CONTAINS [dashboard_page, login_page, signup_page, items_page, admin_page, settings_page, recovery_pages]
  features       CONTAINS [items_feat, admin_feat, settings_feat]
  common_ui      CONTAINS [sidebar_ui, data_table, ui_primitives]
  services_layer CONTAINS [api_client, auth_hook, theme_provider]
```
