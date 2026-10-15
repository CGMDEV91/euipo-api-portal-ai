# 02 — Credentials sync (`eap_credentials_sync`) and the Sync Events log

Read before touching anything related to applications, credentials, the identity provider registration, or the **Sync Events** report that the identity team uses.

## What it does (one paragraph)

When a developer creates, edits, deletes an application or resets its secret in the portal, API Connect issues/changes the app's `client_id`/`client_secret`. Those credentials must also exist as an **OIDC client in the EUIPO identity provider** so the app can obtain tokens. This module records each change as a **sync event** (outbox), sends it to the IdP's registration API authenticated with an OAuth2 `client_credentials` token, and stores the outcome. The **Sync Events report** (`/admin/reports/sync-events`) is the log another team reads to know whether an application's credentials were **registered correctly in the authentication service or failed, and why**.

> Note: "user registered in the auth service" in this project means **an application's credentials registered as an OIDC client**, not a person's login. Person login is a different flow (`03-authentication.md`).

## Why it exists and why it is built this way

**The problem.** API Connect issues the app credentials, but the EUIPO identity provider (which issues the access tokens API consumers use) knows nothing about them. Nothing in the IBM product pushes clients to an external IdP, and we can't change API Connect. The only place we control that sees every app change is the **portal**, through the `apic_app` hooks — so the portal propagates the credentials.

**History.** The previous module (`eap_appcreds_sync`) registered apps in CAS. According to `eap/docs/module-dblog-persistent.md`, CAS bugs frequently broke the synchronization: apps existed in API Connect/Gateway but not in the authorization server, so consumers got credentials that could not obtain tokens. With no server access, the workaround was the `dblog_persistent` contrib module, keeping Drupal log entries long enough to recover the data and fix the registration by hand. The outbox + report of the current module replaces that manual recovery.

**Design decisions in the current module** (reasons marked *(inferred)* come from the code and platform constraints, not from written docs):

| Decision | Reason |
|---|---|
| **Outbox entity** (`sync_event`) written before calling the IdP | A failed call is never lost: the intent and its payload stay in the portal DB |
| Immediate attempt + **cron retries** up to `max_attempts` | *(inferred)* Most registrations succeed instantly (developer can use the app right away); transient IdP failures heal themselves |
| **Manual *Synchronize*** ignoring the limit | Operators fix the cause (settings, IdP) and replay from the UI — no DB/drush access exists |
| **Report in the admin UI** with HTTP code, URL, body and `WWW-Authenticate` | The identity team and operators diagnose without server logs; the message must be self-explanatory |
| Payload keeps the **secret** | `create`/`reset` can only be replayed with the secret; API Connect won't return it again |
| Retention purge of `success` events | *(inferred)* Bound the table and the number of stored secrets |
| **Domain / Application / Infrastructure** layering with `CredentialsRegistryInterface` | The IdP changes (CAS → OIDC): only `OidcHttpClient`/`OAuthTokenProvider` know the wire protocol; the rest is unit-tested without HTTP |
| All endpoints/credentials in a **settings form**, secret write-only | No `settings.php` or secret store in the product; the same package serves test/sandbox/production |
| `processing_enabled` off by default | *(inferred)* A freshly uploaded module must not call an unconfigured/wrong IdP before an admin sets the endpoints |
| *Import Sync Events config* button | The product has no `drush cim`/`update.php`: the shipped view/action can be (re)applied from the UI on an existing install |
| *Delete all sync events* button | Drupal refuses to uninstall a module whose content entities exist; upgrading through the UI requires uninstalling (`01`) |
| Redirect-URL uniqueness validator | *(inferred)* The IdP rejects clients with an identical redirect set; failing in the portal form gives the developer the error instead of a silent sync failure |
| Plain `curl` with TLS verification | *(inferred)* Full control over the raw headers/body recorded in the message; no extra dependency |

## CAS → OIDC

