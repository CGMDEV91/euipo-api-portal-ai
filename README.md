# EUIPO API Portal — AI Integration

Vendor-neutral AI knowledge base for the EUIPO API Portal local project (IBM API Developer Portal 10.0.8.12 + EUIPO custom modules/theme).

It is the **single source of truth** for what an AI coding assistant needs on this project: conventions, agent definitions, on-demand context and session history. Every AI tool is pointed here through a thin entry file in the repo root (`CLAUDE.md`, `AGENTS.md`), which redirects to [`orchestrator.md`](orchestrator.md).

## Getting started

This repo lives at `euipo-api-portal/.ai/`, nested inside the project checkout as its own git repository (excluded locally via the project's `.git/info/exclude`, together with the root `CLAUDE.md`/`AGENTS.md`, so the two histories never mix).

1. From the `euipo-api-portal` root, clone it **directly into `.ai`**:
   ```bash
   git clone git@github.com:CGMDEV91/euipo-api-portal-ai.git .ai
   ```
2. Copy the entry-point files to the project root, dropping the `example.` prefix:
   ```bash
   cp .ai/example.CLAUDE.md CLAUDE.md
   cp .ai/example.AGENTS.md AGENTS.md
   ```
3. Open an AI session in the project — the assistant reads `CLAUDE.md`/`AGENTS.md`, which redirects to `.ai/orchestrator.md`.

To get updates (skills, context, MEMORY/BUGLOG history): `cd .ai && git pull`.

## Structure

```
.ai/
├── orchestrator.md          # Entry point — read first, every session
├── example.CLAUDE.md        # Snapshot of the root CLAUDE.md fallback
├── example.AGENTS.md        # Snapshot of the root AGENTS.md fallback
├── agents/                  # backend-architect, code-reviewer, context-manager,
│                            # fullstack-developer, prompt-engineer
├── skills/                  # 00-session-tracking.md (+ future skills), README.md index
├── context/                 # On-demand architecture/reference docs (starts empty)
└── session/                 # MEMORY.md (session log), BUGLOG.md (bug history)
```

## How it works

1. The assistant reads `CLAUDE.md`/`AGENTS.md`, which points to the orchestrator.
2. The orchestrator holds environment facts, critical conventions, commands and a table of reference files.
3. Deeper files (`skills/`, `context/`, `eap/docs/`) are pulled in **on demand**, keeping sessions token-efficient.
4. Session start/end syncs `MEMORY.md` (what was done) and `BUGLOG.md` (what broke and why).

## Keeping it in sync

- Changes to conventions/commands/session rules → update `orchestrator.md` first, then propagate per its *Entry-point sync rule*.
- New recurring business rule or bug pattern → add a `skills/` or `context/` file and log it in `BUGLOG.md`.
- A new module-scoped agent is a deliberate decision (recurrence + specialisation + payoff), never speculative.
