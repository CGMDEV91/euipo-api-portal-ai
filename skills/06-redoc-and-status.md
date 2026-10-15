# 06 — API docs rendering (`eap_redoc`) and status page (`eap_api_status`)

## `eap_redoc` — Redoc instead of IBM's API explorer

### Why

IBM's built-in API explorer was considered unclear for EUIPO's OpenAPI specs (`eap/docs/module-redoc.md`). The explorer is part of IBM's `apic_api` module and its templates, which we can't patch, so the module:
- keeps IBM's page and data (the spec stored in the `api_swagger` field of the `api` node, synced from API Connect),
- passes the spec to the browser via `drupalSettings` and lets **Redoc render it client-side** into IBM's `.explorerWrapper` container,
- ships `redoc.standalone.js` **inside the module** (no npm/composer/CDN dependency on the product),
- links API reference ↔ guide using IBM's own `apiref` / `prodref` fields on `page`, so admins relate guides to APIs with the standard page form (`eap/docs/content-creation.md`).

### How

`hook_entity_view_alter`:

- **API node (`api`, full):** unserializes `api_swagger` → `drupalSettings.eap_api`; attaches library `eap_redoc` (bundled `js/redoc.standalone.js` + `eap_redoc.js`), which runs `Redoc.init()` on `.explorerWrapper`. If a `page` references this API (`apiref` field), renders `templates/apiref-header.html.twig` (Reference / Guide tabs) into `drupalSettings.docsHeader`, injected into `.leftTitle`.
- **Page node (`page`, full) with `apiref` + `prodref`:** renders `templates/docs-header.html.twig` (breadcrumb Products › product › API + version/state badge + Reference/Guide tabs) into `drupalSettings.docsHeader`, injected before `.apicMainContent` by `eap_redoc_doc.js`. If the body format is `ghmarkdown`, adds class `markdown-guide` (used by the TOC in `eap_apim_customs`).

Notes: `apiref`/`prodref` are IBM `apim_profile` fields on `page` (comma-separated ids — first one used). Templates are rendered with a standalone Twig env (no Drupal filters / autoescape context) — escape carefully if adding user-provided values. `eap_redoc.js` has a leftover `console.log(settings)`.

Operator doc: `eap/docs/module-redoc.md`.

## `eap_api_status` — `/status`

### Why

API users need to see live availability of sandbox and production APIs. Monitoring is done in EUIPO's **PRTG**, which publishes a public sensor map; the portal only embeds it. *(inferred)* A module instead of a hand-made page gives a fixed route, the main-menu link on install and a settings form for the map URL, with no content to recreate per portal. The same map shows both catalogues, so the same URL is used in sandbox and production (`eap/docs/module-status-page.md`).

### How

- Route `/status` (`access content`), main-menu link "Status" (weight 90).
- `StatusController::content()` renders title, legend (Running/Warning/Error/Maintenance) and an **iframe** to `eap_api_status.settings:monitor_iframe_url` (PRTG public map), `max-age 0`.
- Settings: `/admin/config/eap_api_status/settings`.

Operator doc: `eap/docs/module-status-page.md`.
