# Platform constraints, deployment and why the code looks like this

Read before designing anything. Almost every "strange" decision in `eap/` comes from the constraints described here.

---

## 1. The product we build on

- The portal is **IBM API Connect Developer Portal** (10.0.8.x, Drupal 11) — a managed product. **We have no access to its repository, servers, shell, database or `settings.php`.**
- The IBM folders in this repo (`apim_profile/`, `ibm_apim/`, `apic_app/`, `consumerorg/`, `auth_apic/`, `connect_theme/`, …) are a **read-only copy** of the public `ibm-apiconnect/devportal` source, kept so the local site behaves like production. Read them to find hooks and understand behaviour; never change them expecting it to ship. Local patches in `apim_profile/patches` only make the local copy run.
- The portal is fed by **API Connect** (management plane): catalog, consumer orgs, apps and credentials live there and are pushed/synced to the portal. Users log in through the identity provider configured as a user registry in API Connect (`.ai/skills/02-authentication.md`).
- The **only things that reach an environment are our custom modules, our theme, and what an administrator does in the admin UI.**

## 2. What the product allows and forbids (verified in IBM code)

| Fact | Evidence | Consequence for us |
|---|---|---|
| Extensions are added through **Extend › Add new module** (`/admin/modules/install`) and **Appearance › Install new theme** (`/admin/theme/install`) — Drupal Update Manager, tarball upload | `update` module enabled in `apim_profile.info.yml`; menu link in `ibm_apim.links.menu.yml` | Every deliverable is a `.tar.gz` per extension (`eap/scripts/build.sh`) |
| Upload **refuses an extension whose files already exist** ("%project is already present") | `core/modules/update/src/Form/UpdateManagerInstall.php` | There is no in-place upgrade through the UI: the old version must be removed first |
| Custom modules can be deleted only when **disabled** (`/admin/modules/delete`); custom themes only when **uninstalled and not default/admin** (`/admin/appearance/delete`) | `ibm_apim/src/Form/CustomModulesDeleteForm.php`, `IbmApimThemeInstallController::delete/uninstall` | Upgrade = uninstall → delete → upload → install |
| **`/update.php` is removed** | `IbmApimRouteSubscriber`: `$collection->remove('system.db_update')` | `hook_update_N` / `hook_post_update_NAME` **cannot be triggered by us**. Don't rely on them |
| No drush, no composer, no `settings.php` | managed product | Everything configurable must be a **settings form**; new libraries can't be added (only what's in the product `vendor/`: scssphp, Guzzle, Twig…) |
| Theme install / set-default runs `compile_scss()` when the theme has `auto_build_scss: 1` | `IbmApimThemeInstallController::install/setDefaultTheme` | `css/style.css` is produced **by the product** on install, from our `scss/` |
| Config module is enabled (single-item import/export UI) | `config` in `apim_profile.info.yml` | Possible manual way to move a config item between environments — verify access before relying on it |

## 3. Deployment paths

### Current practice

| Change | How it reaches an environment |
|---|---|
| **Styles** (the usual case) | **C — Extra SCSS upload.** No deployment: the theme stays installed. |
| Module code (PHP, YAML, Twig, JS) that adds no entity/table | **B — In-place deploy** with `eap_deployer` (settings and data kept). Fallback: A. |
| Theme templates, assets, `.theme` PHP | **B — In-place deploy** (keeps blocks and `css/`, recompiles SCSS). Fallback: A, expensive. |
| New entity types/fields/tables, or `eap_deployer` itself | **A — UI reinstall**. |

`eap_deployer` 1.0.0 was verified on the **sandbox** portal (2026-11) deploying `eap_apim_customs` 1.2.0 and `eap_credentials_sync` 1.0.1. Full details: `.ai/skills/05-deployer.md`.

### B. In-place deploy (`eap_deployer`) — preferred for code

1. Once per portal: install `eap_deployer` through path A (it saves a **stable** copy of every installed `eap_*` extension).
2. **Configuration › Development › EAP deployer** → upload the full `eap/build/<ext>.tar.gz` → **Review** (diff, errors, warnings) → **Deploy**.
3. Run the module's post-deploy buttons for anything shipped as config (the deployer doesn't import `config/install|optional` nor run install/update hooks).
4. Check, then **Deprecate stable** (keep) or **Roll back**.

