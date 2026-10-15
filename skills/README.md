# Skills

Reusable knowledge the agent loads on demand.

| # | File | Covers |
|---|---|---|
| 00 | [00-session-tracking.md](00-session-tracking.md) | **Agent behavior** — MEMORY.md/BUGLOG.md maintenance, confirmation prompts, when to propose a skill update. Read once per session. |
| 01 | [01-platform-and-delivery.md](01-platform-and-delivery.md) | IBM product constraints (verified in IBM code: no `update.php`, no in-place upload), deployment today (Extra SCSS upload for styles, UI reinstall for code), environments, and the **why** behind every pattern in `eap/`. **Read first for any new work.** |
| 02 | [02-credentials-sync.md](02-credentials-sync.md) | `eap_credentials_sync`: app credentials registered as OIDC clients in the IdP, IBM hooks, outbox/retries, wire contract, **Sync Events report** read by the identity team, CAS → OIDC, testing on the test portal. |
| 03 | [03-authentication.md](03-authentication.md) | User login: admin registry vs EUIPO Account (OIDC brokered by API Connect), login-form alter, self sign-up. |
| 04 | [04-theme.md](04-theme.md) | `eap_theme`: structure, two CSS pipelines, step-by-step **compile → upload Extra SCSS → verify** flow, full theme update, troubleshooting. |
| 05 | [05-apim-customs.md](05-apim-customs.md) | `eap_apim_customs`: settings, subscription conditions checkbox, sandbox link, product tags, TOC, homepage and **Refresh static content** blocks. |
| 06 | [06-redoc-and-status.md](06-redoc-and-status.md) | `eap_redoc` (Redoc API reference + guide headers) and `eap_api_status` (`/status` iframe). |

## Quick decision guide

- New feature or module? → 01, then the module skill.
- Deploying to test/sandbox/production? → 01 (+ 04 for the theme)
- App created/edited/deleted, secret reset, IdP registration, Sync Events log? → 02
- Login page, registries, who can log in? → 03
- Styles, templates, block regions? → 04 (+ 05 for the static blocks)
- Homepage/footer/banners, subscription wizard, sandbox behaviour? → 05
- API reference page, guides, status page? → 06

Add new skills as `NN-<topic>.md` (next free number) only when a topic recurs and a
generic agent keeps getting it wrong; register it in this table and in the orchestrator.
