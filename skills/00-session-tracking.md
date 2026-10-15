# Session Tracking — Agent Behavior

This is a **meta/agent-behavior skill** (not a business workflow skill): it defines how the agent maintains `.ai/session/MEMORY.md`, `.ai/session/BUGLOG.md`, and — when the session surfaces knowledge worth keeping — how it proposes creating or updating a skill in this same folder.

Numbered `00` so it sorts before any future skills and is visibly distinct from them in listings.

---

## On Every Conversation Start

1. Read `.ai/session/MEMORY.md` — scan the last 2–3 entries to restore context.
2. Read `.ai/session/BUGLOG.md` — note any open/recurring bugs.
3. If either file does not exist, create it with the format defined below.

---

## Classifying Incoming Requests

Before responding to any request, silently classify it:

| Classification | Signals |
|---|---|
| **Bug / problem** | "not working", "wrong", "broken", "mix", "incorrect", "fails", "error", "issue", "doesn't render", "missing", "bug" — or a description of unexpected behaviour |
| **Implementation** | "add", "create", "implement", "build", "update", "change", "refactor", "generate" — or a feature/change request |
| **Question / analysis** | "what", "why", "how", "explain", "review", "check", "diagnose" — no code change requested |

---

## If the Request is a Bug / Problem

1. **Check BUGLOG.md first.** Search for entries with similar symptoms, affected files, or the same module or theme. If a related entry exists, mention it to the user before starting: _"I found a related entry in BUGLOG: BUG-NNN — [title]. Let me check if the pattern applies here."_

2. **Quick code scan.** Read the relevant files (module, template, config) before proposing a fix. Do not guess.

3. **Apply the fix.** Explain what was wrong and what was changed.

4. **Ask the user if it worked — in the same language they used.** Examples:
   - English: _"Does the fix work as expected? Let me know and I'll log it."_
   - Spanish: _"¿Ha funcionado la solución? Dímelo y lo registro."_
   - French: _"Est-ce que le correctif fonctionne ? Dis-moi et je l'enregistre."_

5. **If the user confirms it worked** → write to BUGLOG (see below), then run the "Evaluate a skill update" step.

6. **If the user says it did not work** → continue debugging. Do not write to BUGLOG until confirmed fixed.

---

## If the Request is an Implementation

1. Implement the feature following project conventions.

2. When done, ask the user for confirmation — in their language. Examples:
   - English: _"Implementation done. Does everything look good? I'll add a summary to the session log."_
   - Spanish: _"Implementación lista. ¿Todo correcto? Añado el resumen al log de sesión."_

3. **If the user confirms** → write to MEMORY.md (see below). Only write to BUGLOG if the implementation also fixed a known bug. Then run the "Evaluate a skill update" step.

---

## Evaluate a Skill Update (after any confirmed fix or implementation)

Once the user has confirmed the work is correct, before closing out, check whether it changed or revealed something a skill in `.ai/skills/` or a context file should reflect:

- A behaviour documented in `.ai/skills/` or `.ai/context/` changed → that file is now stale.
- A new business rule, edge case, or gotcha was discovered that isn't documented anywhere in `.ai/skills/` or `.ai/context/`.
- A new recurring pattern emerged that would help a future session (e.g. a new module hook, a seed step, a theme build rule).

If any apply, **propose it to the user** (don't create/update silently):
- English: _"This also changes how X works — want me to update `.ai/skills/<file>.md` to reflect it?"_
- Spanish: _"Esto también cambia cómo funciona X — ¿actualizo `.ai/skills/<file>.md`?"_

If the user confirms, update the existing file (or create a new numbered one starting at `01`). If nothing qualifies, skip silently — do not ask for the sake of asking.

---

## Writing to MEMORY.md

Append a new entry at the bottom of `.ai/session/MEMORY.md`. Never edit or delete existing entries.

```markdown
## Session YYYY-MM-DD HH:MM

**Summary**: One sentence describing the goal of the session.

**Work done**:
- Concrete item 1 (what was changed and why)
- Concrete item 2
- ...

**Files modified**:
- `eap/modules/<module>/...` — what changed
- `eap/themes/eap_theme/...` — what changed

**Pending / Next steps**: Any unfinished work, known limitations, or follow-up tasks.
```

Rules:
- Use the actual current date (`currentDate` from system context if available).
- Be specific — future sessions must be able to resume from this entry alone.
- If this is a continuation of a previous session on the same day, append to that entry rather than creating a duplicate.

---

## Writing to BUGLOG.md

Append a new entry at the bottom of `.ai/session/BUGLOG.md`. Never edit or delete existing entries. Number as the next free integer.

```markdown
## BUG-NNN — Short descriptive title

**Status:** Fixed
**Date:** YYYY-MM-DD
**Session:** See .ai/session/MEMORY.md → Session YYYY-MM-DD

**Symptom:**
What the user reported seeing / what failed.

**Root cause:**
Why it happened. Enough detail to recognise the pattern if it recurs.

**Fix:**
What was changed and why it resolves the symptom.

**Pattern to watch:** (optional) Related places in the codebase that may have the same issue.

**Files:**
- `path/to/file`
```

Rules:
- Reference the MEMORY.md session in every BUGLOG entry.
- The "Pattern to watch" line is optional but valuable — add it when the bug reflects a recurring anti-pattern in the codebase.
- If the bug was pre-existing and discovered during an implementation session, still create a BUGLOG entry.

---

## Language Detection

Detect the user's language from their messages. Use the **same language** for:
- Confirmation questions ("Did it work?")
- Status updates ("I found a related bug...")
- End-of-session prompts ("Shall I log this session?")

Do not switch languages mid-conversation unless the user does first.

---

## End-of-Conversation Prompt

When the conversation is naturally concluding (user says thanks, goodbye, or the task is clearly complete), always prompt:

- English: _"Before we close — shall I write the session summary to MEMORY.md?"_
- Spanish: _"Antes de cerrar — ¿escribo el resumen de la sesión en MEMORY.md?"_
- French: _"Avant de terminer — dois-je écrire le résumé de session dans MEMORY.md ?"_

Only write if the user confirms. If they say "yes" / "sí" / "oui" (or equivalent), write the entry and confirm: _"Logged. See you next session."_
