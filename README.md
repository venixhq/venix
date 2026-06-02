<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.png">
  <img src="assets/logo-light.png" alt="Venix" width="620">
</picture>

### The headless, multi-tenant commerce backend: one API key, a full production store engine. Built for developers, technical founders, and agencies who need a real backend, not a CMS.

[![Python](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)](https://postgresql.org)
[![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-2.0-D71F00?logo=sqlalchemy&logoColor=white)](https://sqlalchemy.org)
[![Redis](https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=white)](https://redis.io)
[![Celery](https://img.shields.io/badge/Celery-37814A?logo=celery&logoColor=white)](https://docs.celeryq.dev)
[![Stripe](https://img.shields.io/badge/Stripe-635BFF?logo=stripe&logoColor=white)](https://stripe.com)
[![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)](https://docker.com)
[![CI](https://github.com/venixhq/venix/actions/workflows/ci.yml/badge.svg)](https://github.com/venixhq/venix/actions)

**[Live API](https://venix.website)** · **[Interactive Docs](https://venix.website/docs)** · **[Health](https://venix.website/health)**

<sub>⏳ The live demo runs on a free tier and spins down when idle, so the first request may take ~50 s to wake.</sub>

</div>

---

## What Venix Is

Venix is a headless, multi-tenant commerce backend, effectively a store engine sold as an API-only service. A store signs up, receives one API key, and immediately has a complete, isolated production backend: authentication, catalog, cart, orders, payments, and admin. No servers to provision, no database to manage, no background workers to babysit.

Tenants bring their own frontend (React, Next.js, a mobile app, a no-code builder, anything), hosted wherever they like. Venix owns nothing on the frontend side and imposes no template. It owns the hard part: correctness, concurrency, security, and reliability behind every endpoint.

Every tenant is fully isolated. One store can never read, write, or affect another store's data, and that guarantee is enforced by the platform itself, not by careful coding.

## Why Venix

| | Easy but rigid | **Venix** | Flexible but DIY |
|---|---|---|---|
| | Shopify · WooCommerce | API-first commerce backend | Self-hosted from scratch |
| **Frontend** | Locked to their themes | Bring your own, anywhere | Yours, but you build everything |
| **Backend** | Hidden, not yours | Production-grade, hosted for you | You build, host, and operate it |
| **Infra** | Managed | Managed | Postgres, Redis, workers: all on you |
| **Time to first call** | Fast | One API key | Weeks of plumbing |

Venix fills the gap: the flexibility of a custom backend with none of the infrastructure burden.

## Current Status

> **Multi-tenancy is in its final stages on the `feat/multi-tenancy` branch.** The tenant model, isolation layer, and onboarding are implemented and working in code; the public deployment on `main` currently runs the single-tenant engine, pending the imminent coordinated multi-tenant merge.

---

## Architecture

Venix rests on two engineering differentiators. They are explained below primarily *through the diagrams*. Each one is traced directly from the code that implements it.

### 1 · Bridge Isolation Model

Row-level multi-tenant isolation enforced **automatically at the SQLAlchemy session layer**, not by developer discipline. A `do_orm_execute` event listener injects a `tenant_id` filter into *every* `SELECT` for tenant-scoped models, so a query physically cannot return another tenant's rows. A designed path to dedicated-database "silos" exists for a future enterprise tier.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0F1419','primaryTextColor':'#F2EFE6','primaryBorderColor':'#5BBA6F','lineColor':'#5BBA6F','tertiaryColor':'#F2EFE6','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
flowchart TB
    subgraph REQ[" Incoming requests "]
        direction LR
        TA["Tenant A<br/>API key / JWT"]:::ext
        TB2["Tenant B<br/>API key / JWT"]:::ext
        TC["Tenant C<br/>API key / JWT"]:::ext
    end

    subgraph APP[" Single Venix app instance "]
        direction TB
        Resolve["Tenant resolver<br/>sets request.state.tenant"]:::accent
        GetDB["get_db()<br/>db.info['tenant_id'] = tenant.id"]:::accent
        Listener["do_orm_execute listener<br/>injects WHERE tenant_id = :id<br/>on every SELECT"]:::accent
        Writes["Service writes<br/>tenant_id passed explicitly"]:::core
    end

    SharedPG[("Shared PostgreSQL<br/>every row tagged by tenant_id<br/><b>Pool model, shipping now</b>")]:::store

    subgraph SILO[" Silo model: planned enterprise tier "]
        direction LR
        DBA[("Dedicated tenant DB")]:::future
        DBB[("Dedicated tenant DB")]:::future
    end

    TA --> Resolve
    TB2 --> Resolve
    TC --> Resolve
    Resolve --> GetDB --> Listener
    Listener -->|"auto-scoped SELECT"| SharedPG
    Writes -->|"INSERT with tenant_id"| SharedPG
    GetDB -.->|"if tenant.db_url is set (future)"| SILO

    classDef core fill:#0F1419,stroke:#5BBA6F,color:#F2EFE6;
    classDef accent fill:#5BBA6F,stroke:#0F1419,color:#0F1419;
    classDef ext fill:#F2EFE6,stroke:#0F1419,color:#0F1419;
    classDef store fill:#1E2630,stroke:#5BBA6F,color:#F2EFE6;
    classDef future fill:#F2EFE6,stroke:#9AA0A6,color:#5F6368,stroke-dasharray:4 4;
```

<sub><b>Reads</b> are auto-scoped by the session listener; <b>writes</b> carry <code>tenant_id</code> explicitly. One instance, one shared database, hard per-tenant isolation, with a dashed path to dedicated databases on demand.</sub>

### 2 · Zero-Infrastructure Onboarding

One API key resolves to a full backend. A single middleware is the sole enforcement point: it hashes the key, resolves the tenant (cache-first, database on miss), and rejects anything unresolved, inactive, or abusive, all before a request ever touches a route.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0F1419','primaryTextColor':'#F2EFE6','primaryBorderColor':'#5BBA6F','lineColor':'#0F1419','actorBkg':'#0F1419','actorBorder':'#5BBA6F','actorTextColor':'#F2EFE6','signalColor':'#0F1419','signalTextColor':'#0F1419','noteBkgColor':'#5BBA6F','noteTextColor':'#0F1419','noteBorderColor':'#0F1419','labelBoxBkgColor':'#F2EFE6','labelBoxBorderColor':'#5BBA6F','labelTextColor':'#0F1419','loopTextColor':'#0F1419','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
sequenceDiagram
    autonumber
    participant C as Client
    participant M as Tenant Resolver
    participant R as Redis
    participant DB as PostgreSQL

    C->>M: Request + X-Tenant-API-Key (or Bearer JWT)
    Note over M: Bypass: /health, /tenants/register,<br/>/docs, /redoc, /openapi.json, /webhooks/stripe/*
    M->>R: GET fail-counter for client IP
    alt >= 10 failures / minute
        M-->>C: 429 Too many failed attempts
    end
    M->>M: SHA-256(api_key)
    M->>R: GET tenant:apikey:{hash}
    alt cache hit
        R-->>M: tenant
    else cache miss
        M->>DB: SELECT tenant WHERE api_key_hash = :hash
        alt found
            M->>R: SET cache (TTL 300 s)
        else not found
            M->>R: INCR IP fail-counter
            M-->>C: 401 Invalid API Key
        end
    end
    Note over M: JWT path mirrors this via the<br/>tenant_id claim, resolved to tenant:id:{id}
    alt key tenant != JWT tenant
        M-->>C: 403 Tenant mismatch
    else tenant inactive
        M-->>C: 403 Tenant is deactivated
    end
    M->>C: attach request.state.tenant, then continue
```

<sub>Every Redis call degrades gracefully: on a cache outage the resolver logs a warning and falls through to PostgreSQL. The platform stays correct, just slower.</sub>

### System at a glance

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0F1419','primaryTextColor':'#F2EFE6','primaryBorderColor':'#5BBA6F','lineColor':'#5BBA6F','tertiaryColor':'#F2EFE6','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
flowchart LR
    Client(["Client<br/>any frontend"]):::ext

    subgraph APP[" Venix app instance (one deployment) "]
        direction TB
        MW["Middleware chain<br/>CORS · logging · request-ID · tenant resolver"]:::accent
        Router["Routers → Schemas<br/>HTTP contract · Pydantic validation"]:::core
        Service["Services<br/>business logic · authz · transactions"]:::core
        Model["Models<br/>async SQLAlchemy 2.0 + asyncpg"]:::core
        Worker["Celery worker"]:::core
        Beat["Celery beat<br/>reconciliation scheduler"]:::core
    end

    PG[("PostgreSQL<br/>tenant-scoped rows")]:::store
    Redis[("Redis<br/>tenant cache · rate limit · cache · broker")]:::store
    Stripe(["Stripe"]):::ext
    Resend(["Resend"]):::ext

    Client -->|HTTPS| MW
    MW -->|"resolve tenant"| Redis
    MW -.->|"on cache miss"| PG
    MW --> Router --> Service --> Model --> PG
    Service -->|"cache-aside"| Redis
    Service <-->|"checkout · refund"| Stripe
    Stripe -->|"signed webhook"| Router
    Redis -->|broker| Worker
    Beat -->|"every 15 min"| Redis
    Worker <-->|"reconcile orders"| Stripe
    Worker -->|"emails"| Resend
    Worker --> PG

    classDef core fill:#0F1419,stroke:#5BBA6F,color:#F2EFE6;
    classDef accent fill:#5BBA6F,stroke:#0F1419,color:#0F1419;
    classDef ext fill:#F2EFE6,stroke:#0F1419,color:#0F1419;
    classDef store fill:#1E2630,stroke:#5BBA6F,color:#F2EFE6;
```

<sub>Strict layering: routers never touch the database, services own all business logic and authorization, Redis is reached only from middleware and services. External dependencies (Redis, Celery, Stripe) are reliability layers, never correctness dependencies.</sub>

---

## Key Flows

### Atomic checkout

`SELECT FOR UPDATE` locks each product row before stock is read, so two concurrent buyers of the last unit can never both succeed. For Cash-on-Delivery the order, stock decrement, inventory log, and cart clear commit in **one transaction**; for Stripe the order is created, a Checkout Session is issued, and that same atomic commit happens when the signature-verified webhook confirms payment.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0F1419','primaryTextColor':'#F2EFE6','primaryBorderColor':'#5BBA6F','lineColor':'#0F1419','actorBkg':'#0F1419','actorBorder':'#5BBA6F','actorTextColor':'#F2EFE6','signalColor':'#0F1419','signalTextColor':'#0F1419','noteBkgColor':'#5BBA6F','noteTextColor':'#0F1419','noteBorderColor':'#0F1419','labelBoxBkgColor':'#F2EFE6','labelBoxBorderColor':'#5BBA6F','labelTextColor':'#0F1419','loopTextColor':'#0F1419','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
sequenceDiagram
    autonumber
    participant C as Customer
    participant CO as CheckoutService
    participant DB as PostgreSQL
    participant S as Stripe
    participant WB as Webhook + Beat

    C->>CO: POST /orders (address, payment method)
    CO->>DB: validate address + cart (400 empty / 409 stock)
    CO->>DB: INSERT order (PENDING, UNPAID)

    alt Cash on Delivery
        CO->>DB: SELECT product FOR UPDATE
        Note over CO,DB: one transaction: order items + price<br/>snapshot + stock decrement + inventory<br/>log + clear cart → COMMIT
        CO-->>C: 201 order confirmed
    else Stripe
        CO->>S: create Checkout Session (idempotency key)
        S-->>CO: session id + url
        CO->>DB: COMMIT (order holds session id)
        CO-->>C: 201 + checkout_url
        C->>S: completes payment
        S->>WB: checkout.session.completed (signed)
        Note over WB: verify signature → 400 if invalid<br/>dedup via processed_webhook_events
        WB->>DB: SELECT order FOR UPDATE
        Note over WB,DB: same atomic transaction →<br/>stock decrement + clear cart →<br/>PAID + CONFIRMED → COMMIT
    end

    Note over WB,S: Reconciliation beat (every 15 min): UNPAID Stripe<br/>orders older than 30 min → poll Stripe → recover a<br/>lost webhook, or mark EXPIRED + CANCELLED
```

### Auto-refund saga

If stock is exhausted between checkout and payment confirmation, Venix can't fulfil the order, so it runs a **compensating transaction**: refund the charge through Stripe, then mark the order refunded and cancelled.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0F1419','primaryTextColor':'#F2EFE6','primaryBorderColor':'#5BBA6F','lineColor':'#0F1419','actorBkg':'#0F1419','actorBorder':'#5BBA6F','actorTextColor':'#F2EFE6','signalColor':'#0F1419','signalTextColor':'#0F1419','noteBkgColor':'#5BBA6F','noteTextColor':'#0F1419','noteBorderColor':'#0F1419','labelBoxBkgColor':'#F2EFE6','labelBoxBorderColor':'#5BBA6F','labelTextColor':'#0F1419','loopTextColor':'#0F1419','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
sequenceDiagram
    autonumber
    participant S as Stripe
    participant W as WebhookService
    participant DB as PostgreSQL

    S->>W: checkout.session.completed (paid)
    W->>DB: SELECT order FOR UPDATE
    alt already PAID
        W-->>S: ack (idempotent no-op)
    else stock still available
        W->>DB: decrement stock + items + clear cart
        W->>DB: order → PAID + CONFIRMED
    else stock gone since checkout
        Note over W,S: compensating action
        W->>S: Refund.create(payment_intent)
        W->>DB: order → REFUNDED + CANCELLED
    end
    W->>DB: record processed event → COMMIT
```

### Order status lifecycle

Orders move through a strict finite-state machine. The row is locked before any transition is validated, and skipping or reversing a state is rejected with `409 Conflict`.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#5BBA6F','primaryTextColor':'#0F1419','primaryBorderColor':'#0F1419','lineColor':'#0F1419','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
stateDiagram-v2
    direction LR
    [*] --> PENDING: checkout
    PENDING --> CONFIRMED: admin advance · payment confirmed
    CONFIRMED --> SHIPPED: admin advance
    SHIPPED --> COMPLETED: admin advance
    PENDING --> CANCELLED: customer/admin cancel · auto-refund · expired
    CONFIRMED --> CANCELLED: admin cancel
    COMPLETED --> [*]
    CANCELLED --> [*]
    note right of COMPLETED: terminal
    note right of CANCELLED: terminal, stock restored, change audited
```

---

## Engineering Highlights

> Why this is production-grade: every claim below maps to code on the branch.

| | |
|---|---|
| 🔐 **Automatic tenant isolation** | A `do_orm_execute` listener adds a `tenant_id` filter to every `SELECT` on tenant-scoped models. Isolation is a property of the session layer, not of individual queries. |
| 🔑 **One key, full backend** | A single resolver middleware authenticates the API key (or JWT `tenant_id` claim), enforces active status, and guards against IP brute-force. It is the sole entry gate for every request. |
| ⚛️ **Atomic checkout** | Order creation, stock decrement, inventory log, and cart clear commit in one transaction. Any failure rolls everything back: no partial orders, no phantom stock. |
| 🔒 **Concurrency-safe by construction** | `SELECT FOR UPDATE` locks product rows before reading stock; cancellations lock in deterministic order to avoid deadlocks. |
| 💳 **Stripe with reliability guarantees** | Signature-verified, idempotent webhooks (dedup table), an auto-refund saga when stock runs out, and a reconciliation beat that recovers lost webhooks and sweeps stale orders. |
| 🔄 **Token rotation with reuse detection** | Refresh tokens are SHA-256 hashed and rotated on every use; presenting a revoked token is rejected. |
| 💰 **Price snapshots at purchase** | Order items capture the price at checkout, so later price changes never rewrite order history. |
| 📋 **Fully audited inventory** | Every stock change is logged with a typed reason; stock is never mutated silently. |
| ⚙️ **Full async data layer** | One event loop end to end: async routes, async SQLAlchemy 2.0 (asyncpg), async Redis. |
| 🧪 **A test suite engineered for speed** | **500+ tests run in ~16 s**: savepoint-based isolation, parallel execution via `pytest-xdist`, passwords pre-hashed once at module load. |
| 🖥️ **Structured, traceable logs** | Every request emits JSON with a unique request ID, status, duration, and client IP. Stdout-only in production (12-Factor). |
| 🩺 **Real readiness checks** | `/health` pings PostgreSQL, Redis, and the Celery broker, returning `503` if any dependency is down. |

---

## Features

Capabilities, not an endpoint inventory. The full, always-current API contract lives in the **[interactive Swagger docs](https://venix.website/docs)**.

| Domain | Capabilities |
|---|---|
| 🏢 **Tenancy** *(the product)* | Tenant self-registration with a one-time `vnx_` API key · API-key rotation & revocation · email verification · profile management · two-step password change · self-deactivation |
| 👤 **Auth & Identity** | Email registration with verification codes · JWT access + refresh with rotation & reuse rejection · password reset · single / all-device logout · profile editing · `CUSTOMER` / `ADMIN` RBAC |
| 🛍️ **Catalog** | Product browsing with category & price filters and pagination · product detail · category listing (Redis-cached) |
| 🛒 **Cart & Addresses** | Add / update / remove / clear cart · multiple delivery addresses with a default flag · ownership enforced |
| 📦 **Orders & Checkout** | Cash-on-Delivery & Stripe checkout · atomic, concurrency-safe order placement · reuse-if-valid Stripe sessions · paginated order history · customer cancellation |
| 💳 **Payments** | Stripe Checkout Sessions · signature-verified idempotent webhooks · auto-refund saga · scheduled reconciliation |
| 🔧 **Admin** | Full product & category CRUD · order status FSM & cancellation · user management (list, view, activate/deactivate, role changes) |
| 🛡️ **Platform** | Redis-backed rate limiting (multi-worker safe) · structured request logging · dependency health checks · async task queue with scheduled jobs |

---

## Auth System

Not a tutorial JWT setup. Every edge case is handled, and tokens are now tenant-aware.

| Capability | Detail |
|---|---|
| Registration | Email + password, validated with Pydantic v2 and `phonenumbers` (E.164) |
| Email verification | 6-digit code with a 10-minute expiry |
| Login | Short-lived access token + long-lived refresh token |
| Tenant-aware tokens | JWTs carry a `tenant_id` claim, used by the resolver for browser/dashboard clients |
| Token rotation | The old refresh token is revoked on every refresh; reuse is rejected |
| Token storage | Refresh tokens stored as SHA-256 hashes; plaintext never persists |
| Password change | Two-step, confirmation-code based |
| Password reset | Time-limited, single-use token |
| Logout | Single device or all devices at once |
| Account deactivation | Soft delete; all sessions revoked |
| RBAC | `CUSTOMER` and `ADMIN` enforced via FastAPI dependency injection |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | FastAPI + Uvicorn / Gunicorn |
| Database | PostgreSQL · async SQLAlchemy 2.0 (asyncpg) · Alembic migrations |
| Cache & broker | Redis: cache-aside with write-through invalidation; also the Celery broker & result backend |
| Task queue | Celery + Redis · scheduled jobs via Celery Beat · at-least-once delivery, JSON serialization |
| Payments | Stripe Checkout Sessions · signature-verified webhooks · Stripe Refund API |
| Auth | python-jose (JWT) · passlib (bcrypt) · SHA-256 token hashing |
| Validation | Pydantic v2 · email-validator · phonenumbers (E.164) |
| Rate limiting | SlowAPI, Redis-backed and multi-worker safe |
| Email | Resend, dispatched through Celery tasks |
| Identifiers | UUIDv7 tenant keys · integer domain keys |
| Logging | Structured JSON · request-ID tracing |
| Tooling | Docker · docker-compose · pytest + pytest-asyncio + httpx + pytest-xdist · Ruff |
| Delivery | GitHub Actions CI · Render (web + worker + beat co-located) |

---

## Get Started

Venix is **API-first**. You don't deploy it; you call it.

| Step | What happens |
|---|---|
| **1 · Sign up** | Register a tenant and receive one `vnx_` API key (shown exactly once) |
| **2 · Authenticate** | Send the key as the `X-Tenant-API-Key` header; your isolated backend is live |
| **3 · Call the API** | Catalog, cart, orders, payments, and admin all respond, scoped to you |
| **4 · Build any frontend** | Wire it to your store, mobile app, or no-code builder, hosted anywhere |

The fastest way to explore the surface is the **[live interactive docs](https://venix.website/docs)**: every endpoint, schema, and status code, always current.

The live API currently runs the single-tenant engine, so the catalog is open and no key is needed yet. Try it:

```bash
curl "https://venix.website/products?limit=5&min_price=10"
```

<details>
<summary><b>Run it locally</b>, for contributors and the curious</summary>

<br/>

**Option 1 · Docker (recommended, no local Postgres/Redis needed)**

```bash
git clone https://github.com/venixhq/venix.git
cd venix
docker-compose up --build
```

The app runs at `http://localhost:8000`. Migrations apply automatically on startup, and the Celery **worker** and **beat** services start alongside the API.

**Option 2 · Local**

> Requires PostgreSQL and Redis running locally.

```bash
git clone https://github.com/venixhq/venix.git
cd venix
cp .env.example .env        # fill in DATABASE_URL, SECRET_KEY, REDIS_URL, MAIL_*
pip install -r requirements.txt
alembic upgrade head
uvicorn main:app --reload
```

Run the Celery worker separately when not using Docker:

```bash
celery -A core.celery_app worker --loglevel=info
```

> Tests don't need a running worker; `CELERY_TASK_ALWAYS_EAGER=True` runs tasks inline.

**Seed an admin user** (single-tenant runs)

```bash
# Linux / macOS
SEED_ADMIN_EMAIL=admin@example.com \
SEED_ADMIN_PASSWORD=yourpassword \
SEED_ADMIN_FIRST_NAME=Admin \
SEED_ADMIN_LAST_NAME=User \
python -m scripts.seed_admin
```

```powershell
# Windows PowerShell
$env:SEED_ADMIN_EMAIL="admin@example.com"
$env:SEED_ADMIN_PASSWORD="yourpassword"
$env:SEED_ADMIN_FIRST_NAME="Admin"
$env:SEED_ADMIN_LAST_NAME="User"
python -m scripts.seed_admin
```

> On the multi-tenant branch, the store admin is provisioned automatically when a tenant registers; manual seeding is only for the single-tenant engine.

</details>

---

## Data Model

Entities and key relationships: `tenants` is the root every domain table hangs from.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0F1419','primaryTextColor':'#F2EFE6','primaryBorderColor':'#5BBA6F','lineColor':'#5BBA6F','fontFamily':'ui-sans-serif, system-ui, sans-serif'}}}%%
erDiagram
    TENANTS ||--o{ USERS : owns
    TENANTS ||--o{ CATEGORIES : owns
    TENANTS ||--o{ PRODUCTS : owns
    TENANTS ||--o{ ORDERS : owns
    TENANTS ||--o{ CART_ITEMS : owns
    TENANTS ||--o{ ADDRESSES : owns
    TENANTS ||--o{ PROCESSED_WEBHOOK_EVENTS : owns
    USERS ||--o{ REFRESH_TOKENS : "has sessions"
    USERS ||--o{ ADDRESSES : "ships to"
    USERS ||--o{ CART_ITEMS : fills
    USERS ||--o{ ORDERS : places
    CATEGORIES ||--o{ PRODUCTS : groups
    PRODUCTS ||--o{ CART_ITEMS : "added as"
    PRODUCTS ||--o{ ORDER_ITEMS : "sold as"
    PRODUCTS ||--o{ INVENTORY_CHANGES : "audited by"
    ADDRESSES ||--o{ ORDERS : "delivered to"
    ORDERS ||--o{ ORDER_ITEMS : contains

    TENANTS {
        uuid id PK
        string slug UK
        string api_key_hash UK
    }
    USERS {
        int id PK
        uuid tenant_id FK
        string role "CUSTOMER / ADMIN"
    }
    PRODUCTS {
        int id PK
        uuid tenant_id FK
        int stock
    }
    ORDERS {
        int id PK
        uuid tenant_id FK
        enum status "FSM"
        enum payment_status
    }
    ORDER_ITEMS {
        int id PK
        decimal price_at_time "snapshot"
    }
    PROCESSED_WEBHOOK_EVENTS {
        uuid tenant_id PK
        string event_id PK
    }
```

---

## Roadmap

| Stage | Capabilities |
|---|---|
| ✅ **Shipped** | Single-tenant commerce engine: auth, catalog, cart, atomic checkout, orders, Stripe payments, admin, rate limiting, structured logging, health checks; deployed with CI |
| 🔄 **In progress** | **Multi-tenancy**: tenant onboarding & API keys, automatic per-tenant isolation, tenant-aware sessions & caching, per-tenant payment configuration, cross-tenant isolation test suite |
| 🧭 **Planned** | Coupons & promotions · product relationships (bundles, upsell) · reviews & wishlists · OAuth login · shipment tracking · product search · per-tenant analytics |

<details>
<summary>More on what's coming</summary>

<br/>

- **Payments & commerce**: coupons and promo codes, product relationships (bundles / alternatives) for frontend-built merchandising
- **Engagement & fulfillment**: OAuth login (per-tenant credentials), shipment & delivery tracking, wishlists, purchased-only reviews & ratings with moderation, in-app notifications
- **Search**: fast, typo-tolerant product search with an isolated index per tenant
- **Platform**: usage metrics and per-tenant dashboards, richer observability, hierarchical categories, SEO slugs
- **Enterprise isolation**: the silo model, a dedicated database per tenant, provisioned on demand

</details>

---

<div align="center">

**Built by [Anas Mohamed](https://github.com/anasmohamed05221)**

*Backend engineering. No shortcuts.*

</div>
