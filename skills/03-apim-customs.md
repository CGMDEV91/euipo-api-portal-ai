---
tickets: [EUIPOFO-00002, EUIPOFO-00003]
related_skills: [05-deployer, 02-authentication]
---

# SKILL-03 — Portal customizations (`eap_apim_customs`): settings, static blocks, subscription checkbox, duplicated nodes fixes

## Purpose
Read before touching the EUIPO overrides of IBM portal features: the subscription conditions checkbox, the Sandbox link, product tags, guide TOC, the homepage and the *Refresh static content* blocks, the IBM service swaps that stop duplicated applications and consumer organizations, and the settings export / import. Login form changes are in `.ai/skills/02-authentication.md`.

## Actors / Roles
- Developers: subscribe to products (conditions of use), browse products and guides.
- Portal administrators: settings, *Refresh static content*, the duplicates cleanup pages, settings export / import.
- API Connect: sends the webhooks that race with the portal UI when an app or an organization is created.

Small overrides of IBM portal features plus the homepage/static blocks. Login-form changes are in `.ai/skills/02-authentication.md`.

## Why this module exists

A grab-bag of EUIPO requirements the product doesn't offer as configuration and that can't be done by patching IBM modules. Everything is implemented as **hooks/alters on IBM forms and render arrays**, plus code-defined content that an admin can re-apply with one button.

Reasons marked *(inferred)* come from reading the code and the platform constraints, not from written documentation — confirm before relying on them.

| Feature | Why it is implemented like this |
|---|---|
| Conditions-of-use checkbox in the subscription wizard | EUIPO requires accepting the conditions of use before subscribing; IBM's wizard has no such option → `hook_form_FORM_ID_alter` on IBM's form. The link is a setting because the PDF lives in each portal's Media library |
| `is_sandbox` setting | Sandbox and production run the **same packages**; the differences (colours, Sandbox link, banner text) must be switched by configuration, not by separate builds |
| "Sandbox" main-menu link added in preprocess | *(inferred)* Lives in code instead of a hand-made menu link: no manual step per portal, survives reinstalls, and disappears automatically when `is_sandbox` is on |
| Product tags on product cards | *(inferred)* IBM's card view mode doesn't render `apic_tags`; IBM owns the `product` displays (it protects its fields/view modes and may reset them on upgrade), so preprocess is the upgrade-safe place |
| TOC for guides | *(inferred)* Guides are `ghmarkdown` pages managed by admins; a TOC generated client-side works for any guide without editing content |
| Homepage route with empty markup | *(inferred)* The front page needs a path that IBM doesn't own; the visible content is blocks restricted to `<front>` |
| **Static blocks from Twig templates + "Refresh static content"** | Block content and placements are configuration/content in the DB, not deployable code, and **they are deleted when the theme is reinstalled** (`.ai/services/euipo-api-portal/context/platform-and-delivery.md`, `.ai/services/euipo-api-portal/context/theme-and-styles.md`). Keeping their HTML in the module and recreating them idempotently with a button makes the layout reproducible in any environment and after any theme reinstall. It also deletes `block.block.eap_theme_branding` / `eap_theme_footer`, the copies Drupal creates from the base theme when `eap_theme` is installed |
| Standalone Twig environment for those templates | They are rendered once into `block_content` bodies (`full_html`), outside a request render pipeline |
| Swap of IBM's `apic_app.application` service (1.2.0) | IBM's `createOrUpdate()` races with the `app_create` webhook and duplicates apps; IBM code can't be patched in production, a service override from our module can |
| Duplicates cleanup page (1.2.0) | No DB access in production and the UI "Delete" would delete the real app in API Connect |
| Settings export/import (1.2.0) | A UI reinstall drops every setting |

## Settings (`/admin/config/eap_apim_customs/settings`, config `eap_apim_customs.settings`)