- **Before:** module `eap_appcreds_sync` talked to Apereo CAS: `POST …/cas-server-webapp/register`, token from `…/oidc/accessToken`, optional client certificates for SecureGateway. Still described in `eap/docs/module-eap-appcreds-sync.md` (**stale** — describes the old module, path and settings).
- **Now:** `eap_credentials_sync` is IdP-agnostic: a generic registration endpoint (`applications_endpoint`) + standard OAuth2 `client_credentials` token (`auth_token_endpoint`, `auth_client_id`, `auth_client_secret`, `auth_scope`). No certificates.
- The **wire contract** (below) still uses CAS-style parameter names (`ext_param_client_id`, `ext_param_client_secret`). Any change of target IdP must be agreed with the IdP team and is isolated in `Infrastructure/Http/OidcHttpClient.php`.

## Trigger points (IBM `apic_app` hooks)

| IBM hook (fired by) | Our implementation | Operation | Payload |
|---|---|---|---|
| `hook_apic_app_create` (`ApplicationService::invokeAppCreateHook`, also from the *update* path when the event is `create`/`app_create`) | `eap_credentials_sync_apic_app_create` | `create` | clientName = `<app name>-<app id>`, redirectUrls, clientId, **clientSecret** |
| `hook_apic_app_update` (`ApplicationService::update` on any non-internal event when the node hash changed — includes webhooks/content refresh — and `ApplicationUpdateForm`) | `eap_credentials_sync_apic_app_update` | `update` | clientName, redirectUrls, clientId of the **first** credential (no secret) |
| `hook_apic_app_pre_delete` (`ApplicationService::deleteNode`) | `eap_credentials_sync_apic_app_pre_delete` | `delete` | clientId of the first credential; `app_name` = node `application_name` |
| `hook_apic_app_clientsecret_reset` (`ResetClientSecretForm`) | `eap_credentials_sync_apic_app_clientsecret_reset` | `reset` | clientName, redirectUrls, clientId, **new clientSecret** |

Gotchas:
- `apic_app_pre_delete` is invoked with **string keys** (`['node' => …, 'data' => …]`) → PHP passes them as **named arguments**: the parameters must be named exactly `$node` and `$data`.
- **Not handled** (by design or gap — confirm before adding): `apic_app_clientid_reset`, `apic_app_creds_create/update/delete` (additional credentials). Production disables "reset Client ID" and "additional credentials" in `/admin/config/system/ibm_apim` (`eap/docs/site-configuration.md`), which is why only the first credential is synced.
- Missing data (no secret / no credentials) → no event, only a `warning` in the Drupal log (`eap_credentials_sync` channel). Those cases **won't appear** in the Sync Events report.

Also: `hook_form_alter` on `application_create_form` / `application_update_form` adds `RedirectUrlValidator`: the **set** of redirect URLs (normalised: empty removed, sorted) must not equal another application's set → error "Invalid redirect URL." (the IdP rejects duplicated redirect sets).

## Flow

```
IBM hook ─► SyncEventRecorder::recordX()  ─► sync_event (status=pending, payload JSON, attempts=0)
                                     │
             processing_enabled? ────┤ yes → _eap_credentials_sync_process_now(): immediate attempt in the same request
                                     │
hook_cron ─► SyncEventProcessor::processPending(maxAttempts, batchSize)   (status ≠ success AND attempts < max, oldest first)
                   └► CredentialSynchronizer ─► OidcHttpClient ─► IdP
                         └► markSuccess / markFailure(attempts+1), message = "[Y-m-d H:i:s] HTTP <code> on <VERB> <url>: <body>"
hook_cron ─► SyncEventCleaner::clean(): deletes SUCCESS events older than success_retention_days
```

Layers (DDD-style, keep them): `Domain/` (models, enums, interfaces) · `Application/` (recorder, processor, synchronizer, cleaner, redirect validator) · `Infrastructure/` (HTTP client, token provider, Drupal repository, config importer) · `Entity/`, `Form/`, `Controller/` (Drupal UI). Unit tests in `tests/src/Unit` cover the Application layer and `Credentials`.

## Wire contract (`OidcHttpClient`)

