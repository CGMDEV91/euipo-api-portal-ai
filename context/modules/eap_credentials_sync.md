# eap_credentials_sync — module card (generated: `node .ai/scripts/ai.mjs integrations --update`)

<!-- notes -->
_Purpose in 2–3 lines, written by a human._
<!-- /notes -->

164 functions/methods indexed — `where` / `callers` for details.

**Services** (9): `eap_credentials_sync.credential_synchronizer` → CredentialSynchronizer · `eap_credentials_sync.oauth_token_provider` → OAuthTokenProvider · `eap_credentials_sync.oidc_http_client` → OidcHttpClient · `eap_credentials_sync.redirect_url_validator` → RedirectUrlValidator · `eap_credentials_sync.sync_event_cleaner` → SyncEventCleaner · `eap_credentials_sync.sync_event_config_importer` → SyncEventConfigImporter · `eap_credentials_sync.sync_event_processor` → SyncEventProcessor · `eap_credentials_sync.sync_event_recorder` → SyncEventRecorder · `eap_credentials_sync.sync_event_repository` → DrupalSyncEventRepository

**Routes** (4): `eap_credentials_sync.config_form` /admin/config/eap_credentials_sync/settings · `eap_credentials_sync.delete_all_events` /admin/config/eap_credentials_sync/delete-all-events · `entity.sync_event.canonical` /admin/reports/sync-events/{sync_event} · `entity.sync_event.synchronize` /admin/reports/sync-events/{sync_event}/synchronize

**Hooks** (10): hook_apic_app_clientsecret_reset · hook_apic_app_create · hook_apic_app_pre_delete · hook_apic_app_update · hook_cron · hook_form_alter · hook_form_views_exposed_form_alter · hook_preprocess_views_view · hook_preprocess_views_view_field · hook_views_data_alter

**Integrations** (1): cron (hook_cron): eap_credentials_sync_cron() (`cron:eap_credentials_sync`)