| Key | Effect |
|---|---|
| `is_sandbox` | Sandbox mode: no "Sandbox" link in main menu, products banner without the sandbox paragraph, `sandbox` var in `page.html.twig` |
| `all_register_options` | Show every login registry and allow self sign-up (off in production) |
| `data_protection_link` | Footer "Data protection" link |
| `product_use_assessment_link` | "Conditions of use" link (subscription checkbox + footer) |
| `log_app_payloads` (1.2.0, *Diagnostics*) | Logs every application payload reaching `createOrUpdate()` (channel `eap_apim_customs`) with event, app id, "node existed" and the JSON with secrets replaced by `[redacted: N chars]`. Off by default; a content refresh logs one entry per app, so switch it off after checking |

Admin pages (1.2.0) have menu links prefixed **"EAP"**: *Configuration › System* → EAP APIC customizations, EAP settings export / import; *Reports* → EAP duplicated applications.

## Features

| Feature | Where | Notes |
|---|---|---|
| Accept conditions before subscribing | `hook_form_subscription_wizard_confirm_subscribe_alter` | Prepends a checkbox; submit enabled only when checked (`#states`, client-side only) |
| "Sandbox" link in main menu | `hook_preprocess_menu` (menu `main`) | Only when not sandbox; hardcoded URL `https://dev-sandbox.euipo.europa.eu/` |
| Product tags on product cards | `hook_preprocess_node` (`product`, view mode `card`) | Prefixes `apic_description` with `.eap-product-card-tags` spans |
| Table of contents / active anchors on pages | `hook_entity_view_alter` (`page`, `full`) → library `indexed_pages` | JS builds TOC for `.markdown-guide` (class added by `eap_redoc`) and highlights `.eap-guide-index` links |
| Homepage | Route `/homepage` (`HomepageController`, empty markup) | Front page content comes from blocks |

## "Refresh static content" (`/admin/config/eap_apim_customs/refresh_site`)

Button on the settings form → `RefreshSiteController` → `EapStaticContent::refresh_blocks()`:

1. Sets `system.site:page.front` = `/homepage`.
2. Deletes `block.block.eap_theme_branding` and `block.block.eap_theme_footer`.
3. For each block — **deletes the existing block placement and block_content, then recreates them** from `content/blocks/*.html.twig` (rendered with a standalone Twig env, saved as `basic` block_content in `full_html`):

| Block id | Region (`eap_theme`) | Visibility | Template vars |
|---|---|---|---|
| `eap_site_branding` (system_branding_block) | `top_header` | all | — |
| `eap_main_banner` | `content` | `<front>` | — |
| `eap_steps_banner` | `content` | `<front>` | — |
| `eap_products_banner` | `content` | `/product` | `is_sandbox` |
| `eap_footer` | `footer` | not `/product/*/api/*` | `data_protection_link`, `use_conditions_link` |

⚠️ Manual edits to these blocks in the environment are **lost** on refresh. Run it after changing templates or the links/sandbox settings. `refresh_pages()` (Security page) exists but is disabled and its `content/pages/` folder does not exist.

The seed runs the same controller (`local-dev/scripts/seed.php`, step `eap`).

## Duplicated applications fix (1.2.0, see `EUIPOFO-00002-duplicated-applications-createorupdate-race`)

### Cause (IBM product race)

Creating an app makes IBM build the Drupal `application` node from **two places at once**: the portal UI request (right after POSTing the app to API Connect, event `create`) and the `app_create` **webhook** API Connect sends for the same app. Both end in `ApplicationService::createOrUpdate()` (`apic_app/src/Service/ApplicationService.php`), which looks the node up by `application_id` and creates it if missing **without any lock** → two nodes for one API Connect app (two identical cards in `/application`) and `hook_apic_app_create` fired once per node (double IdP registration). Not related to CAS/OIDC, which is why the OIDC migration didn't fix it.

### `SerializedApplicationService` (swap of IBM's `apic_app.application`)

