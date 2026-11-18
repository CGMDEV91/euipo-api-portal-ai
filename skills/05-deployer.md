---
tickets: [EUIPOFO-00002]
related_skills: [01-credentials-sync, 03-apim-customs]
---

# SKILL-05 — In-place deploy of `eap_*` extensions (`eap_deployer`)

## Purpose
Read before shipping a new version of any `eap_*` module or of `eap_theme` to the test portal, sandbox or production.

## Actors / Roles
- Portal administrators (permission `administer software updates`): upload, review, deploy, roll back, promote.
- The developer who builds the package (`eap/scripts/build.sh`) and checks the portal's stable package against git.

Read before shipping a new version of any `eap_*` module or of `eap_theme` to test, sandbox or production.

## Why this module exists

The product only upgrades an extension by **uninstall → delete → upload → install** (`.ai/services/euipo-api-portal/context/platform-and-delivery.md`). That loses the module settings and data (e.g. every sync event, since *Delete all sync events* is required to uninstall `eap_credentials_sync`), and for the theme it loses the whole block layout. `eap_deployer` replaces the **files** of an installed `eap_*` extension with the content of a new package **without uninstalling it**, so settings, data and blocks are kept.

It is viable because IBM's own upload/delete flow writes and removes extension folders under `sites/<site>/modules|themes` from PHP (`ibm_apim/src/Updater/ApicModule.php`, `ApicModuleService`): the web process can write there.

Verified: deployed `eap_apim_customs` 1.2.0 and `eap_credentials_sync` 1.0.1 on the **sandbox** portal (2026-11) with settings and sync events kept.

## Where

**Configuration › Development › EAP deployer** — `/admin/config/development/eap_deployer` (permission `administer software updates`, same as *Add new module*). Tab **Diagnostics**: `/admin/config/development/eap_deployer/diagnostics`.

## Flow

1. **Upload** the full package built by `eap/scripts/build.sh` (`.tar.gz`, `.zip` also accepted). Always the **whole** extension, never a partial patch: partial patches can't delete files, leave the server in a state that matches no tarball in git, and can mix versions.
2. **Review** (`Deployer::inspect()`): installed vs new version, files added / modified / removed, plus
   - **errors (block the deploy):** not an `eap_*` extension; not installed; package type ≠ installed type; installed outside the site path; the deployer itself; missing dependencies / base theme; code IBM's uploader would refuse (`ApicModule::checkFunctionNames`, IBM function prefixes); unsafe archive paths.
   - **warnings:** items of the install and optional config new or changed (not applied in place), `.install` changed (no install/update hooks run), same or older version, identical package.
3. **Deploy** (`Deployer::apply()`):
   - saves the current files as **stable** if there is no stable package yet;
   - copies the package to a hidden staging folder next to the target, swaps folders with two `rename()` calls (old one restored if the second fails), deletes the old one;
   - theme: keeps `css/` (compiled `style.css`/`mail.css` + `extra.css` from the Extra SCSS upload) and recompiles the SCSS with IBM's compiler;
   - `opcache_reset()` + `drupal_flush_all_caches()`;
   - warns about pending entity schema changes (those need a real reinstall);
   - stores the package as **new**, records history.

## Stable / new model (table `eap_deployer_release`, archives stored in the DB)

| Action | Result |
|---|---|
| Install `eap_deployer` (or *Save missing stable versions*, idempotent) | Every installed `eap_*` module/theme except the deployer gets a **stable** package = files on disk at that moment |
| Deploy v2 | stable = v1, new = v2 |
| Deploy v3 before deciding on v2 | stable = v1, new = v3 (v2 dropped; the review warns) |
| **Roll back** | Reinstalls stable (through the same inspect/apply), deletes new |
| **Deprecate stable** (promote) | Deletes stable; new becomes stable for the next deploy |

