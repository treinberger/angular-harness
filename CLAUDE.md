# CLAUDE.md — angular-harness

This repo is the central development harness for Angular web projects:
shared tooling packages (`@treinberger/harness-*`), layered guidelines,
an `ng add` onboarding schematic, a `harness doctor` CLI, a shared Renovate
preset, and a reusable CI workflow. Consuming projects hold only thin
configs; the reusable parts live here.

A session in this repo is one of two jobs. Decide which before doing anything:

| The user wants… | Job | Go to |
|---|---|---|
| to build/extend the harness itself | **A — Implement** | [docs/plans/](docs/plans/) |
| the harness installed into some Angular project ("harness-setup", "harness einrichten") | **B — Setup** | Runbook below |

## Job A — Implement the harness

Follow the newest plan in `docs/plans/` task-by-task. Each task is TDD with
complete code, ends with green verification and a commit. Do not reorder
tasks. Version-drift notes in the plan (rule ids, export names) must be
checked against the majors that `pnpm add` actually resolves.

## Job B — Setup runbook (install the harness into a target project)

### Step 0 — Preflight

1. **Are the packages published yet?** Check:

   ```bash
   npm view @treinberger/harness-schematics version --registry=https://npm.pkg.github.com
   ```

   If this fails with 404, the harness is not implemented/released yet.
   Stop and tell the user: Job A must happen first. Do not improvise a
   partial setup by copying files out of this repo — the packages are the
   contract.
2. **Where is the target project?** If the session started in this repo,
   ask the user for the target path (or the repo to clone). Never scaffold
   into `angular-harness` itself.
3. **Is it an Angular project?** Target must contain `angular.json`
   (or be an empty/new project the user wants created via `ng new`).
   Anything else (React, plain TS, backend): stop — this harness is
   Angular-only.

### Step 1 — Classify the target

- **Greenfield**: freshly generated (`ng new`, no meaningful app code yet)
  → follow [docs/consuming-a-project.md](docs/consuming-a-project.md).
- **Brownfield**: existing app code
  → follow [docs/adopting-existing-project.md](docs/adopting-existing-project.md).

(Until Job A is complete these two files may not exist yet; the plan in
`docs/plans/` defines their final content — that means the harness is not
released and Step 0 already told you to stop.)

### Step 2 — Execute

Work inside the target project, not in this repo. Follow the chosen doc
literally. Non-negotiables:

- Selector prefix: reuse the target's existing prefix from `angular.json`;
  only greenfield projects pick a new one (ask the user).
- One commit per logical step (ng add scaffold, each Angular migration,
  ratchet config, baseline doc). Conventional Commits.
- Brownfield: run the official Angular migrations BEFORE reaching for
  `legacyDirs`; the ratchet is for what migrations cannot fix.
- Never weaken the shared presets globally. Relaxations go through the
  documented extension points and into `docs/guidelines/legacy-baseline.md`
  with an exit criterion per row.
- Existing husky hooks / old eslint or prettier configs: port their
  project-specific content into the harness extension points, then remove
  them — hooks must not run twice.

### Step 3 — Verify (definition of done)

In the target project, all green:

```bash
pnpm lint && pnpm test && pnpm build && pnpm doctor
```

Plus e2e if scaffolded (`pnpm e2e`). Report to the user: what was
scaffolded, which migrations ran, every relaxation recorded in the baseline
(if any), and what the ratchet exit criteria are.

## Setup from inside the target project (skill)

To trigger Job B from a session running in the target project (user says
"mach harness-setup"), install the skill from this repo once:

```bash
mkdir -p ~/.claude/skills
cp -r .claude/skills/harness-setup ~/.claude/skills/
```

The skill clones/fetches this repo, loads this runbook, and executes Job B
against the current working directory. See
[.claude/skills/harness-setup/SKILL.md](.claude/skills/harness-setup/SKILL.md).

## Hard rules for this repo

- Language: English for all code, docs, commits.
- pnpm only. TDD. Conventional Commits.
- The six packages stay on one fixed version (changesets `fixed`); doctor
  depends on that.