- `EapApimCustomsServiceProvider::alter()` replaces the class of `apic_app.application`, **keeping IBM's constructor arguments**; lock, logger and config factory come in through setters. Requires `dependencies: apic_app` in the `.info.yml`. Check with `get_class(\Drupal::service('apic_app.application'))`.
- `createOrUpdate()` runs under a Drupal lock `eap_apim_customs:application:<appId>` (timeout 60 s, wait 60 s). The second caller then finds the node and takes IBM's update path. If the lock can't be acquired it logs a warning and **proceeds without lock** (never blocks app creation).
- **Webhook first, UI second:** IBM re-fires `hook_apic_app_create` on the update path only if the node hash changed. With an app without description the hash is identical → the app was never registered in the IdP (pre-existing bug, more frequent once serialized). So when the event is `create` (UI) and the node already existed, the subclass fires `hook_apic_app_create` itself with the UI payload (the one carrying the secret), normalising `client_id`/`client_secret` like IBM's private `invokeAppCreateHook()`. Safe because the only implementation, `eap_credentials_sync`, is idempotent since 1.0.1.
- Upgrading IBM: re-check that `createOrUpdate($app, $event, $formState)` and the protected `$entityTypeManager` / `$moduleHandler` still exist in `ApplicationService`.

### Cleanup page — **EAP duplicated applications** (`/admin/config/eap_apim_customs/duplicate_applications`)