Before the first deploy to a portal, download that portal's **stable** package and diff it against git: a portal may run code that isn't in the repo (it happened with the dummy portal's `eap_credentials_sync`, see `.ai/skills/01-credentials-sync.md`).

### C. Style-only change — the normal way to update the theme

Compile locally → upload the flattened `styles/extra.scss` in **Appearance › Settings › eap_theme › Extra SCSS** → the server compiles it into `css/extra.css`. Nothing is uninstalled, so blocks and settings are kept. Full step-by-step in `context/theme-and-styles.md`.

### A. UI reinstall (the only way to ship code)

**Module upgrade**
1. Export the settings in `eap_apim_customs` › **EAP settings export / import** (`/admin/config/eap_apim_customs/settings_transfer`, YAML with every `eap_*` settings object; tick *Include secrets* only if the file is kept somewhere safe). For `eap_credentials_sync`, export/check pending or failed sync events.
2. **Extend › Uninstall** the module. Drupal refuses to uninstall a module whose content entities still exist → for `eap_credentials_sync` use *Delete all sync events* first.
3. `/admin/modules/delete` → delete its files.
4. `/admin/modules/install` → upload the new `eap/build/<module>.tar.gz`.
5. Enable it and **import the settings YAML** from step 1 (or re-enter them; values per environment are in `eap/docs/module-*.md`). The import only touches installed `eap_*` extensions and keeps the current secret when the file has none.
6. Run the module-specific post-steps (e.g. *Refresh static content* for `eap_apim_customs`).

**Theme upgrade** (expensive — only when Flow C can't cover the change, see `context/theme-and-styles.md`):
1. Take a screenshot of **Structure › Block layout** for `eap_theme`.
2. Set another theme as default (e.g. `connect_theme`), uninstall `eap_theme` → **Drupal deletes every block placed in `eap_theme`**.
3. Delete it (`/admin/appearance/delete`), upload the new `eap_theme.tar.gz` (`/admin/theme/install`), install it and set it as default → the product compiles `css/style.css` and `css/mail.css`.
4. Upload the **Extra SCSS** (`css/extra.css` is not in the package).
5. *Refresh static content* in `eap_apim_customs` (recreates branding, banners and footer) and re-place every other block by hand, using the screenshot.

### Open question — multi-node portals

Path B writes files from PHP on the node that serves the request. If production has several portal nodes and the platform doesn't share/replicate that folder, nodes could diverge. Check with the deployer's **Diagnostics** tab (serving host + on-disk fingerprint per extension) when using B on a new environment, and ask IBM if in doubt.

## 4. Environments

```
local (ddev: Keycloak + mock-apic + seed)  →  TEST portal (real API Connect + IdP)  →  sandbox  →  production
```

- **Local** reproduces the product; it is where code is written and unit/functional checks happen.
- **Test portal** is connected to the real IBM product and identity provider: only there can we verify real login, real app creation in API Connect and real credential-registration events (`.ai/skills/01-credentials-sync.md`).
- **Sandbox** (`dev-sandbox.euipo.europa.eu`) and **production** (`dev.euipo.europa.eu`) run the **same packages**; differences are settings (`is_sandbox`, endpoints, links).

## 5. How the constraints shaped the code (the "why" catalogue)

| Constraint | Pattern adopted | Where |
|---|---|---|
| No `settings.php`, no drush | Every external value is a **config form** (endpoints, client id/secret, links, iframe URL) | all `*ConfigForm.php` |
| Same package for sandbox and production | Behaviour switched by a setting, not by a separate build | `eap_apim_customs.settings:is_sandbox` |
| No `update.php`, reinstall loses data | No `hook_update_N`; idempotent **"apply from code" buttons** an admin can press | *Refresh static content* (`eap_apim_customs`), *Import Sync Events config* (`eap_credentials_sync`) |
| Uninstall blocked while content entities exist | *Delete all sync events* button | `SyncEventDeleteAllForm` |
| Reinstalling the theme wipes the block layout | Theme updates avoided; styles shipped through the **Extra SCSS** upload; key blocks recreated from Twig files in a module | `eap_theme.theme`, `EapStaticContent` |
| Can't patch IBM modules | Hook into IBM hooks/alters and override in the browser (JS) | `apic_app` hooks, `hook_form_*_alter`, `hook_entity_view_alter` + JS (Redoc) |
| Can't patch an IBM **service** | Subclass it and swap the class in a `ServiceProviderBase::alter()`, keeping IBM's constructor arguments and injecting extras through setters | `EapApimCustomsServiceProvider` → `SerializedApplicationService` (`.ai/skills/03-apim-customs.md`) |
| Reinstall loses settings | Settings export/import YAML; in-place deploy | `SettingsTransfer` (`.ai/skills/03-apim-customs.md`), `eap_deployer` (`.ai/skills/05-deployer.md`) |
| No composer/npm on the server | Libraries **bundled** inside the module; SCSS pre-flattened locally | `eap_redoc/js/redoc.standalone.js`, `styles/extra.scss` |
| No DB/server access to diagnose | Diagnostics stored **in the portal and visible in the admin UI** | Sync Events report (successor of the `dblog_persistent` workaround, see `.ai/skills/01-credentials-sync.md`) |
| Content not deployable as code | Content is either created by admins following `eap/docs/content-creation.md`, or kept in Twig templates inside a module | `eap_apim_customs/content/blocks/` |

## 6. Rules for new work

- [ ] Everything an operator must set → a settings form; document the per-environment values in `eap/docs/`.
- [ ] Anything that must exist after install/upgrade → idempotent and re-runnable from a button (never only `hook_install`/`hook_update_N`).
- [ ] No new Composer/npm dependency at runtime; bundle assets inside the module.
- [ ] No reliance on keeping data across a module reinstall.
- [ ] Every admin page has a route **and** a menu link (`*.links.menu.yml`); titles of our custom routes/links start with **"EAP"** so they are recognisable among IBM's.
- [ ] Bump `version:` in `*.info.yml`, run `eap/scripts/build.sh`, commit `eap/build/*.tar.gz`.
- [ ] Update the operator docs (`eap/docs/`) and the relevant skill.
- [ ] Verify on the test portal before sandbox/production.
