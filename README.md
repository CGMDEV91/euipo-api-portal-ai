# euipo-api-portal — AI knowledge base

The knowledge AI agents use on the EUIPO API Portal (IBM API Developer Portal 10.0.8.12 + the EUIPO custom modules and theme in `eap/`): the system, the service, the tickets (one folder each, with one diary per developer), the known issues and the skills. It is its own git repository inside the project, ignored by the project (`.git/info/exclude`): nothing from it is ever committed to the project.

| Path | What |
|---|---|
| `.ai/orchestrator.md` | The system: conventions common to the project, the services table, the Drupal commands and the gate |
| `.ai/services/euipo-api-portal/service.md` and `context/` | The facts of the service, its gotchas and the technical background (IBM platform constraints and delivery, the theme and its CSS pipelines, the product overview) |
| `.ai/skills/` | Workflows of the portal: credentials sync to the identity provider, user login, the portal customizations, Redoc and the status page, the in-place deployer |
| `.ai/known-issues/` | One folder per confirmed issue — search it before debugging |
| `.ai/tickets/` | One folder per Jira ticket: the shared file and the diaries |
| `.ai/CONTEXT.md` | Permanent decisions, manual test procedures, technical data |
| `.ai/product-overview_ES.md` | The product guide for humans, in Spanish |
| `.ai/integrations.md` · `.ai/context/` | Generated: integrations, module cards, code and API indexes |
| the rest (`commands/`, `procedures/`, `rules/`, `scripts/`, `install/`, `templates/`…) | The ai-core framework, replaced by every `update-ai-framework` |

## Start working: clone it as `.ai`

With ai-core installed on your machine it is one command, which also asks for the agent (opencode, Claude Code or both). Run it at the project root; the `.ai` folder must not exist:

```bash
install-ai-framework --remote https://github.com/CGMDEV91/euipo-api-portal-ai.git
```

Without ai-core, by hand. The project's own repository is cloned as usual; this one is cloned **into a folder named `.ai` at its root**. Nothing else is needed from the team: the framework, the tools and the project's `project.json` are in this repository.

```bash
cd <project folder>                                   # the root of euipo-api-portal, where build-local-site.sh is
git clone https://github.com/CGMDEV91/euipo-api-portal-ai.git .ai
echo ".ai/" >> .git/info/exclude                      # the project's git never sees it (local, the project's .gitignore is not changed)
cp .ai/hooks/pre-commit-guard.sh .ai/.git/hooks/pre-commit && chmod +x .ai/.git/hooks/pre-commit   # blocks secrets and AI files in commits
bash .ai/install/install-opencode.sh                  # once per machine, if you use opencode
node .ai/scripts/ai.mjs claude install && node .ai/scripts/ai.mjs claude project   # if you use Claude Code
node .ai/scripts/ai.mjs detect --index                # builds the local indexes (they are not in the repository)
node .ai/scripts/ai.mjs doctor                        # one table, one fix per problem
```

The repository is private: use your GitHub credentials, or its SSH URL if you have a key set up.

Then open opencode (or `claude`) in the project folder, never in a subfolder, run `/doctor`, and `ddev start` before the first `/gate`. `/start` is not needed: the knowledge base exists. Keep it current with `git -C .ai pull` before you start a ticket, and commit and push `.ai` when you close one (the agent never does).

Day to day: `/analysis <KEY>` then `/bugfix <KEY>` for a bug, `/adaptive <KEY>` for a change. The old `session/MEMORY.md` and `session/BUGLOG.md` are now the diaries of the tickets and the known issues; the old `CLAUDE.md` / `AGENTS.md` fallbacks and the five generic agents are gone (the framework writes `CLAUDE.md` and has its own agents).
