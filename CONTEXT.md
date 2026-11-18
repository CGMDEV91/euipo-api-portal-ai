# CONTEXT — EUIPO API Portal

Project-level facts that change rarely. Changed through its own pull request, never as part of a ticket.

## Permanent decisions
- The IBM product is never patched for production: EUIPO behaviour lives in the `eap_*` modules and `eap_theme`, through hooks, alters and service swaps (`.ai/services/euipo-api-portal/context/platform-and-delivery.md`).
- IBM services that race with the API Connect webhooks are replaced at runtime by `eap_apim_customs` (`apic_app.application`, `ibm_apim.consumerorg`), keeping IBM's constructor arguments (EUIPOFO-00002, EUIPOFO-00003, `.ai/skills/03-apim-customs.md`).
- Code ships in place with `eap_deployer`; styles ship through the Extra SCSS upload; a UI reinstall is the last resort (`.ai/skills/05-deployer.md`).
- The repository version of `eap_credentials_sync` is the good one (confirmed by the team, 2026-11); the dummy portal's 1.0.0 with a WSO2 client was abandoned.

## Manual test procedures
- Real login, real app creation in API Connect and real credential registration can only be verified on the **test portal** connected to the real API Connect and identity provider (`.ai/skills/01-credentials-sync.md`, `.ai/skills/02-authentication.md`).
- Duplicates cleanup pages: review the list and note the node ids first, run off-peak, then spot-check one or two cleaned items (credentials, subscriptions, image).

## Technical data
- Environments: local (ddev: Keycloak + mock-apic + seed) → test portal (real API Connect + identity provider) → sandbox `dev-sandbox.euipo.europa.eu` → production `dev.euipo.europa.eu`. Sandbox and production run the same packages; the differences are settings (`is_sandbox`, endpoints, links).
- Production runs IBM API Developer Portal 10.0.8.11, local runs 10.0.8.12 (no public 10.0.8.11).
- Monitoring: EUIPO's PRTG public sensor map, embedded in `/status`.
- Jira project key: not recorded yet. The tickets migrated from the old session log use generic keys `EUIPOFO-00001`, `EUIPOFO-00002`, `EUIPOFO-00003` (`euipo_ref: none (internal)`).
