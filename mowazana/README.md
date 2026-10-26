<p align="center"><img src="assets/banner-en.jpg" alt="Mowazana: ZATCA e-invoicing for every tenant" width="100%"></p>

<p align="center">
  <b>English</b> · <a href="README.ar.md">العربية</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Laravel-10-00103e?style=flat-square&logo=laravel&logoColor=7ac943" alt="Laravel">
  <img src="https://img.shields.io/badge/PHP-8.2-00103e?style=flat-square&logo=php&logoColor=white" alt="PHP">
  <img src="https://img.shields.io/badge/React-18%20%2B%20Vite-00103e?style=flat-square&logo=react&logoColor=7ac943" alt="React">
  <img src="https://img.shields.io/badge/MySQL-RDS-00103e?style=flat-square&logo=mysql&logoColor=white" alt="MySQL">
  <img src="https://img.shields.io/badge/AWS-EC2%20%C2%B7%20RDS%20%C2%B7%20S3-00103e?style=flat-square&logo=amazonwebservices&logoColor=7ac943" alt="AWS">
  <img src="https://img.shields.io/badge/ZATCA-Phase%202-00103e?style=flat-square&logoColor=white" alt="ZATCA Phase 2">
  <img src="https://img.shields.io/badge/OpenAI-RAG-00103e?style=flat-square&logo=openai&logoColor=white" alt="OpenAI">
</p>

# Mowazana (موازنة): multi-tenant ERP and POS for Saudi businesses

**One platform, many businesses, every invoice signed for the tax authority.** A shop, a café or a trading company signs up, gets its own books, cashier, stock and VAT reports, and every sale is signed, chained and sent to ZATCA (the Saudi Zakat, Tax and Customs Authority) without the cashier waiting for it.

