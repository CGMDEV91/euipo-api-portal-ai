# EUIPO API Portal — Business Workflow Knowledge Base

Each file documents ONE workflow of the portal: actors, rules, edge cases and key files.
**Read the relevant file before implementing or debugging any feature related to that flow.**

Skills are created on demand, never speculatively: a skill is proposed when an area meets the recurrence + specialisation + payoff criteria in `.ai/procedures/skills.md`, and created only after the human approves. They are numbered in creation order.

Technical background that is not a business workflow (the IBM platform constraints and the delivery paths, the theme and its CSS pipelines, the product overview) is in `.ai/services/euipo-api-portal/context/`.

## Index

| # | File | Covers |
|---|---|---|
| 01 | [01-credentials-sync.md](01-credentials-sync.md) | `eap_credentials_sync`: app credentials registered as OIDC clients in the identity provider, IBM `apic_app` hooks, outbox and retries, wire contract, the **Sync Events report** the identity team reads |
| 02 | [02-authentication.md](02-authentication.md) | User login: admin registry vs EUIPO Account (OIDC brokered by API Connect), login form alter, self sign-up |
| 03 | [03-apim-customs.md](03-apim-customs.md) | `eap_apim_customs`: settings, subscription conditions checkbox, Sandbox link, product tags, TOC, homepage and **Refresh static content**, the **duplicated applications and consumer organizations fixes** (IBM service swaps + cleanup pages), settings export / import |
| 04 | [04-redoc-and-status.md](04-redoc-and-status.md) | `eap_redoc` (Redoc API reference + guide headers) and `eap_api_status` (`/status` iframe) |
| 05 | [05-deployer.md](05-deployer.md) | `eap_deployer`: in-place deploy of `eap_*` modules and theme without uninstalling, review, stable / new packages, rollback, diagnostics, pre-deploy checklist |

## Quick decision guide

- **New feature or module?** → `.ai/services/euipo-api-portal/context/platform-and-delivery.md`, then the module skill
- **Deploying to the test portal, sandbox or production?** → `platform-and-delivery.md` + SKILL-05 (+ `theme-and-styles.md` for the theme)
- **App created, edited, deleted, secret reset, identity provider registration, Sync Events?** → SKILL-01
- **Duplicated nodes of IBM-synced content (apps, consumer organizations)?** → SKILL-03
- **Login page, registries, who can log in?** → SKILL-02
- **Styles, templates, block regions?** → `.ai/services/euipo-api-portal/context/theme-and-styles.md` (+ SKILL-03 for the static blocks)
- **Homepage, footer, banners, subscription wizard, sandbox behaviour?** → SKILL-03
- **API reference page, guides, status page?** → SKILL-04

## Maintenance rules

- Update the relevant skill in the same task that changes the behaviour it documents. A merged change with a stale skill is not Done.
- Cross-reference known issues: when a confirmed known issue touches a documented workflow, add it to that skill's "Related known issues".
- New skill files are created from `.ai/templates/SKILL.template.md`.
