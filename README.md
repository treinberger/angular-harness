# angular-harness

State-of-the-art development harness for future Angular web projects: shared tooling presets (TypeScript, ESLint, Prettier, testing), **enforced** architecture boundaries (Sheriff), a layered guidelines system (harness-wide core rules + per-project overrides), an agentic-development layer (CLAUDE.md + verify skill), a `harness doctor` compliance CLI, a shared Renovate preset, git hygiene (lefthook + commitlint), one-command onboarding via `ng add` with an `ng update` migration path, and hardened reusable CI.

**Status: planning.** Nothing is implemented yet.

👉 **Start here:** [docs/plans/2026-07-15-angular-harness-implementation.md](docs/plans/2026-07-15-angular-harness-implementation.md) — the complete, task-by-task implementation plan (12 tasks). Execute it in order; each task ends with a passing verification step and a commit. Task 12 replaces this bootstrap README with the final one.

**For Claude Code sessions:** [CLAUDE.md](CLAUDE.md) routes between the two jobs — implementing the harness itself (Job A, the plan above) and installing it into a target project (Job B, setup runbook). To trigger setup from inside a target project, install the [harness-setup skill](.claude/skills/harness-setup/SKILL.md) once: `cp -r .claude/skills/harness-setup ~/.claude/skills/`.
