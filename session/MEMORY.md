# MEMORY.md — EUIPO API Portal Session History

This file is maintained automatically by the agent. One entry per session. Never delete entries — only append. Newest entry at the bottom.

---

## Session 2026-10-15

**Summary**: Bootstrapped the `.ai/` knowledge base for the EUIPO API Portal (adapted from giportal) and documented the custom modules, the theme and the IBM platform constraints as skills.

**Work done**:
- Cleaned the giportal copy: removed scan scripts, giportal context files and workflow skills; rewrote orchestrator, README, `example.CLAUDE.md`/`example.AGENTS.md` and the 5 agents for this project (IBM devportal 10.0.8.12, Drupal 11, `eap/` modules + `eap_theme`).
- Created skills 01–06: platform & delivery, credentials sync / Sync Events, authentication, theme & Extra SCSS, `eap_apim_customs`, `eap_redoc` + `eap_api_status`. Each has a "why" section; reasons not backed by docs are marked *(inferred)*.
- Verified in IBM code: `/update.php` removed (`IbmApimRouteSubscriber`), Update Manager refuses to upload an existing extension, custom extensions deletable only when uninstalled → upgrade = uninstall → delete → upload → install.
- Confirmed with the user: styles ship **only** via the Extra SCSS upload (`npm run build` → `styles/extra.scss` → `/admin/appearance/settings/eap_theme`); no full-folder overwrite exists today (kept as "not used today" in skill 01).
- `.ai/` published as its own repo: git@github.com:CGMDEV91/euipo-api-portal-ai.git (excluded from the main repo via `.git/info/exclude`).

**Files modified**:
- `.ai/**` — whole knowledge base.
- `.git/info/exclude` (main repo, local only) — excludes `/.ai/`, `/CLAUDE.md`, `/AGENTS.md`.

**Pending / Next steps**:
- Update stale operator docs: `eap/docs/module-eap-appcreds-sync.md` (old CAS module) and `eap/docs/euipo-theme-installation.md` (no Extra SCSS flow).
- Confirm with the user: CAS → OIDC scope for user login (assumed: API Connect registry change only); inferred reasons in skills.
- Review risks noted in skill 02: client secret stored in plain text in `sync_event.payload` and shown on the detail page.
- Check that `npm run build` with no changes reproduces the versioned `styles/extra.scss`.
- Copy `example.CLAUDE.md`/`example.AGENTS.md` to the repo root when the user wants them active.