| Operation | Request |
|---|---|
| create | `POST {applications_endpoint}` body `{client_name, redirect_uris, ext_param_client_id, ext_param_client_secret}` |
| update | `PUT {applications_endpoint}/{clientId}` body without the secret |
| delete | `DELETE {applications_endpoint}/{clientId}` |
| reset | `DELETE …/{clientId}` (result ignored) then `POST` with the new secret — the event's result is the POST's |

- Header `Authorization: Bearer <token>` from `OAuthTokenProvider` (`grant_type=client_credentials`, memoised per request).
- Endpoint must start with `https://`; TLS verified.
- Success = HTTP < 400. Transport errors (DNS, TLS, token failure) → failure with the exception message.
- On 401/403 the `WWW-Authenticate` header is appended to the message: `invalid_token` (token rejected) vs `insufficient_scope` (portal client lacks privileges).

## The Sync Events report (what the other team reads)

- **List:** `/admin/reports/sync-events` (View `sync_events`, shipped in `config/optional`). Columns: App name, Operation, Status (coloured badge), Attempts, Created, Updated, Operations. Filters: app name, operation, status, created from/to. Bulk delete.
- **Detail:** `/admin/reports/sync-events/{id}` — all fields + payload table + *Synchronize* / *Delete* buttons.
- **Statuses:** `pending` (recorded, not sent yet or processing disabled) · `success` · `failure` (retried by cron until `max_attempts`; after that it stays `failure` and only a manual *Synchronize* retries it).
- **Reading a failure:** the `message` holds the timestamp, HTTP code, verb, URL and IdP body (or "empty response body"), e.g. `HTTP 409 on POST https://…/register: client already exists`.
- Manual *Synchronize* ignores the attempt limit.
- Permission everywhere: `administer site configuration`.

## Settings (`/admin/config/eap_credentials_sync/settings`, config `eap_credentials_sync.settings`)

`applications_endpoint`, `auth_token_endpoint`, `auth_client_id`, `auth_client_secret` (blank = keep), `auth_scope`, `processing_enabled` (default off → events pile up as pending), `cron_batch_size` (50), `max_attempts` (5), `success_retention_days` (30). Maintenance: **Import Sync Events config** (re-applies view + bulk action on an existing install) and **Delete all sync events**.

No `config/install` defaults: every value falls back in code (`?? default`). A fresh install (including every UI upgrade, which reinstalls) needs the settings filled in by hand — keep the per-environment values documented in `eap/docs/` (`module-eap-appcreds-sync.md` still lists the old CAS values and must be updated).

## Known risks / review points

- The payload stores the **client secret in plain JSON** in `sync_event.payload` and the detail page shows it to administrators. Success events are purged after the retention period; **failed and pending events keep it indefinitely**.
- Immediate processing runs **inside the user's request** (app create/reset waits for the IdP).
- `hook_apic_app_update` fires on webhook/content-refresh updates too → frequent `update` events are expected, and a `create` can arrive via the update path.
- `reset` deletes then re-registers: if the POST fails, the client is missing in the IdP until a retry succeeds.

## Local testing

Seed points the module to mock-apic `/cas/register` (which creates real Keycloak clients) with Keycloak tokens (client `eap-credentials-sync`). Steps and failure simulation: `eap/docs/dev-documentation/05-mocks.md` → *End-to-end application flow*. Fake rows for the listing: Devel Generate plugin `sync_event` (only if `devel_generate` is enabled).

## Testing in the test environment (real API Connect + IdP)

1. Deploy `eap_credentials_sync` (see `01-platform-and-delivery.md`). Through the UI path this means uninstall (after *Delete all sync events* — **export/note pending and failed events first, they are lost**) → delete → upload → enable.
2. Fill the settings with the test IdP values; enable processing.
3. Log in as a developer of a consumer org and generate each event: **create app** → `create`; **edit app / redirect URLs** → `update`; **reset client secret** → `reset`; **delete app** → `delete`.
4. Check each row in `/admin/reports/sync-events` (status + message) and confirm with the identity team that the client exists / was updated / removed and that a token can be obtained with the app credentials.
5. For failures: fix settings, then *Synchronize* the event (or wait for cron).