[app.mowazana-pos.com](https://app.mowazana-pos.com) · [mowazana.sa](https://mowazana.sa) · Web app + cashier app API · Arabic-first · **My role: sole engineer. I took an existing single-company ERP codebase and built the SaaS platform on top of it: multi-tenancy, ZATCA Phase 2, subscription billing, the POS API, the Rajih AI assistant, the redesigned UI, AWS infrastructure and CI/CD.**

> **The code belongs to the client and is private.** I'm happy to walk through the architecture, the schema or any decision below:
> **[HazemSalama108@gmail.com](mailto:HazemSalama108@gmail.com)** · [LinkedIn](https://linkedin.com/in/hazem-salama-290310310) · [hazemsalama.dev](https://hazemsalama.dev)

<p align="center">
  <img src="assets/dashboard.jpg" width="19%" alt="Dashboard">
  <img src="assets/pos-cashier.jpg" width="19%" alt="Point of sale">
  <img src="assets/trial-balance.jpg" width="19%" alt="Trial balance">
  <img src="assets/zatca-onboarding.jpg" width="19%" alt="ZATCA onboarding">
  <img src="assets/rajih-assistant.jpg" width="19%" alt="Rajih assistant">
</p>
<p align="center"><sub>Real screens from the staging environment (demo tenant, sample data): dashboard → point of sale → trial balance → ZATCA onboarding → Rajih assistant</sub></p>

---

## At a glance

| | |
|---|---|
| **Product** | Cloud ERP + POS sold as a subscription: one shared platform, each business a tenant with its own data, users, roles and plan |
| **Modules** | POS · sales · purchases · inventory · double-entry accounting · VAT and ZATCA · HR · e-commerce, switched on per plan |
| **Surfaces** | Tenant web app · super-admin portal · public sign-up · API for a Flutter cashier app (the app itself is built by another team) |
| **Backend** | Laravel 10 / PHP 8.2 · ~40k lines in `app/` · 401 API endpoints · 85 data models (71 tenant-scoped) · 129 migrations · 53 service classes |
| **Frontend** | React 18 + Vite · ~120k lines across 675 files · 155 routes · Arabic and English, RTL-first |
| **E-invoicing** | ZATCA Phase 2: onboarding (CSR, compliance and production certificates), UBL 2.1, signing, QR, reporting and clearance, through 8 dedicated services |
| **AI** | Rajih: Arabic support assistant answering from 68 product articles, with a grounding gate and an opt-in path to the tenant's own figures |
| **Quality** | 134 PHPUnit test files, with the money and VAT rules covered by unit tests |
| **Cloud** | AWS (EC2, RDS MySQL, S3) · GitHub Actions deploys from `main` and `staging` · systemd queue worker and scheduler |

---

## Architecture

### System

```mermaid
flowchart LR
    subgraph Clients
      WEB[Tenant web app<br/>React · ERP + POS]
      SA[Super-admin portal<br/>React]
      APP[Cashier app<br/>Flutter · other team]
      PUB[Public sign-up]
    end
    subgraph AWS
      NG[nginx · TLS]
      API[Laravel API<br/>JWT · TenantScope · module gate]
      QW[Queue worker<br/>zatca, default]
      SCH[Scheduler<br/>billing · dunning · retries · health]
      DB[(MySQL on RDS<br/>shared schema, tenant_id)]
      S3[(S3<br/>per-tenant prefix)]
    end
    ZATCA[ZATCA Fatoora<br/>compliance · reporting · clearance]
    TELR[Telr<br/>hosted payment page]
    AI[OpenAI<br/>embeddings · chat]
    MAIL[Transactional email]
    WEB & SA & APP & PUB --> NG --> API
    API --> DB & S3
    QW --> DB
    SCH --> DB
    QW <--> ZATCA
    API <--> TELR
    API <--> AI
    API & SCH --> MAIL
```

**A modular monolith on one shared database.** Every tenant lives in the same MySQL schema, separated by a `tenant_id` column and an Eloquent global scope that is bound from the signed token on each request. Sales, stock, ledger and VAT must change together in one transaction, and the original codebase was a single-company Laravel app, so one schema with a tenant column was the shortest safe path to SaaS. Slow or external work (ZATCA submission, billing runs, reminders) runs on the queue and the scheduler, on the same codebase.

### Network and deployment

```mermaid
flowchart TB
    U[Users and cashier app] --> NG[nginx · TLS]
    subgraph EC2["EC2 application server"]
      NG --> PROD[Production<br/>PHP-FPM · Laravel · React build]
      NG --> STG[Staging<br/>basic auth · noindex]
      QW[systemd: queue worker<br/>--queue=zatca,default]
      TM[systemd timer<br/>scheduler every minute]
    end
    subgraph RDS["RDS MySQL"]
      PDB[(production database)]
      SDB[(staging database<br/>own user, own grants)]
    end
    PROD --> PDB
    STG --> SDB
    QW --> PDB
    TM --> PDB
    PROD --> S3[(S3)]
    GH[GitHub Actions] -->|main| PROD
    GH -->|staging| STG
```

- **Branch decides the target.** A push to `staging` deploys the staging copy and runs migrations; a push to `main` deploys production, where migrations run only when explicitly requested.
- **Each deploy:** build artifact, upload, extract, cache config and routes, reload PHP-FPM, restart the queue worker, refresh the assistant's knowledge base, then a health check against the live API.
- **Unit files live in the repo** (`infra/systemd/`) with an idempotent install script, so the worker's queue list cannot drift on the server.

### Tenant life cycle

```mermaid
stateDiagram-v2
    [*] --> Applied: public sign-up
    Applied --> Reviewing
    Reviewing --> NeedsInfo
    NeedsInfo --> Reviewing
    Reviewing --> Rejected
    Reviewing --> Active: super admin approves<br/>seed roles, chart of accounts, VAT, admin
    Active --> Active: trial row (0 SAR), then paid periods
    Active --> Locked: invoice 3 days overdue
    Locked --> Active: invoice paid
    Locked --> Wiped: 180 days overdue
    Rejected --> [*]
    Wiped --> [*]
```

A locked tenant can still sign in and read everything (its books, invoices and reports). Only writes are refused. Reminders escalate at days 3, 15, 30, 45, 90, 120, 150 and 179 before the wipe on day 180, and each one fires once.

### One invoice through ZATCA, without forking the hash chain

```mermaid
sequenceDiagram
    participant C as Cashier / web app
    participant API as Laravel API
    participant Q as zatca queue
    participant DB as MySQL
    participant Z as ZATCA
    C->>API: create sale
    API->>DB: invoice + ledger + stock (one transaction)
    API-->>C: done, status pending
    API->>Q: SubmitInvoiceToZatca(invoice, tenant)
    Q->>DB: BEGIN · lock tenant's credential row
    Q->>DB: next ICV, previous invoice hash (PIH)
    Q->>Q: build UBL 2.1 · sign · Phase 2 QR
    alt simplified (B2C)
      Q->>Z: reporting
    else standard (B2B)
      Q->>Z: clearance
      Z-->>Q: counter-signed XML (stored as the legal copy)
    end
    alt reported / cleared
      Q->>DB: PIH = this invoice's hash · COMMIT
    else error
      Q->>DB: ROLLBACK (ICV and PIH released) · mark failed
      Note over Q,DB: job retries after 30 s, 2 min, 10 min<br/>a sweep re-queues failures every 15 min
    end
```

---

## Architecture decisions

| # | Decision | Instead of | Why | Trade-off accepted |
|---|---|---|---|---|
| 1 | **Shared schema with `tenant_id` and a global Eloquent scope** | A database per tenant | Retrofitting a single-company app: one schema to migrate, one place to add tenancy, small tenants | Isolation depends on queries going through the models; raw queries must filter by hand |
| 2 | **Tenant id inside the signed JWT, bound per request** | A subdomain per tenant | One app URL for every business, and the same API serves the cashier app | The token carries the tenant, so switching business means signing in again |
| 3 | **Overdue tenants are locked read-only in middleware** | Blocking sign-in | A business that is late on its subscription still needs its invoices and books | Every client has to handle a 403 on writes with a clear message |
| 4 | **Plan modules enforced on the server (`module:sales`)** | Hiding unpaid modules in the sidebar | Hidden menus do not stop direct API calls | Each route group carries its gate; an empty module list means unrestricted for older tenants |
| 5 | **Row lock on the tenant's ZATCA credential from ICV to PIH** | An optimistic counter | Two invoices reading the same previous hash would fork the chain, which ZATCA rejects | A tenant's submissions run one at a time, and the lock spans the HTTP call |
| 6 | **ZATCA submission on its own queue, with backoff and a retry sweep** | Submitting inside the checkout request | The cashier must not wait on the tax authority, and its outages must not stop sales | Invoices show "pending" for a moment, and a health check watches the queue every 5 minutes |
| 7 | **B2B goes to clearance, B2C to reporting; the counter-signed XML is kept** | One submission path for every invoice | That is the Phase 2 rule, and the cleared XML, not the local one, is the legal document | Two flows to test, and more stored XML per invoice |
| 8 | **A ZATCA library for signing, my own UBL builder, onboarding and submission** | A third-party e-invoicing middleman | No per-invoice fee and no extra party holding tenants' certificates | Keeping up with ZATCA spec changes is on us |
| 9 | **Money as integer halalas; VAT-inclusive lines derived from the gross** | Floats and net-first rounding | 400.00 typed at the till must print as 400.00, never 399.99 | For a few grosses the tax differs from round(net × 15%) by one halala, documented in code |
| 10 | **Reports computed live from ledger rows** | Stored running balances | Trial balance, income statement and balance sheet can never disagree with the journal | Each report is a query over the period, not a lookup |
| 11 | **Accounts resolved by name per tenant; product type picks the posting** | Hard-coded account ids | Each tenant has its own seeded chart of accounts; stock, service and cost lines post differently | A resolver call on every posting |
| 12 | **Telr: a fresh hosted order per click; the IPN only triggers a status check** | Trusting the IPN body | Hosted pages are single-use, and a callback can be forged or stale; the amount must also match | One extra API call per payment |
| 13 | **Subscription as a chain of periods with a wallet** | A status flag and a next-billing date | Trials, plan changes mid-period and credit for unused days all become plain rows | More logic to compute "where does this tenant stand" |
| 14 | **Rajih searches a packed float32 index in PHP** | A vector database | About two thousand chunks: a full scan is tens of milliseconds, with no extra service to run | Past a much larger corpus, a real index is needed |
| 15 | **Gulf dialect mapped to formal Arabic by a lexicon; both versions embedded** | Rewriting the question with an LLM | Measured: a dialect question scored 0.283 against its article, the formal version above 0.50, while off-topic questions reach 0.349 | A word list to maintain |
| 16 | **No grounding, no answer; tenant figures only through whitelisted metrics** | An open chatbot or text-to-SQL | A refused question becomes a support ticket; an invented answer becomes a wrong business decision | Some valid questions are refused until an article covers them |
| 17 | **Branch-driven deploys with a separate staging database** | Manual deploys to production | Every change is tested on staging before it reaches paying tenants | Staging shares the production host, so it is kept light (synchronous queue, local disk) |

---

## Deep dives

<details>
<summary><b>ZATCA Phase 2 end to end</b></summary>

- **Onboarding per tenant:** generate a CSR on secp256k1, exchange it with an OTP from the Fatoora portal for a compliance certificate, pass the six compliance invoices (standard and simplified × invoice, credit note, debit note), then request the production certificate.
- **Per invoice:** next ICV, previous invoice hash, UUID; UBL 2.1 XML; signature and the 9-tag Phase 2 QR; then reporting or clearance.
- **The chain never forks.** The tenant's credential row is locked for the whole step. On failure the transaction rolls back, so the ICV and PIH slot are released and the next invoice takes them.
- **Failures are visible.** The error and retry count are saved outside the rolled-back transaction, so the user sees what went wrong. A scheduled job re-queues failed invoices every 15 minutes, up to three tries.
- **Final means final.** Before a tenant is linked to ZATCA production, an invoice can be deleted and its ledger and stock entries unwound. After that, the only correction is a return.
</details>

<details>
<summary><b>Turning a single-company ERP into SaaS</b></summary>

- A `TenantScope` trait on 71 of the 85 models filters every read by the current tenant and stamps `tenant_id` on every insert. The few global tables (plans, applications, the assistant's knowledge base) are global on purpose.
- The authorization middleware decodes the token, binds the tenant for the request, applies the read-only rule for locked tenants and loads the role's permissions. Super admins have no tenant and sit outside the scope.
- Fixed account ids from the original code were replaced by name lookups per tenant, and payment lines must post to an account the tenant owns.
- **New tenant:** a public application, super-admin review, then one step that creates the tenant and seeds its roles and permissions, chart of accounts, payment methods, currency, VAT rates (15% and 0%) and root admin, and emails one-time credentials.
- **Wipe:** child tables first, then the tenant's S3 prefix, with a summary in the activity log.
</details>

<details>
<summary><b>Subscription billing and dunning</b></summary>

- Plans carry modules and seat caps (for example, how many cashiers). Super admins can also set a custom price per tenant.
- Cycles are monthly, quarterly, semi-annual or annual; annual is billed as ten months.
- Each period's invoice is issued on the day it starts. The wallet pays open invoices first, oldest first. A plan or cycle change mid-period can credit the unused part to the wallet, by days, months or used days.
- Card payments go through Telr's hosted page. The callback is verified, then the order is re-checked with Telr and the amount compared before the invoice is marked paid. Payment unlocks a locked tenant at once.
- The reminder ladder runs daily, and each step is recorded so it is never sent twice.
</details>

<details>
<summary><b>Rajih, the Arabic support assistant</b></summary>

- **Pipeline:** gate (key, tenant, switch, length, blocked topics, rate limits, monthly token cap) → support hand-off → optional data path → retrieve the top 6 chunks → grounding check → answer → audit row.
- **Knowledge base:** 68 Markdown articles, one `##` section per chunk, tables never split. Each deploy re-embeds only chunks whose SHA-256 changed, in one transaction. Super admins can add entries in the portal without a deploy.
- **Retrieval:** vectors are unit-normalised and packed into one binary blob, cached in memory and on disk, versioned by the corpus state. Similarity is a plain dot product.
- **Dialect:** questions in Gulf dialect («شلون», «وين», «ابغى») are mapped to the formal Arabic of the articles for retrieval only. The user's own words are what the model sees and what is logged.
- **Tenant figures (opt-in):** needs the switch, the tenant's consent and the user's permission for that metric. The model only picks one of 10 metric ids and fills in dates. It never writes SQL, and only the final number reaches it.
- **The team's loop:** a portal tab lists unanswered and disliked questions, a tester shows what Rajih would retrieve and the scores, and a calibration command helps set the threshold.
</details>

<details>
<summary><b>The API behind the cashier app</b></summary>

- A separate team builds the Flutter cashier app. I own the API it uses: login with its own rate limit, products, shifts, orders, purchases and receipts (A4 and 80 mm, both with the ZATCA QR).
- **The server is the authority on price.** The app sends the price charged and the list price it started from. A list price that does not match the product is rejected, so a price override can't be hidden from the tenant's admin.
- **Discounts follow the tenant's setting.** When discounts are off, the server refuses any line that still carries one.
- **Cashier seats follow the plan:** no limit, none, or at most N, and none at all without the POS module.
- The API specs are written to stand on their own, because the app team works from them without access to the backend.
</details>

---

## What I owned

- **Architecture and the multi-tenant data model**, retrofitted onto the original single-company codebase.
- **Backend:** tenancy, permissions and plan gating, sales, purchases, vouchers, ledger posting, VAT math, the full ZATCA Phase 2 integration, subscription billing, Telr payments and dunning.
- **Rajih:** retrieval, grounding, dialect handling, the metrics path, ingestion and the super-admin tooling.
- **Frontend:** the redesigned tenant app and POS, the super-admin portal and the sign-up flow, RTL-first in Arabic and English.
- **API for the cashier app**, with specs the other team builds against.
- **Cloud:** AWS EC2, RDS and S3, nginx, systemd units, the staging environment and the GitHub Actions pipelines.

---

<p align="center"><b>Want the deeper version: the tenancy model, the ZATCA pipeline, or how Rajih decides what it knows?</b><br>
<a href="mailto:HazemSalama108@gmail.com">HazemSalama108@gmail.com</a> · <a href="https://linkedin.com/in/hazem-salama-290310310">LinkedIn</a> · <a href="https://hazemsalama.dev">hazemsalama.dev</a></p>
