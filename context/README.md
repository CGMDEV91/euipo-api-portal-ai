# Context

Architecture, conventions and reference material, read **on demand** (never front-loaded).
Add a file here only when a topic has proven worth documenting, then
register it in the Reference Files table of [`../orchestrator.md`](../orchestrator.md).

| File | Contents |
|---|---|
| [`product-overview.md`](product-overview.md) | Business model (free products/plans/subscriptions), actors (API Connect, Gateway, IdP, portal), customer journey, Sandbox vs Production, where each custom module fits |

Candidate topics (create when needed, not before):

(Module and theme knowledge lives in `.ai/skills/01`–`06`.)

- `ibm-hooks.md` — catalogue of IBM hooks/events available to custom modules
- `ibm-integration.md` — extension points of the IBM devportal and what must not be touched
- `local-dev.md` — Keycloak / mock-apic / seed internals

Functional and local-dev documentation already exists in `eap/docs/` (see orchestrator) —
link to it instead of duplicating it.
