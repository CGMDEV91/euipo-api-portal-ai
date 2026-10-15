# Product overview — what the portal is and how customers use it

Functional picture of the EUIPO API Portal: business model, actors, the customer journey and where each custom module fits. Read it before designing anything user-facing, or when a task needs the "why" behind the portal.

Sources: `eap/` code, `eap/docs/`, the real portal content in `local-dev/fixtures/` (Getting started, Security, FAQ) and a browser walkthrough of the local portal (anonymous and as `developer`).
Module internals live in `.ai/skills/02`–`06`; this file only says where each module sits in the journey. A Spanish, more didactic version for humans is in `.ai/product-overview_ES.md`.

---

## 1. Business model

The portal is EUIPO's **self-service storefront for its APIs**. Companies, IP law firms and developers use it to integrate their systems with the EUIPO and manage trade marks and designs: search, file applications, read the mailbox, and so on.

- **Nothing is sold.** Sandbox and Production APIs are **free of charge** (FAQ).
- APIs are grouped into **products**. Each product has **plans**, and each plan carries **rate limits**.
- A customer **subscribes** an application to a plan. The subscription is a **permission EUIPO approves**, not a purchase.
- Policy: one subscription per API per organization; exceptions are reviewed case by case.

### Catalogue (11 products, each with a single API and a single "Default Plan")

| Product | Version | Tags | Purpose | Approval |
|---|---|---|---|---|
| Trademark search | 1.1.0 | Trade mark | Search EUIPO trade marks | Yes |
| Design search | 1.1.0 | Design | Search designs | Yes |
| EUTM Filing | 1.1.0 | Application, Trade mark | **File** EU trade mark applications | Yes |
| EUD Filing | 1.1.1 | Application, Design | **File** EU design applications | Yes |
| Goods And Services | 1.2.0 | Classification, Trade mark | TMClass taxonomy: search, validate, translate | Yes |
| Product Indications | 1.0.0 | Classification, Design | DSClass taxonomy | Yes |
| Me | 1.2.0 | Trade mark, Design, Appeal… | Manage your trade mark/design portfolio | Yes |
| My EUIPO mailbox | 1.2.1 | … Communication | User Area communication inbox | Yes |
| Document Repository | 1.0.0 | Attachment | Temporary document storage for attachments | Yes |
| Persons | 1.0.0 | Applicant, Representative | Search applicants and representatives | Yes |
| Out of Commerce Works | 1.2.0 | Out of commerce works | Metadata of out-of-commerce works | No |

---

## 2. Actors

```
                    ┌───────────────────────────┐
  EUIPO admin ─────►│  IBM API Connect          │  products, plans, rate limits,
                    │  (management plane)       │  subscription approval, client_id/secret
                    └──────┬─────────────┬──────┘
          syncs catalogue  │             │ configures
                           ▼             ▼
                 ┌──────────────┐   ┌─────────────────┐
  Developer ────►│ PORTAL       │   │ API Gateway     │◄── customer app calls
  (browser)      │ (Drupal/IBM) │   │ (enforcement)   │    api.euipo.europa.eu
                 └──────┬───────┘   └────────┬────────┘
   eap_credentials_sync │                    │ validates token
   registers the app    ▼                    ▼
                ┌────────────────────────────────┐
                │ EUIPO IdP (formerly CAS, OIDC) │  issues access tokens
                └────────────────────────────────┘
```

| Actor | Role | Under our control | Local replacement |
|---|---|---|---|
| IBM API Connect | Defines products/plans/limits, approves subscriptions, issues app credentials, pushes the catalogue to the portal | No | `mock-apic` + seed |
| API Gateway | Receives every real API call; checks subscription, token and rate limit (exceeded → **429**) | No | None |
| EUIPO IdP | Authenticates portal users; issues OAuth2 tokens to apps | No | Keycloak |
| Portal (IBM devportal 10.0.8.12, Drupal 11) | Catalogue, docs, app registration, subscription requests | Only our modules + theme | Real IBM code |

---

## 3. Customer journey

