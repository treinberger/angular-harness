---
name: harness-setup
description: Use when the user asks to set up, install, or integrate the treinberger Angular harness into the current project ("harness-setup", "harness einrichten", "set up the harness", "adopt the harness") - fetches the angular-harness repo and executes its setup runbook against the current working directory
---

# Harness Setup

Install the treinberger Angular harness (https://github.com/treinberger/angular-harness)
into the CURRENT project by executing the harness repo's own runbook.

## Steps

1. **Get the harness repo.** Prefer a local clone if one exists (ask the
   user if unsure). Otherwise clone read-only into a temp directory:

   ```bash
   gh repo clone treinberger/angular-harness /tmp/angular-harness -- --depth=1
   ```

2. **Load the runbook.** Read `CLAUDE.md` in the harness repo and follow
   **Job B — Setup runbook** exactly: preflight (packages published?
   target is Angular?), classify greenfield vs. brownfield, then execute
   the matching doc (`docs/consuming-a-project.md` or
   `docs/adopting-existing-project.md`) against the current working
   directory.

3. **Do not improvise.** If preflight fails (packages not published, target
   not Angular), report why and stop. If the runbook and this skill ever
   disagree, the harness repo's `CLAUDE.md` wins — it is the source of
   truth and may be newer than this installed copy.

4. **Finish with the runbook's verification** (`pnpm lint && pnpm test &&
   pnpm build && pnpm doctor`, plus e2e if scaffolded) and report:
   scaffolded files, migrations run, baseline relaxations with exit
   criteria.
