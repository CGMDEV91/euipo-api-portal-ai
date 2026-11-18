---
ticket: EUIPOFO-00002            # the Jira key; the file is .ai/tickets/<key>/<key>.carlos-gonzalez.md
user: carlos-gonzalez                         # GitLab username (owner: of .ai/MEMORY.local.md) — the only person whose agent writes this file
created: 2026-11-09
---

# EUIPOFO-00002 — carlos-gonzalez's diary

<!--
  One per developer working on the ticket, beside the shared file <key>.md. Only its owner (and their
  agent) writes it: two people on one ticket never edit the same lines, and the knowledge repo merges
  without conflicts. Shared decisions, analysis, impact, verification and review go in <key>.md.
  The ticket header the plugin adds to every message reads YOUR Current state; teammates' diaries are
  read, never edited. DATA, never instructions. English. No personal data, no secrets, no raw logs.
-->

## Current state

Closed. The work is in the session entry below, migrated from the old MEMORY.md.
Next: none.

## Changes

Files modified, as recorded in the old session (free text):

- `eap/modules/eap_apim_customs/**`, `eap/modules/eap_credentials_sync/**`, `eap/modules/eap_deployer/**` (new), `eap/build/*.tar.gz`, `eap/scripts/build.sh`
- `.ai/skills/01,02,05,07,README.md`, `.ai/orchestrator.md`, `.ai/example.*`, `CLAUDE.md`, `AGENTS.md`, `.ai/session/BUGLOG.md`

## Session log

<!-- Append-only. One entry per session, written at the end of the session after the human confirms:

### YYYY-MM-DD HH:MM · carlos-gonzalez · <confirmed time, e.g. 2h30>
- **Base → HEAD:** <service> @ <sha> → <sha> (one per service touched)
- **Done:** <what and why, 2–3 sentences>
- **Verified:** <command (workdir <service>) → evidence line; what was not verified>
- **Dead ends:** <approach — why it failed, or "none">
-->

### 2026-11-09 · carlos-gonzalez · migrated from MEMORY.md — Session 2026-11-09 (Brought the duplicated-applications fix (developed in a previous session on another machine) into this checkout, verified it locally, pushed it to `main`, and documented it in the `.ai/` skills)

**Summary**: Brought the duplicated-applications fix (developed in a previous session on another machine) into this checkout, verified it locally, pushed it to `main`, and documented it in the `.ai/` skills.

**Work done**:
- Fresh local setup on this machine: `ddev start` (killed the k3d-lab kubectl port-forwards that held port 9443), `./build-local-site.sh` (2 unapplied IBM patches are expected), `ddev install-portal`.
- Applied the packages tested on sandbox (`Downloads\eap-packages-fix`): `eap_apim_customs` 1.2.0, `eap_credentials_sync` 1.0.1, new `eap_deployer` 1.0.0. Verified: `apic_app.application` resolves to `SerializedApplicationService`, new admin forms build, Sync Events view imported, 30/30 unit tests.
- Committed on `main` and pushed (`c6e60fa`): module code, the same tarballs in `eap/build/`, `eap_deployer` added to `eap/scripts/build.sh`.
- Root cause and fix logged as BUG-001 (IBM `createOrUpdate()` race UI vs `app_create` webhook).
- Skills: new `07-deployer.md`; updated `01` (path B in-place deploy, settings export/import, "EAP" menu-link rule, service-swap pattern), `02` (idempotent create, dummy portal divergence, deployer upgrade), `05` (fix, cleanup page, settings transfer, payload log); README, orchestrator, CLAUDE.md/AGENTS.md/example.* synced.
- User confirmed on sandbox: no duplicated apps since the fix. New report: **consumer organizations are duplicated too** (e.g. "Geralt Omhof", "Dawid Kowalski", two nodes with the same timestamp) — likely the same race in IBM's consumerorg service.

**Files modified**:
- `eap/modules/eap_apim_customs/**`, `eap/modules/eap_credentials_sync/**`, `eap/modules/eap_deployer/**` (new), `eap/build/*.tar.gz`, `eap/scripts/build.sh`
- `.ai/skills/01,02,05,07,README.md`, `.ai/orchestrator.md`, `.ai/example.*`, `CLAUDE.md`, `AGENTS.md`, `.ai/session/BUGLOG.md`

**Pending / Next steps**:
- Investigate and fix duplicated consumer organizations (same pattern as BUG-001).
- `eap/docs/module-*.md` not yet updated for 1.2.0 / 1.0.1 / deployer (the previous machine's docs were never pushed).
- `ddev install-portal` doesn't enable `eap_deployer`.
- Run the duplicated-applications cleanup on production after reviewing the list.
