# 05 — Portal customizations (`eap_apim_customs`)

Small overrides of IBM portal features plus the homepage/static blocks. Login-form changes are in `03-authentication.md`.

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
| **Static blocks from Twig templates + "Refresh static content"** | Block content and placements are configuration/content in the DB, not deployable code, and **they are deleted when the theme is reinstalled** (`01`, `04`). Keeping their HTML in the module and recreating them idempotently with a button makes the layout reproducible in any environment and after any theme reinstall. It also deletes `block.block.eap_theme_branding` / `eap_theme_footer`, the copies Drupal creates from the base theme when `eap_theme` is installed |
| Standalone Twig environment for those templates | They are rendered once into `block_content` bodies (`full_html`), outside a request render pipeline |

## Settings (`/admin/config/eap_apim_customs/settings`, config `eap_apim_customs.settings`)

| Key | Effect |
|---|---|
| `is_sandbox` | Sandbox mode: no "Sandbox" link in main menu, products banner without the sandbox paragraph, `sandbox` var in `page.html.twig` |
| `all_register_options` | Show every login registry and allow self sign-up (off in production) |
| `data_protection_link` | Footer "Data protection" link |
| `product_use_assessment_link` | "Conditions of use" link (subscription checkbox + footer) |

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