| Step | What happens | Where our code acts |
|---|---|---|
| 0. Account + organization | EUIPO account created on the EUIPO website (portal self sign-up disabled). The user must belong to a **consumer organization** that exists in API Connect; without one → "No permission" | — |
| 1. Explore (anonymous) | Home (3 steps: Explore → Register, build & test → Apply), `/product` catalogue with category filter, product page with plans (lock = approval, "2 rate limits"), API reference, guides, Problems registry, FAQ, `/status` | `eap_apim_customs` (home blocks, card tags, Sandbox link, TOC), `eap_redoc` (API docs), `eap_api_status` |
| 2. Sign in | Single **"EUIPO Account"** button: portal → API Connect consumer-api → EUIPO IdP → portal. API Connect provides orgs/roles; Drupal creates the user. Per-user session limit (`session_limit`). Admin (uid 1) uses `/user/login?registry_url=/admin` | `eap_apim_customs` (login form alter) — see skill `03` |
| 3. Register an app | `/application/new`: title, description, OAuth redirect URLs. API Connect issues **API Key (`client_id`)** + **API Secret** (shown once; can be reset) | `eap_credentials_sync`: registers the credentials in the IdP; rejects a redirect-URL set identical to another app's — skill `02` |
| 4. Subscribe | Product page → *Select* plan → choose app → **accept conditions of use** → confirm. Sandbox approval ≈ 1 day; Production up to 1 week + identity documents emailed to `docs.apiplatform@euipo.europa.eu` | `eap_apim_customs`: conditions-of-use checkbox — skill `05` |
| 5. Call the API (outside the portal) | Get a token from the IdP, call `https://api.euipo.europa.eu/{api}` with `X-IBM-Client-Id` + `Authorization: Bearer`. Gateway checks subscription, token, rate limit | — |
| 6. Sandbox → Production | Twin portals, same APIs, separate IdPs. Sandbox first; Production after successful Sandbox tests + documents + approval | `eap_apim_customs.is_sandbox` switches the differences — same packages in both |

### Roles in a consumer organization

| Role | Can |
|---|---|
| owner / administrator | Everything: org, members, settings, apps, subscriptions |
| developer | View org; manage **apps and subscriptions** |
| viewer | Read only (cannot create apps) |

### API security (from the Security page)

| Flow | Identifies | Use case | Example |
|---|---|---|---|
| `client_credentials` | App only | API doesn't need the end user | Trademark search (scope `uid`) |
| `authorization code` | App + consenting end user | Acting on behalf of a user; needs the app's redirect URLs | EUTM Filing (scope `eutm-filing.application.write`) |

| Fact | Value |
|---|---|
| Access / refresh token TTL | 2 h / 4 h |
| Rate limit exceeded | HTTP 429 |
| Sandbox token used in Production (or vice versa) | HTTP 401 (separate IdPs) |
| Pagination | 0-based |

### Sandbox vs Production

| | Sandbox | Production |
|---|---|---|
| Data | Realistic test data | Real |
| IdP | `auth-sandbox.euipo.europa.eu` | Own |
| Access | EUIPO account + approval | Sandbox tests done + documents + approval |
| Site name | "Sandbox environment" | "API Portal" |
| `is_sandbox` | On | Off (main menu shows the "Sandbox" link) |

---

## 4. Custom modules in one line

| Module | Role in the journey | Skill |
|---|---|---|
| `eap_credentials_sync` | Propagates every app create/update/delete/secret reset to the EUIPO IdP (outbox + retries + Sync Events report) so app credentials can actually get tokens | `02` |
| `eap_apim_customs` | EUIPO tweaks to IBM features (conditions checkbox, login, Sandbox link, card tags, guide TOC) + "Refresh static content" rebuilding home/footer blocks | `05` (login: `03`) |
| `eap_redoc` | Renders API reference with Redoc inside IBM's page; links API reference ↔ guide | `06` |
| `eap_api_status` | `/status` page embedding EUIPO's PRTG public map | `06` |

---

## 5. Purpose of this repository

A **local, production-like replica** of the portal: EUIPO has no access to IBM's server, so customisations are built and tested here, then shipped as packages (`eap/scripts/build.sh`) uploaded through the admin UI → test portal (real API Connect + IdP) → Sandbox → Production (skill `01`).

### Local limitations that affect functional testing

| Limitation | Consequence |
|---|---|
| No API Gateway | Rate limits are not enforced; *Try it* would hit production services |
| Subscriptions auto-approved | Approval workflow can't be tested |
| No analytics/billing | Those endpoints return 404/501 |
| No webhooks | Catalogue changes arrive via the seed, not live |
| Plan rate limits are placeholders | 100/s soft + 10,000/day hard, set by `seed.php`; real values live in API Connect |

Test users and passwords: `eap/docs/dev-documentation/03-users-and-login.md` and `11-credentials.md`.
