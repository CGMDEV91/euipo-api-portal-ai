# euipo-api-portal — Orchestrator (the system)

<!-- Built by `node .ai/scripts/ai.mjs orchestrator --write`. Never edit inside ai:begin / ai:end: a run rewrites it.
     The rest is the team's (/learning appends gotchas) and a run keeps it. -->

euipo-api-portal — the EUIPO developer portal: IBM API Developer Portal 10.0.8.12 (Drupal 11) plus our modules and theme in `eap/`; ddev with Keycloak and `mock-apic`.

<!-- ai:begin facts -->
**Code:** Drupal 11.4 (core in `site/core`) · PHP 8.4 in ddev · `eap/modules/` (eap_api_status, eap_apim_customs, eap_credentials_sync, eap_deployer, eap_redoc) · `eap/themes/` (eap_theme) · no config sync folder · tools run in ddev
**Checks:** phpcs no phpcs.xml · phpstan no phpstan.neon
<!-- ai:end facts -->

## Services

<!-- ai:begin services -->
| Service | Folder | Type | Base branch | Gate (`workdir` = folder) | Owner |
|---|---|---|---|---|---|
| euipo-api-portal | `.` | service · Drupal 11.4 | `main` | `node .ai/scripts/ai.mjs gate` | [team — confirm] |

In-repo: the working directory is the repository, `.ai/` lives inside it (ignored by it). The facts of the service:
`.ai/services/<name>/service.md`, loaded by the ticket commands.
<!-- ai:end services -->

## Conventions common to every service

### Architecture — non-negotiable
- **IBM product, no server access:** only our `eap_*` modules, `eap_theme` and admin-UI actions reach an environment (`eap/scripts/build.sh` → test portal → sandbox → production). Before designing: `.ai/services/euipo-api-portal/context/platform-and-delivery.md`.
- Custom code only in `eap/modules/` and `eap/themes/`. **Never edit** IBM code (`apim_profile/`, `ibm_apim/`, `apic_app/`, `apic_api/`, `product/`, `consumerorg/`, `auth_apic/`, `connect_theme/`…) nor `site/` (generated): use hooks, alters, service swaps or patches.
- No `/update.php`, drush or `settings.php` in the product: no `hook_update_N`; what must exist after a deploy is an idempotent admin button, what varies per environment a settings form. Bump `version:` in `*.info.yml`. No `config/sync`: defaults in `config/install`, the rest in the seed.
- Constructor injection (`<module>.services.yml`); procedural hooks only where Drupal requires them.

### Code style
- PHP variables and parameters are camelCase (`$routeMatch`), hooks and preprocess too; array keys, route parameters, config keys and hook signatures keep their names.
- No `else` blocks: early returns (`if / elseif` is fine).
- Code comments, identifiers and docblocks are always in English, whatever the conversation language.

### Specific restrictions of this project
- **Never credit an AI tool in git history** (no `Co-Authored-By:`, no "Generated with", no AI references in commits, pull requests or tags), in this repository and in `.ai/`. Commit messages: one short line.

### Known Gotchas (system-wide)
- PHP, Composer, Drush and ddev exist only in WSL (from Windows: `wsl.exe -d ubuntu -- bash -lc '<command>'`).
- An IBM `createOrUpdate()` fed by the UI and a webhook duplicates nodes, and every `hook_apic_*_create` must be idempotent: `EUIPOFO-00002-duplicated-applications-createorupdate-race`, `EUIPOFO-00003-duplicated-consumer-orgs-createnode-race`.

## Drupal — commands and gate

<!-- ai:begin drupal -->
Drush by its real command names, never a short alias (`cr`, `cex`, `cim`, `cst`) nor a shell alias: the commands in
tickets, plans and test instructions must run for anyone. `ddev drush cache:rebuild` · `config:export` ·
`config:import` · `config:status` · `updatedb` · `pm:install <module>`.

