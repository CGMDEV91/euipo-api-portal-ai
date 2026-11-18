---
jira_key: EUIPOFO-00002
euipo_ref: none (internal)
type: corrective
title: "Applications duplicated in the portal: eap_apim_customs 1.2.0, eap_credentials_sync 1.0.1, eap_deployer 1.0.0"
owner: carlos-gonzalez
status: done
resolution: ""
services: [euipo-api-portal]
code_branch: main
base: {}
baseline: {}
skills: [01-credentials-sync, 03-apim-customs, 05-deployer]
known_issues: [EUIPOFO-00002-duplicated-applications-createorupdate-race]
manual_estimate: ""
created: 2026-11-09
updated: 2026-11-09
commits: [c6e60fa]
---

# EUIPOFO-00002 — Applications duplicated in the portal: eap_apim_customs 1.2.0, eap_credentials_sync 1.0.1, eap_deployer 1.0.0

## Intent

Newly created applications appeared twice in the portal and the identity provider received two registrations of the same `client_id` (open IBM support ticket since May). Bring the fix developed on another machine into this checkout, verify it locally, push it to `main` and document it.

## Boundaries

## Prior art

## Analysis

## Impact analysis

## Changes

- `c6e60fa` (2026-11-09) — Fix duplicated applications: eap_apim_customs 1.2.0, eap_credentials_sync 1.0.1, add eap_deployer 1.0.0 (module code, the same tarballs in `eap/build/`, `eap_deployer` added to `eap/scripts/build.sh`).

## Verification

- Locally: `apic_app.application` resolves to `SerializedApplicationService`, the new admin forms build, the Sync Events view imported, 30/30 unit tests; 5/5 concurrent UI + webhook races → 1 node, 1 Create.
- Sandbox: deployed with `eap_deployer`; the user confirmed no duplicated apps since the fix.

## Review

## Test instructions

## Notes

- Generic key: the project's git history has no ticket keys; the work matches commit `c6e60fa`.
- The duplicated consumer organizations reported at the end of this session became EUIPOFO-00003.

## Knowledge

- `.ai/skills/03-apim-customs.md` (fix, cleanup page, settings transfer, payload log), `.ai/skills/01-credentials-sync.md` (idempotent create, dummy portal divergence), new `.ai/skills/05-deployer.md`.
- Known issue `EUIPOFO-00002-duplicated-applications-createorupdate-race`.

## Closure

Migrated closed ticket: fixed, verified on sandbox and committed (see Changes). Time was not recorded in the old notes.
