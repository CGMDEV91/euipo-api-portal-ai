---
id: EUIPOFO-00003-duplicated-consumer-orgs-createnode-race
title: "Consumer organizations duplicated in the portal (two nodes per organization)"
resolution: external-system
status: fixed
services: [euipo-api-portal]
components: [eap/modules/eap_apim_customs/src/Service/SerializedConsumerOrgService.php]
first_seen: 2026-11-09
last_seen: 2026-11-09
related: [EUIPOFO-00003, EUIPOFO-00002-duplicated-applications-createorupdate-race]
skills: [03-apim-customs]
---

# Consumer organizations duplicated in the portal (two nodes per organization)

## Symptom

The same consumer organization appeared twice in the portal (two nodes with the same timestamp), reported on sandbox on 2026-11-09 after the duplicated applications fix.

## Root cause

From the docs of commit `4acba8a` (`eap/docs/dev-documentation/06-product-changes.md`): IBM's `ibm_apim.consumerorg` (`ConsumerOrgService`) runs `createNode()` / `createOrUpdateNode()` from the login or "Create organization" request and from the `consumer_org_create` webhook without a lock, and `createNode()` clones an organization that already has a node. Same family as `EUIPOFO-00002-duplicated-applications-createorupdate-race`.

## Fix / solution

`eap_apim_customs` 1.3.0: `EapApimCustomsServiceProvider::alter()` swaps `ibm_apim.consumerorg` for `SerializedConsumerOrgService`, which adds a lock per organization URL and turns `createNode()` into an update when the node exists. Cleanup page *EAP duplicated consumer organizations* (`/admin/config/eap_apim_customs/duplicate_consumerorgs`). Committed on `feature/consumerorg-duplicates`; verification not recorded.

## Workaround

The cleanup page removes the extra nodes. Never use IBM's **Delete** on a duplicated organization: it deletes all its applications.

## Pattern to watch

Every IBM service that creates a node from both the UI request and an API Connect webhook can duplicate it; check the other `*Service::create*()` methods fed by webhooks before assuming a new duplicate is a different bug.

## Files

- `eap/modules/eap_apim_customs/src/EapApimCustomsServiceProvider.php`
- `eap/modules/eap_apim_customs/src/Service/SerializedConsumerOrgService.php`
- `eap/modules/eap_apim_customs/src/Service/DuplicateConsumerOrgCleaner.php`, `src/Form/DuplicateConsumerOrgsForm.php`

## References

- EUIPOFO-00003 (2026-11-09) — commit `4acba8a`; built from the git history at the migration (no BUGLOG entry existed)
