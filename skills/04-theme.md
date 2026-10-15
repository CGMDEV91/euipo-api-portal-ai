# 04 — Theme (`eap_theme`) and how to ship styles

Read before any style, template or block-region change. Platform constraints behind these flows: `01-platform-and-delivery.md`.

---

## 1. Structure

```
eap/themes/eap_theme/
├── eap_theme.info.yml        # base theme connect_theme, regions, auto_build_scss: 1, library global-styling
├── eap_theme.libraries.yml   # global-styling → css/style.css + css/extra.css
├── eap_theme.theme           # Extra SCSS upload (theme settings) + preprocess_page (sandbox flag)
├── compile-scss.yml          # input/output used by IBM compile_scss()
├── scss/
│   ├── overrides.scss        # connect variables + ../styles/variables + connect-styles → css/style.css
│   └── mail-overrides.scss   # → css/mail.css
├── styles/                   # ★ OUR STYLES (source of truth)
│   ├── styles.scss           # entry point: @use of every partial below
│   ├── _variables.scss       # palette, font (also imported by scss/overrides.scss)
│   ├── mixins/ layout/ components/ blocks/ content/ pages/
│   └── extra.scss            # GENERATED: flattened plain CSS of styles.scss (what gets uploaded)
├── css/                      # compiled output (versioned): style.css, mail.css, extra.css
├── assets/                   # bg.png, home.svg — referenced from styles as ../assets/...
├── templates/                # page.html.twig, block--system-branding-block, block--system-menu-block--main
├── fonts/ bootstrap/ logo.svg favicon.ico screenshot.png
└── package.json              # npm run build → sass styles/styles.scss styles/extra.scss --no-source-map
```

- Sub-theme of IBM `connect_theme` (Bootstrap 3 + Carbon classes such as `bx--card`). Regions are declared again in `eap_theme.info.yml`; block placements reference those region names.
- `eap_theme_preprocess_page()` exposes `sandbox` (`eap_apim_customs.settings:is_sandbox`) → `page.html.twig` adds `sandbox-header` to the header; sandbox colours are in `styles/` under `header.sandbox-header`.

## 2. Two CSS files, two pipelines

| File | Source | Who compiles it | When |
|---|---|---|---|
| `css/style.css` (+ `css/mail.css`) | `scss/overrides.scss`: connect variables → `styles/_variables.scss` → **all of connect_theme's styles** | IBM `IbmApimThemeInstallController::compile_scss()` (scssphp, import paths from `compile-scss.yml`) | Only when the theme is **installed or set as default** (`auto_build_scss: 1`). Locally: `ddev eap-theme-css` |
| `css/extra.css` | `styles/extra.scss` = flattened `styles/styles.scss` | `_eap_theme_settings_submit()` (scssphp) on **theme settings upload**. Locally: `ddev eap-theme-css` | Whenever an admin uploads the file |

Why `style.css` is regenerated and not shipped: it is connect_theme's whole stylesheet re-themed with our variables, so it must match the connect_theme version **installed in that portal** (IBM upgrades it). Why `extra.css` exists: it holds **all EUIPO-specific styling** and can be replaced without reinstalling anything.

## 3. Why the Extra SCSS upload exists

- Changing a theme the normal way means uninstall → delete → upload → install (`01`, path A).
- **Uninstalling a theme deletes all its block placements** (`block.block.*` with `theme: eap_theme`): banners, footer, branding, menus, search… The layout then has to be rebuilt by hand / with *Refresh static content*.
- So a hook was added to the theme settings form: upload a `.scss`, compile it **on the server** with the product's scssphp, and write it to `css/extra.css` inside the theme folder. Styles change; the theme stays installed; block layout and configuration are untouched.
- The uploaded file is **pre-flattened plain CSS with `.scss` extension** because scssphp on the server can't resolve our partials/`@use` (they are not on the server in that form) and the upload validator accepts only `.scss`.

## 4. Flow 1 — change styles and ship them without redeploying (normal case)

1. **Edit partials** under `styles/**` (never edit `styles/extra.scss` or `css/*` by hand). New partial → add `@use "folder/name";` to `styles/styles.scss`; inside partials use `@use "../variables" as *;`.
2. **Compile the flattened file**:
   ```bash
   cd eap/themes/eap_theme
   npm ci            # first time (node_modules is not versioned)
   npm run build     # writes styles/extra.scss
   ```
   (Node is available in the ddev web container too: `ddev exec -d /var/www/html/eap/themes/eap_theme "npm ci && npm run build"`.)
