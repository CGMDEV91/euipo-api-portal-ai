# 01 — Platform constraints, deployment and why the code looks like this

Read before designing anything. Almost every "strange" decision in `eap/` comes from the constraints described here.

---

## 1. The product we build on

- The portal is **IBM API Connect Developer Portal** (10.0.8.x, Drupal 11) — a managed product. **We have no access to its repository, servers, shell, database or `settings.php`.**
- The IBM folders in this repo (`apim_profile/`, `ibm_apim/`, `apic_app/`, `consumerorg/`, `auth_apic/`, `connect_theme/`, …) are a **read-only copy** of the public `ibm-apiconnect/devportal` source, kept so the local site behaves like production. Read them to find hooks and understand behaviour; never change them expecting it to ship. Local patches in `apim_profile/patches` only make the local copy run.
- The portal is fed by **API Connect** (management plane): catalog, consumer orgs, apps and credentials live there and are pushed/synced to the portal. Users log in through the identity provider configured as a user registry in API Connect (`03-authentication.md`).
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
| Module code (PHP, YAML, Twig, JS) | **A — UI reinstall** of that module. |
| Theme templates, assets, `.theme` PHP, base variables | **A — UI reinstall** of the theme (expensive, avoided). |

The whole extension folder is **not** overwritten in place today: there is no such mechanism in the product UI. If one is ever offered (by IBM or the platform team), see *Not used today* below.

### C. Style-only change — the normal way to update the theme

Compile locally → upload the flattened `styles/extra.scss` in **Appearance › Settings › eap_theme › Extra SCSS** → the server compiles it into `css/extra.css`. Nothing is uninstalled, so blocks and settings are kept. Full step-by-step in `04-theme.md`.

### A. UI reinstall (the only way to ship code)

**Module upgrade**
1. Note the module's settings (they will be lost) and, for `eap_credentials_sync`, export/check pending or failed sync events.
2. **Extend › Uninstall** the module. Drupal refuses to uninstall a module whose content entities still exist → for `eap_credentials_sync` use *Delete all sync events* first.
3. `/admin/modules/delete` → delete its files.
4. `/admin/modules/install` → upload the new `eap/build/<module>.tar.gz`.
5. Enable it, **re-enter its settings** (values per environment are in `eap/docs/module-*.md`).
6. Run the module-specific post-steps (e.g. *Refresh static content* for `eap_apim_customs`).

**Theme upgrade** (expensive — only when Flow C can't cover the change, see `04-theme.md`):
1. Take a screenshot of **Structure › Block layout** for `eap_theme`.
2. Set another theme as default (e.g. `connect_theme`), uninstall `eap_theme` → **Drupal deletes every block placed in `eap_theme`**.
3. Delete it (`/admin/appearance/delete`), upload the new `eap_theme.tar.gz` (`/admin/theme/install`), install it and set it as default → the product compiles `css/style.css` and `css/mail.css`.
4. Upload the **Extra SCSS** (`css/extra.css` is not in the package).
5. *Refresh static content* in `eap_apim_customs` (recreates branding, banners and footer) and re-place every other block by hand, using the screenshot.

### Not used today — full-folder overwrite (hypothetical)

If the extension folders were ever replaced on the server without uninstalling, settings, data and block layout would be kept. Preconditions to check before relying on it:
- The theme folder would need its **compiled `css/`** (`build.sh` excludes it, because path A recompiles on install).
- No install-time logic would run (`config/install`/`config/optional`, new entity fields/tables).
- Caches would have to be cleared afterwards.

## 4. Environments

```
local (ddev: Keycloak + mock-apic + seed)  →  TEST portal (real API Connect + IdP)  →  sandbox  →  production
```

- **Local** reproduces the product; it is where code is written and unit/functional checks happen.
- **Test portal** is connected to the real IBM product and identity provider: only there can we verify real login, real app creation in API Connect and real credential-registration events (`02-credentials-sync.md`).
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
| No composer/npm on the server | Libraries **bundled** inside the module; SCSS pre-flattened locally | `eap_redoc/js/redoc.standalone.js`, `styles/extra.scss` |
| No DB/server access to diagnose | Diagnostics stored **in the portal and visible in the admin UI** | Sync Events report (successor of the `dblog_persistent` workaround, see `02`) |
| Content not deployable as code | Content is either created by admins following `eap/docs/content-creation.md`, or kept in Twig templates inside a module | `eap_apim_customs/content/blocks/` |

## 6. Rules for new work

- [ ] Everything an operator must set → a settings form; document the per-environment values in `eap/docs/`.
- [ ] Anything that must exist after install/upgrade → idempotent and re-runnable from a button (never only `hook_install`/`hook_update_N`).
- [ ] No new Composer/npm dependency at runtime; bundle assets inside the module.
- [ ] No reliance on keeping data across a module reinstall.
- [ ] Bump `version:` in `*.info.yml`, run `eap/scripts/build.sh`, commit `eap/build/*.tar.gz`.
- [ ] Update the operator docs (`eap/docs/`) and the relevant skill.
- [ ] Verify on the test portal before sandbox/production.
