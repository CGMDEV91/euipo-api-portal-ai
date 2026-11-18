---
jira_key: EUIPOFO-00001
euipo_ref: none (internal)
type: development
title: "Knowledge base bootstrapped for the EUIPO API Portal"
owner: carlos-gonzalez
status: done
resolution: ""
services: [euipo-api-portal]
code_branch: main
base: {}
baseline: {}
skills: [02-authentication, 04-redoc-and-status]
known_issues: []
manual_estimate: ""
created: 2026-10-15
updated: 2026-10-15
commits: []
---

# EUIPOFO-00001 — Knowledge base bootstrapped for the EUIPO API Portal

## Intent

Bootstrap the `.ai/` knowledge base of the EUIPO API Portal (adapted from another project's copy) and document the custom modules, the theme and the IBM platform constraints, each with a "why" section; reasons not backed by docs marked *(inferred)*.

## Boundaries

## Prior art

## Analysis

## Impact analysis

## Changes

- No code commit: the work was the knowledge base itself (`.ai/**`, its own repository `CGMDEV91/euipo-api-portal-ai`) and the local `.git/info/exclude` of the project.

## Verification

- Verified in IBM code: `/update.php` removed (`IbmApimRouteSubscriber`), Update Manager refuses to upload an existing extension, custom extensions deletable only when uninstalled → upgrade = uninstall → delete → upload → install.
- Confirmed with the user: styles ship only through the Extra SCSS upload.

## Review

## Test instructions

## Notes

- Generic key: no Jira ticket and no code commit for this work (the project's git history has no ticket keys).

## Knowledge

- The old skills 01–06 became `.ai/skills/01`–`04` and the context areas `platform-and-delivery.md`, `theme-and-styles.md` of `.ai/services/euipo-api-portal/context/` at the migration.

## Closure

Migrated closed ticket: the work was confirmed in its session. Resolution and time were not recorded in the old notes.