`DuplicateApplicationsForm` + `DuplicateApplicationCleaner`. Fixes the duplicates created before the fix.
- Lists apps with >1 node (two direct field-table queries; `application_id` has no index): app, org, node kept, nodes to remove. Clean all or selected apps.
- Keeps the **lowest nid** (the one IBM's lookup keeps updating). Moves to it credentials/subscriptions referenced only by the removed node (IBM attaches new subscriptions to either node), deletes the copies with the same API Connect uuid not used by any node, moves `application_image` (Drupal-only data) when the kept node has none.
- Deletes nodes with a plain `$node->delete()` — **never** IBM's `deleteNode()` nor the UI "Delete" (they delete the real app in API Connect / the IdP client and the shared credentials). No call to API Connect or the IdP.
- Uses the same per-app lock (wait 10 s); a busy app is skipped and reported → run again. Idempotent, every action logged.
- Irreversible and only tested with mock data: on a real portal review the list first, note the nids, run off-peak and spot-check an app (credentials, subscriptions, image).

### Settings export / import — **EAP settings export / import** (`/admin/config/eap_apim_customs/settings_transfer`)

`SettingsTransferForm` + `SettingsTransfer`: downloads a YAML with every simple-config settings object of installed `eap_*` modules and themes (new modules included automatically); keys containing `secret`/`password` are left out unless *Include secrets* is ticked. Import applies key by key only for installed extensions, keeps the current secret when the file has none, warns when the file comes from another site or has no secrets, rejects files that aren't an export. Config entities (views, blocks) are not handled.

## Duplicated consumer organizations fix (1.3.0)

Taken from commit `4acba8a` (branch `feature/consumerorg-duplicates`, not merged to `main` when this was migrated) and its docs (`eap/docs/module-eap-apim-customs.md`, `eap/docs/dev-documentation/06-product-changes.md`); see `EUIPOFO-00003-duplicated-consumer-orgs-createnode-race`.

- **Cause:** IBM's `ibm_apim.consumerorg` (`ConsumerOrgService`) runs `createNode()` / `createOrUpdateNode()` from the login or "Create organization" request and from the `consumer_org_create` webhook without a lock, and `createNode()` clones an organization that already has a node → two organization nodes.
- **`SerializedConsumerOrgService`** (swapped in by `EapApimCustomsServiceProvider::alter()`, same rules as the application swap): a lock per organization URL (`SerializedConsumerOrgService::lockName()`), and `createNode()` turns into an update when the node already exists.
- **Cleanup page — EAP duplicated consumer organizations** (`/admin/config/eap_apim_customs/duplicate_consumerorgs`, `DuplicateConsumerOrgsForm` + `DuplicateConsumerOrgCleaner`): lists the duplicates, keeps the node with the lowest id and gives it the data only the others had; deletes only the extra Drupal nodes, nothing is sent to API Connect or the identity provider.
- **Never** remove a duplicate with IBM's **Delete**: deleting a consumer organization deletes **all its applications**.
- Upgrading IBM: check that `createNode()` / `createOrUpdateNode()` still exist with the same signatures.

## Call path

<!-- call-path:begin — generated by `node .ai/scripts/ai.mjs callpaths --update`; hand edits are kept -->
Main path of each entry point (distinctive calls only). Full call tree, callers and files: `.ai/context/callgraph.json` — used by `/impact`. Edit the lines below freely: a hand-edited block is never overwritten.

- `EapStaticContent::refresh_blocks()` → `getBlockContent()`
- `EapApimCustomsServiceProvider::alter()`
- `SerializedConsumerOrgService::lockName()`
- `SerializedApplicationService::createOrUpdate()` → `process()` → `loadApplicationNode()`
- `SerializedConsumerOrgService::createNode()` → `withLock()` → `acquireLock()`
- `SerializedConsumerOrgService::createOrUpdateNode()` → `withLock()` → `acquireLock()`
- `DuplicateApplicationCleaner::findDuplicates()` → `fieldTable()` → `tableMapping()` → `nodeStorage()`
- `DuplicateApplicationCleaner::clean()` → `findDuplicates()` → `fieldTable()` → `tableMapping()`
- `DuplicateConsumerOrgCleaner::findDuplicates()` → `fieldTable()` → `tableMapping()` → `nodeStorage()`
- `DuplicateConsumerOrgCleaner::clean()` → `findDuplicates()` → `fieldTable()` → `tableMapping()`
<!-- call-path:end -->

## Key files
- `eap/modules/eap_apim_customs/eap_apim_customs.module` — `eap_apim_customs_form_subscription_wizard_confirm_subscribe_alter()`, `eap_apim_customs_preprocess_menu()`, `eap_apim_customs_preprocess_node()`, `eap_apim_customs_entity_view_alter()`
- `src/EapApimCustomsServiceProvider.php` — `EapApimCustomsServiceProvider::alter()`
- `src/Service/SerializedApplicationService.php` — `SerializedApplicationService::createOrUpdate()`
- `src/Service/SerializedConsumerOrgService.php` — `SerializedConsumerOrgService::createNode()`, `SerializedConsumerOrgService::createOrUpdateNode()`
- `src/Service/DuplicateApplicationCleaner.php` — `DuplicateApplicationCleaner::findDuplicates()`, `DuplicateApplicationCleaner::clean()` · `src/Form/DuplicateApplicationsForm.php`
- `src/Service/DuplicateConsumerOrgCleaner.php` — `DuplicateConsumerOrgCleaner::findDuplicates()`, `DuplicateConsumerOrgCleaner::clean()` · `src/Form/DuplicateConsumerOrgsForm.php`
- `SettingsTransfer::export()`, `SettingsTransfer::import()` · `src/Form/SettingsTransferForm.php`
- `EapStaticContent::refresh_blocks()` · `RefreshSiteController` · `HomepageController` · `content/blocks/*.html.twig`
- `src/Form/EapApimCustomConfigForm.php` — `EapApimCustomConfigForm::buildForm()`, `EapApimCustomConfigForm::submitForm()`

## Verification
Duplicates: race the UI create with the webhook locally (5/5 concurrent UI + webhook races gave 1 node and 1 Create for applications); on sandbox, no new duplicated apps after the 1.2.0 deploy. Static blocks: *Refresh static content* recreates branding, banners and footer in `eap_theme`. Check the swap with `get_class(\Drupal::service('apic_app.application'))`.

## Related known issues
- `EUIPOFO-00002-duplicated-applications-createorupdate-race`
- `EUIPOFO-00003-duplicated-consumer-orgs-createnode-race`

## History
- 2026-10-15 (EUIPOFO-00001): written from the code when the knowledge base was bootstrapped.
- 2026-11-09 (EUIPOFO-00002, `c6e60fa`): 1.2.0, duplicated applications fix, cleanup page, settings export / import, payload log.
- 2026-11-09 (EUIPOFO-00003, `4acba8a`, branch `feature/consumerorg-duplicates`): 1.3.0, duplicated consumer organizations fix and cleanup page; added at the migration from the commit and its docs.