Each slot can be downloaded from the Modules / Themes tables (`/release/{extension}/{slot}`). **The stable download is the exact code installed on that portal** — use it to compare a portal against the repo before deploying (this is how the dummy's divergent `eap_credentials_sync` was found, see `.ai/skills/01-credentials-sync.md`).

⚠️ Install the deployer **before** changing the other modules, otherwise "stable" already holds the new code.

## What an in-place deploy does NOT do

- No `hook_install`, no import of the module's install and optional config, no update hooks, no new entity types/fields. Apply shipped config with the module's own idempotent button (e.g. *Import Sync Events config* in `eap_credentials_sync`). A version that adds entities/tables still needs a reinstall.
- It can't deploy itself: upgrade `eap_deployer` with the classic uninstall → delete → upload → install (its stable/new packages are lost then).
- Multi-node portals: if the platform replicates files only through IBM's own install flow, nodes could diverge. The Diagnostics tab shows the serving host and the on-disk version/fingerprint of each `eap_*` extension: reload it several times on a new environment and check every node has the new fingerprint.

## Before deploying to a portal (checklist)

- [ ] Download the **stable** package from that portal and diff it against the repo's version. Unknown *modified/removed* files in the review = the portal runs code that isn't in git → stop.
- [ ] Export settings (`eap_apim_customs` › *EAP settings export / import*) as an extra safety net.
- [ ] Deploy, check the review warnings, run the module's post-deploy buttons.
- [ ] Check Diagnostics and the module's own report (e.g. Sync Events).
- [ ] Keep or roll back; when happy, *Deprecate stable*.

## Local

`eap/modules` and `eap/themes` are bind-mounted into the container, so **deploying locally rewrites the files in the repo** (`git status` shows it). `ddev install-portal` does not enable `eap_deployer`; enable it with `ddev drush pm:install eap_deployer`.

Code: `eap/modules/eap_deployer/src/Deployment/{Deployer,ReleaseStore,FileTree}.php`, `src/Form/DeployForm.php`, `src/Controller/DiagnosticsController.php`, schema in `eap_deployer.install`.

## Call path

<!-- call-path:begin — generated by `node .ai/scripts/ai.mjs callpaths --update`; hand edits are kept -->
Main path of each entry point (distinctive calls only). Full call tree, callers and files: `.ai/context/callgraph.json` — used by `/impact`. Edit the lines below freely: a hand-edited block is never overwritten.

- `Deployer::inspect()` → `deleteAbandonedWorkDirs()` → `FileTree::deleteTree()`
- `Deployer::apply()` → `saveRelease()` → `FileTree::listFiles()`
- `Deployer::rollback()` → `ReleaseStore::exportTo()`
- `Deployer::promote()` → `ReleaseStore::promote()` → `delete()`
- `Deployer::saveMissingStableReleases()` → `ReleaseStore::all()`
- `ReleaseStore::save()`
- `ReleaseStore::promote()` → `delete()`
- `FileTree::copyTree()`
- `FileTree::fingerprint()`
- `FileTree::unsafeEntries()`
<!-- call-path:end -->

## Key files
- `eap/modules/eap_deployer/src/Deployment/Deployer.php` — `Deployer::inspect()`, `Deployer::apply()`, `Deployer::rollback()`, `Deployer::promote()`, `Deployer::saveMissingStableReleases()`
- `src/Deployment/ReleaseStore.php` — `ReleaseStore::save()`, `ReleaseStore::promote()` · `src/Deployment/FileTree.php` — `FileTree::copyTree()`, `FileTree::fingerprint()`, `FileTree::unsafeEntries()`
- `src/Form/DeployForm.php` — `DeployForm::inspectUpload()`, `DeployForm::deploy()`, `DeployForm::rollback()`, `DeployForm::promote()`
- `src/Controller/DiagnosticsController.php` — `DiagnosticsController::view()`, `DiagnosticsController::downloadRelease()` · schema in `eap_deployer.install`

## Verification
Verified on the sandbox portal (2026-11): `eap_apim_customs` 1.2.0 and `eap_credentials_sync` 1.0.1 deployed in place, settings and sync events kept. On a new environment reload the Diagnostics tab several times: every node shows the new fingerprint.

## Related known issues
- none

## History
- 2026-11-09 (EUIPOFO-00002, `c6e60fa`): created with `eap_deployer` 1.0.0.
