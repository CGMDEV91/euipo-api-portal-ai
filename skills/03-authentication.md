# 03 — Authentication (user login)

Read before touching the login form, user registries, or anything about how people sign in. For **application credentials** registered in the IdP see `02-credentials-sync.md` — a different flow.

## Two kinds of accounts

| Account | Login | Where it is checked |
|---|---|---|
| Drupal admin (uid 1) | `/user/login?registry_url=/admin` (hidden "admin registry") | Drupal DB — `auth_apic/src/Form/ApicUserLoginForm.php::validateApicAuthentication()` only lets uid 1 in this way |
| Everybody else | **EUIPO Account** button (OIDC user registry) | API Connect → EUIPO identity provider |

## OIDC flow (production and local)

```
Browser → Portal "EUIPO Account" → API Connect consumer-api /oauth2/authorize → EUIPO IdP login
       ← IdP → API Connect callback → portal /ibm_apim/oauth2/redirect?code=<apic code>
Portal → API Connect /token + /me?expand=true → user, consumer orgs, roles → Drupal user created/updated
```

- **The portal never talks to the IdP directly**; API Connect brokers the login. The OIDC registry (`production-oidc`) is configured in API Connect, not in our modules.
- The IdP historically is Apereo CAS acting as OIDC provider; the target is a standard OIDC/OAuth2 provider. For user login the change happens in the API Connect registry configuration; our code only depends on the registry being OIDC.
- Consumer-org roles (owner/administrator/developer/viewer) come from API Connect `/me`; Drupal roles (`content_author`, …) are granted by an admin in Drupal.
- Self-onboarding is off: a user without a consumer org lands on "No permission".

## What our code changes (`eap_apim_customs`)

`eap_apim_customs_form_user_login_form_alter()` — unless setting `all_register_options` is on:
- Header text → "Sign in with your EUIPO account".
- OIDC button label → "EUIPO Account" (only when `main_container.plainlogin.oidc_link` exists — it doesn't on the admin registry form; BUG fixed in 1.1.1, see `eap/docs/dev-documentation/09-troubleshooting.md`).
- Hides other registries (`main_container.other`, `or_container`).
- Forces state `ibm_apim.selfSignUpEnabled` = FALSE (TRUE when `all_register_options` is on). Note: it writes state **on every login form build**.

Related product settings (`/admin/config/system/ibm_apim`, see `eap/docs/site-configuration.md`): *Hide the admin registry* on; *OIDC register form* and *OIDC login form* off.

## Why it is done this way

| Decision | Reason |
|---|---|
| Login form altered in a module, not configured | IBM renders the registry list and labels from API Connect config; the "EUIPO Account" wording and hiding the other registries are not settings of the product |
| Self sign-up forced off from the login form alter (state `ibm_apim.selfSignUpEnabled`) | That value is Drupal **state** rewritten by IBM whenever API Connect pushes the catalog config (`ibm_apim/src/Service/SiteConfig.php`), not editable config; resetting it whenever the login form is built guarantees the EUIPO policy (accounts come from the IdP + API Connect orgs) even if the platform rewrites it |
| `all_register_options` setting | *(inferred)* Lets an admin temporarily show every registry (e.g. local registry, troubleshooting) without redeploying |
| Admin keeps the hidden `?registry_url=/admin` login | Only uid 1 can log in locally (IBM rule); it is the only way to administer the portal if the IdP is down |
| No IdP code in our modules for user login | API Connect brokers OIDC; the portal only receives an APIC code. Changing the IdP (CAS → OIDC) is an API Connect registry change, not a deployment |

## Local equivalents

Keycloak (realm `euipo`) = IdP, mock-apic = API Connect consumer-api. Users, clients and troubleshooting: `eap/docs/dev-documentation/03-users-and-login.md`, `04-keycloak.md`, `11-credentials.md`.

## Testing in the test environment

Real login can only be verified on the test portal connected to API Connect + IdP: upload the package, log in with a test EUIPO account, check landing page / org / roles, and the admin login via `?registry_url=/admin`.