3. **Review** `git diff styles/extra.scss` — it is the exact payload that will be uploaded. The first time, check that a build without changes produces no diff (the versioned file must match the partials).
4. **Apply locally**: `ddev eap-theme-css && ddev drush cr` → regenerates `css/style.css`, `css/mail.css`, `css/extra.css`. Check the pages affected (front, `/product`, a product, an API reference, a guide, login; sandbox mode via `eap_apim_customs` setting).
5. **Commit** partials + `styles/extra.scss` + `css/`.
6. **Upload in the target environment** (test first):
   - Log in as admin (`/user/login?registry_url=/admin`).
   - **Appearance › Settings › eap_theme** → `/admin/appearance/settings/eap_theme`.
   - *API Portal theme settings* → **Extra SCSS** → choose `styles/extra.scss` → **Save configuration**.
   - The server writes `css/extra.css`. If the change isn't visible: **Configuration › Performance › Clear all caches** (CSS aggregation) and hard-reload the browser.
7. **Verify** on the environment, then repeat step 6 in sandbox and production.

Limits of this flow:
- Only `css/extra.css` changes. Variables used by `style.css` (`styles/_variables.scss` through `scss/overrides.scss`) won't change connect_theme's base styles until `style.css` is recompiled (theme reinstall / set default, or Flow 2).
- **Images/fonts are not uploaded**: `url(../assets/x.png)` resolves to the theme folder on the server. A new asset needs Flow 2, or **better**, an absolute URL to a file uploaded to the portal's Media library.
- Templates (`templates/*.twig`) and `eap_theme.theme` changes are **not** covered — they need Flow 2.
- After a theme reinstall, `css/extra.css` does not exist until the Extra SCSS is uploaded again.

## 5. Flow 2 — full theme update (templates, assets, PHP, base variables)

Only when Flow 1 can't cover the change. Today the only mechanism is the **UI reinstall** (`01`, path A), which wipes the block layout:

1. Do Flow 1 steps 1–5, bump `version:` in `eap_theme.info.yml`, run `eap/scripts/build.sh`, and commit `eap/build/eap_theme.tar.gz`.
2. On the test portal, take a screenshot of **Structure › Block layout** for `eap_theme`.
3. Set another theme as default, uninstall `eap_theme`, delete it (`/admin/appearance/delete`), upload the tarball (`/admin/theme/install`), install it and set it as default → the product compiles `style.css`/`mail.css`.
4. Upload the Extra SCSS (Flow 1 step 6) — `css/extra.css` is not in the package.
5. *Refresh static content* in `eap_apim_customs`, then re-place every other block by hand from the screenshot.
6. Verify, then repeat in sandbox/production.

Because of this cost, prefer designs that keep theme changes inside `styles/**` (Flow 1). Put markup changes in module templates/blocks rather than theme templates where possible.

## 6. Local development loop

- `ddev eap-theme-css` after any SCSS change (it compiles style/mail like the product, and extra from `styles/extra.scss` — so run `npm run build` first if you edited partials).
- Disable aggregation while working: `ddev drush cset system.performance css.preprocess 0 -y`.
- Changes under `eap/themes` are live in the container (bind mount); `ddev drush cr` for Twig/PHP/info changes.

## 7. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Upload saved but no visual change | Cache/aggregation → clear all caches; check `css/extra.css` timestamp is new (locally) |
| Error on save, page broken | The uploaded file doesn't compile with scssphp (e.g. uploaded `styles.scss` with `@use`) → upload the **flattened** `styles/extra.scss` |
| Background images missing | `../assets/...` not present in the deployed theme folder → Flow 2 |
| Base styles missing after a reinstall | `compile_scss()` only runs on install / set default → set `eap_theme` as default again |
| EUIPO styles missing after a reinstall | `css/extra.css` is not in the package → upload the Extra SCSS |
| Banners/footer gone | Theme was uninstalled → *Refresh static content* (`05-apim-customs.md`) + re-place other blocks |

## 8. Related

- Static blocks placed in `eap_theme` regions: `05-apim-customs.md`.
- Operator doc: `eap/docs/euipo-theme-installation.md` (does not describe the Extra SCSS flow yet).