**The gate** — one run for the automatic checks: `node .ai/scripts/ai.mjs gate` (or `/gate`). Never phpcs, phpstan or the drush checks by
hand: a check run by hand has no NEW / PRE-EXISTING split and its evidence is not the gate's.

| Check | Inside the gate | Evidence to quote |
|---|---|---|
| Coding standard | phpcs (no phpcs.xml) on the changed PHP | `phpcs: 0 new issue(s)` |
| Static analysis | phpstan (no phpstan.neon, JSON) on the changed PHP | `phpstan: 0 new error(s)` — new errors are fixed, never baselined |
| API signatures | hook implementations in the changed files vs the installed `*.api.php` | `api-signatures: 0 new mismatch(es)` — a mismatch is a TypeError waiting to happen |
| Container/caches | `drush cache:rebuild` | `Cache rebuild complete.` |
| Smoke | the paths of `.ai/context/smoke-paths.txt` (no 5xx) + no new PHP error naming a changed file | `smoke: N path(s) without 5xx, 0 new PHP error(s)` — add the pages of your flow to the file |
| Config | `drush config:status` | no differences — this ticket's own `config/sync/` files listed = **not imported yet** (`config:import`), so nothing using them was tested |
| Behaviour | the human in the browser (role · status · language) | `[VERIFIED-BY-HUMAN <date> — "<what they saw>"]` |

Every result goes to the ticket's `## Verification` with its evidence line; a check that did not run is reported as not run.

**Claims about Drupal:** a hook, interface method or event signature is copied from the installed code
(`node .ai/scripts/ai.mjs where <hook> --kind hook` gives its `*.api.php` line), never written from memory. Before
changing access, permissions or a workflow, list the handlers that already decide it
(`node .ai/scripts/ai.mjs where ENTITY_ACCESS --kind event`, `/touchpoints <permission>`). The rest of the Drupal
rules: `.ai/rules/stacks/drupal.md`, read with the service's `service.md`.
<!-- ai:end drupal -->

## Where things live

<!-- ai:begin where -->
| Need | File |
|---|---|
| The facts of ONE service (stack, modules, run commands, restrictions, gotchas) | `.ai/services/<name>/service.md` · its `context/` (architecture · data-model · api) |
| Who provides / consumes each contract; shared resources; dependencies outside the workspace | `.ai/integrations.md` |
| Live status of a piece of work (services, state, decisions, dead ends, verification) | `.ai/tickets/<Jira key>/<Jira key>.md` — written by `/analysis`, `/bugfix`, `/adaptive` |
| Has this failed before? (any service) | `.ai/known-issues/` — search with an explicit path |
| Permanent decisions, manual (integration) test procedures, technical data | `.ai/CONTEXT.md` |
| Business workflows (per actor/scenario, across services) | `.ai/skills/README.md` |
| The commands' procedures · the shared procedures (session start, intake, gate, review, session end, close) | `.ai/commands/` · `.ai/procedures/` |
| The full process and why it is like this | `.ai/METHODOLOGY.md` (humans: `.ai/HOW_TO.md`) |
| Roles: who orchestrates, who executes, who reviews | `.ai/routing.md` |
| Hard coding rules (`auth-security` · `sql-safety` · `stacks/drupal`) | `.ai/rules/` |
| What is written where, and when | `.ai/session/tracking.md` |
| Growth criteria · permissions · no-AI zones · context policy | `.ai/policies/` |
| Your own journal and time (personal, never committed) | `.ai/MEMORY.local.md` |
<!-- ai:end where -->

## Staleness — system level (each service has its own table in its `service.md`)

| Change | File to update, in the SAME ticket |
|---|---|
| Service added, removed or renamed · its base branch changed · Drupal upgraded · a custom module or theme added or removed · phpcs / phpstan configuration changed | `node .ai/scripts/ai.mjs orchestrator --write` (the generated blocks of this file) · `.ai/integrations.md` |
| Business rule added or changed | The relevant `.ai/skills/` file (or propose a new one) |
