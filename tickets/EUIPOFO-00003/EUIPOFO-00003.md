---
jira_key: EUIPOFO-00003
euipo_ref: none (internal)
type: corrective
title: "Consumer organizations duplicated in the portal: eap_apim_customs 1.3.0"
owner: carlos-gonzalez
status: done
resolution: ""
services: [euipo-api-portal]
code_branch: feature/consumerorg-duplicates
base: {}
baseline: {}
skills: [03-apim-customs]
known_issues: [EUIPOFO-00003-duplicated-consumer-orgs-createnode-race]
manual_estimate: ""
created: 2026-11-09
updated: 2026-11-09
commits: [4acba8a]
---

# EUIPOFO-00003 — Consumer organizations duplicated in the portal: eap_apim_customs 1.3.0

## Intent

Consumer organizations were duplicated too (e.g. two nodes with the same timestamp for one organization), reported on sandbox after the duplicated applications fix (EUIPOFO-00002) and suspected to be the same race in IBM's consumer organization service.

## Boundaries

## Prior art

## Analysis

## Impact analysis

## Changes

- `4acba8a` (2026-11-09) — Fix duplicated consumer organizations: eap_apim_customs 1.3.0 (`SerializedConsumerOrgService`, `DuplicateConsumerOrgCleaner`, `DuplicateConsumerOrgsForm`, menu link and route, docs `module-eap-apim-customs.md` and `06-product-changes.md`, `eap/build/eap_apim_customs.tar.gz`). On branch `feature/consumerorg-duplicates`, not merged to `main` at the migration.

## Verification

- Not recorded: no session notes exist for this work.

## Review

## Test instructions

## Notes

- Built at the migration from the git history only (commit `4acba8a` and the docs it changed): the old knowledge base had no session or bug entry for it, only the open report in the 2026-11-09 session of EUIPOFO-00002.
- Generic key: the project's git history has no ticket keys.

## Knowledge

- `.ai/skills/03-apim-customs.md` → "Duplicated consumer organizations fix (1.3.0)".
- Known issue `EUIPOFO-00003-duplicated-consumer-orgs-createnode-race`.

## Closure

Status from the commit only: the fix is committed on `feature/consumerorg-duplicates`; its verification and merge are not recorded.
