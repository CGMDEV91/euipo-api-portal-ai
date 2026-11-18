---
id: EUIPOFO-00002-duplicated-applications-createorupdate-race
title: "Applications duplicated in the portal (two cards / two nodes per app)"
resolution: external-system
status: fixed
services: [euipo-api-portal]
components: [eap/modules/eap_apim_customs/src/Service/SerializedApplicationService.php, eap/modules/eap_credentials_sync/eap_credentials_sync.module]
first_seen: 2026-11-09
last_seen: 2026-11-09
related: [EUIPOFO-00002, EUIPOFO-00003-duplicated-consumer-orgs-createnode-race]
skills: [03-apim-customs, 01-credentials-sync]
---

# Applications duplicated in the portal (two cards / two nodes per app)

## Symptom

Newly created applications appeared twice in the portal (e.g. two "Symbiofy Trademark Check" cards, same `/application/<id>`). Open IBM support ticket since May. The CAS → OIDC migration was believed to have fixed it, but it kept happening. The IdP also received two registrations of the same `client_id`.

## Root cause

IBM product race, not our sync module nor CAS. On app creation IBM builds the `application` node from two places concurrently: the portal UI request (after POSTing to API Connect, event `create`) and the `app_create` webhook from API Connect. Both call `ApplicationService::createOrUpdate()` (`apic_app/src/Service/ApplicationService.php`), which looks up the node by `application_id` and creates it when missing **without any lock** → two nodes. `hook_apic_app_create` ran on both paths → `eap_credentials_sync` registered the client twice. Secondary pre-existing bug: when the webhook arrived first without secret and the UI payload produced an identical node hash (e.g. app without description), IBM did not re-fire the create hook → the app was never registered in the IdP.

## Fix / solution

- `eap_apim_customs` 1.2.0: `EapApimCustomsServiceProvider` swaps IBM's `apic_app.application` for `SerializedApplicationService` (subclass): per-app Drupal lock around `createOrUpdate()`; fires `hook_apic_app_create` when the UI create finds an existing node; optional payload logging (secrets redacted). Cleanup page *EAP duplicated applications* for the existing duplicates (keeps lowest nid, merges credentials/subscriptions/image, plain node delete, no API Connect/IdP calls).
- `eap_credentials_sync` 1.0.1: idempotent create (`SyncEventRepositoryInterface::exists()`; duplicate CREATE skipped).
- Reproduced and verified locally (5/5 concurrent UI+webhook races → 1 node, 1 Create); deployed on sandbox with `eap_deployer`.

## Workaround

The cleanup page *EAP duplicated applications* removes the duplicates created before the fix.

## Pattern to watch

Every IBM `*Service::createOrUpdate()` fed by both the UI and webhooks has the same unlocked find-or-create (consumer organizations duplicated the same way: `EUIPOFO-00003-duplicated-consumer-orgs-createnode-race`). Any `hook_apic_*_create` implementation must be idempotent. A deploy review listing unknown modified/removed files means the portal runs code not in git (dummy portal's `eap_credentials_sync`).

## Files

- `eap/modules/eap_apim_customs/src/EapApimCustomsServiceProvider.php`
- `eap/modules/eap_apim_customs/src/Service/SerializedApplicationService.php`
- `eap/modules/eap_apim_customs/src/Service/DuplicateApplicationCleaner.php`, `src/Form/DuplicateApplicationsForm.php`
- `eap/modules/eap_credentials_sync/eap_credentials_sync.module`, `src/Application/SyncEventRecorder.php`, `src/Domain/Repository/SyncEventRepositoryInterface.php`, `src/Infrastructure/Repository/DrupalSyncEventRepository.php`

## References

- EUIPOFO-00002 (2026-11-09) — fixed and verified on sandbox; legacy id BUG-001 of the old BUGLOG; commit `c6e60fa`
