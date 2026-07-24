# Angular Project Harness — Implementation Plan (v2, SOTA)

> **For agentic workers:** Implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Each task ends with a passing verification step and a commit. Do not start a task before the previous one is committed. If a `superpowers:subagent-driven-development` or `superpowers:executing-plans` skill is available in your environment, use it; otherwise execute the tasks sequentially in order.

**Goal:** Build a state-of-the-art, reusable development harness for Angular web projects — greenfield **and existing codebases**: shared versioned tooling presets (TypeScript, ESLint, Prettier, testing), **enforced** architecture boundaries (Sheriff), a layered guidelines system with per-project overrides, an agentic-development layer (CLAUDE.md + project verify skill), a `harness doctor` compliance CLI, automated dependency management (shared Renovate preset), git hygiene (lefthook + commitlint), one-command onboarding via `ng add` with an `ng update` migration path, and hardened reusable CI.

**Architecture:** A pnpm monorepo (`angular-harness`) publishing focused npm packages under the `@treinberger` scope to GitHub Packages. Consuming projects run `ng add @treinberger/harness-schematics`, which scaffolds configs, `harness.config.json`, layered guidelines, agent files, hooks, Renovate config, and CI. Guidelines follow a two-layer model: **core** guidelines live here (version-pinned); **project** guidelines live in each consuming repo and win on conflict. Where a guideline can be enforced by a machine, it is: ESLint + Sheriff enforce style and boundaries, `harness doctor` detects config drift, CI blocks on all of it. Prose guidelines are the fallback, not the mechanism. The reusability lives in this repo, not in the consuming project: a project — new or existing — only holds thin config files plus its own overrides. Existing codebases adopt through the same `ng add` plus a documented brownfield path: official Angular migrations first, then a **legacy ratchet** (`legacyDirs` in the ESLint factory, documented tsconfig relaxations) that downgrades modernization rules to warnings for listed legacy directories — the list may only ever shrink.

**Tech Stack:** Angular ≥ 21 (standalone, signals, zoneless), TypeScript strict, Node 22 LTS, pnpm, ESLint 9 flat config (`angular-eslint` + `typescript-eslint`), Sheriff (`@softarc/sheriff-core`) for module boundaries, Prettier 3, Vitest (unit, via `@angular/build:unit-test`), Angular Testing Library, Playwright + `@axe-core/playwright` (e2e + a11y), lefthook + commitlint, Renovate, Angular Schematics, GitHub Actions reusable workflows, Changesets.

## Global Constraints

- Node `>=22.12`, pnpm `>=9`. Use `pnpm` for everything; never `npm install` inside the monorepo.
- All published packages are scoped `@treinberger/*`, published to GitHub Packages (`npm.pkg.github.com`).
- Package names: `@treinberger/harness-tsconfig`, `@treinberger/harness-prettier-config`, `@treinberger/harness-eslint-config`, `@treinberger/harness-testing`, `@treinberger/harness-cli`, `@treinberger/harness-schematics`.
- All code, docs, comments, and commit messages in **English**.
- Commit style: Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`).
- TypeScript `strict: true` everywhere, including the harness's own source.
- ESM-first: config packages ship `.mjs`/JSON; the schematics package ships CJS (required by `@angular-devkit/schematics`).
- Do not pin exact dependency versions in this plan; install with `pnpm add` (latest) and let the lockfile pin. Peer ranges use the major that `pnpm add` resolves at implementation time.
- Every task ends with green verification (`pnpm test` / documented manual check) before committing.
- Angular-facing opinions baked into presets: standalone components only, `ChangeDetectionStrategy.OnPush` enforced, signals-first, native control flow (`@if`/`@for`), zoneless change detection, feature isolation enforced by Sheriff.
- **Enforcement-first principle:** whenever a guideline can be checked by a tool, the tool check is the source of truth; the markdown guideline explains the *why*.

---

## Repository layout (target state)

```
angular-harness/
├── package.json                  # private root, pnpm workspace
├── pnpm-workspace.yaml
├── vitest.config.mts             # runs all package tests
├── renovate.json                 # this repo's own renovate config
├── renovate/
│   └── default.json              # SHARED Renovate preset for consumers
├── .github/workflows/
│   ├── ci.yml                    # CI for this repo itself
│   ├── release.yml               # changesets publish to GitHub Packages
│   └── angular-ci.yml            # REUSABLE workflow consumed by projects
├── packages/
│   ├── tsconfig/                 # @treinberger/harness-tsconfig
│   ├── prettier-config/          # @treinberger/harness-prettier-config
│   ├── eslint-config/            # @treinberger/harness-eslint-config
│   ├── testing/                  # @treinberger/harness-testing
│   ├── cli/                      # @treinberger/harness-cli  (harness doctor)
│   └── schematics/               # @treinberger/harness-schematics
├── guidelines/                   # LAYER 0: core guidelines (versioned here)
│   ├── 00-layering.md
│   ├── 01-architecture.md
│   ├── 02-angular-style.md
│   ├── 03-testing.md
│   ├── 04-git-workflow.md
│   ├── 05-dependencies.md
│   └── 06-agentic-development.md
├── schemas/
│   └── harness.config.schema.json
├── docs/
│   ├── plans/                    # this plan
│   └── consuming-a-project.md    # how a project adopts the harness
└── examples/
    └── demo-app/                 # consuming Angular app, integration proof
```

## What a consuming project ends up with

```
my-app/
├── harness.config.json           # machine-readable harness contract
├── eslint.config.mjs             # extends harness factory, project overrides inline
├── sheriff.config.ts             # module boundary rules (core/shared/features)
├── lefthook.yml                  # pre-commit lint, commit-msg commitlint
├── commitlint.config.mjs
├── renovate.json                 # extends shared preset
├── playwright.config.ts          # harness factory
├── CLAUDE.md                     # agent contract: layered guideline reading order
├── .claude/skills/verify-project/SKILL.md
├── docs/guidelines/              # LAYER 1: project-specific rules (win on conflict)
└── .github/workflows/ci.yml      # calls reusable harness workflow
```

---

### Task 1: Monorepo scaffolding

**Files:**
- Create: `package.json`, `pnpm-workspace.yaml`, `.gitignore`, `.editorconfig`, `.nvmrc`, `vitest.config.mts`, `.github/workflows/ci.yml`

**Interfaces:**
- Produces: pnpm workspace with `packages/*` and `examples/*`; root scripts `pnpm test`, `pnpm lint`; CI that runs them on push/PR.

- [ ] **Step 1: Write root `package.json`**

```json
{
  "name": "angular-harness",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "engines": { "node": ">=22.12", "pnpm": ">=9" },
  "packageManager": "pnpm@9",
  "scripts": {
    "test": "vitest run",
    "lint": "prettier --check ."
  }
}
```

Note: `packageManager` must carry an exact version — after `pnpm install`, replace `pnpm@9` with the exact installed version (`pnpm --version`), e.g. `pnpm@9.15.4`.

- [ ] **Step 2: Write `pnpm-workspace.yaml`**

```yaml
packages:
  - "packages/*"
  - "examples/*"
```

- [ ] **Step 3: Write `.gitignore`**

```
node_modules/
dist/
coverage/
.angular/
playwright-report/
test-results/
*.tsbuildinfo
.DS_Store
```

- [ ] **Step 4: Write `.editorconfig`**

```ini
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
```

- [ ] **Step 5: Write `.nvmrc`**

```
22
```

- [ ] **Step 6: Write root `vitest.config.mts`**

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["packages/*/test/**/*.test.ts", "packages/*/test/**/*.test.mts"],
    watch: false,
    passWithNoTests: true,
  },
});
```

- [ ] **Step 7: Install root dev dependencies**

Run: `pnpm add -D -w vitest prettier`
Expected: lockfile created, no errors.

- [ ] **Step 8: Write `.github/workflows/ci.yml` (CI for the harness repo itself)**

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test
```

(Task 11 extends this job with package builds and the example-app verification.)

- [ ] **Step 9: Verify**

Run: `pnpm lint && pnpm test`
Expected: prettier passes; vitest passes with no test files (allowed only in this task; every later task adds tests).

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "chore: scaffold pnpm monorepo with vitest and CI"
```

---

### Task 2: `@treinberger/harness-tsconfig`

**Files:**
- Create: `packages/tsconfig/package.json`, `packages/tsconfig/base.json`, `packages/tsconfig/README.md`
- Test: `packages/tsconfig/test/base.test.ts`

**Interfaces:**
- Produces: `@treinberger/harness-tsconfig/base.json` — consuming projects set `"extends": "@treinberger/harness-tsconfig/base.json"` in their `tsconfig.json`.

- [ ] **Step 1: Write the failing test**

`packages/tsconfig/test/base.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const base = JSON.parse(readFileSync(join(here, "..", "base.json"), "utf8"));

describe("harness-tsconfig base", () => {
  it("enforces strict mode", () => {
    expect(base.compilerOptions.strict).toBe(true);
  });
  it("enforces strict Angular templates", () => {
    expect(base.angularCompilerOptions.strictTemplates).toBe(true);
  });
  it("enables noUncheckedIndexedAccess", () => {
    expect(base.compilerOptions.noUncheckedIndexedAccess).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test`
Expected: FAIL — `base.json` does not exist (ENOENT).

- [ ] **Step 3: Write `packages/tsconfig/base.json`**

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "target": "ES2022",
    "module": "preserve",
    "moduleResolution": "bundler"
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true,
    "typeCheckHostBindings": true
  }
}
```

- [ ] **Step 4: Write `packages/tsconfig/package.json`**

```json
{
  "name": "@treinberger/harness-tsconfig",
  "version": "0.1.0",
  "description": "Strict TypeScript base config for Angular harness projects",
  "license": "MIT",
  "files": ["base.json"],
  "exports": {
    "./base.json": "./base.json"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/treinberger/angular-harness.git",
    "directory": "packages/tsconfig"
  }
}
```

- [ ] **Step 5: Write `packages/tsconfig/README.md`**

```markdown
# @treinberger/harness-tsconfig

Strict TypeScript base config for Angular harness projects.

## Usage

In your project's `tsconfig.json`:

    {
      "extends": "@treinberger/harness-tsconfig/base.json",
      "compilerOptions": {
        "outDir": "./dist/out-tsc"
      }
    }

Project-specific overrides go into the project's own `tsconfig.json` and win over the base.
```

(Indented code style keeps this plan renderable; in the actual README, normal triple-backtick fences are fine.)

- [ ] **Step 6: Run tests to verify they pass**

Run: `pnpm test`
Expected: 3 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/tsconfig
git commit -m "feat: add @treinberger/harness-tsconfig strict base config"
```

---

### Task 3: `@treinberger/harness-prettier-config`

**Files:**
- Create: `packages/prettier-config/package.json`, `packages/prettier-config/index.json`, `packages/prettier-config/README.md`
- Test: `packages/prettier-config/test/config.test.ts`

**Interfaces:**
- Produces: consuming projects set `"prettier": "@treinberger/harness-prettier-config"` in their `package.json`.

- [ ] **Step 1: Write the failing test**

`packages/prettier-config/test/config.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const config = JSON.parse(readFileSync(join(here, "..", "index.json"), "utf8"));

describe("harness-prettier-config", () => {
  it("uses single quotes and 100 char width", () => {
    expect(config.singleQuote).toBe(true);
    expect(config.printWidth).toBe(100);
  });
  it("parses Angular HTML templates with the angular parser", () => {
    const htmlOverride = config.overrides.find((o: { files: string }) => o.files === "*.html");
    expect(htmlOverride.options.parser).toBe("angular");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test`
Expected: FAIL — ENOENT `index.json`.

- [ ] **Step 3: Write `packages/prettier-config/index.json`**

```json
{
  "printWidth": 100,
  "singleQuote": true,
  "semi": true,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
  "overrides": [
    {
      "files": "*.html",
      "options": { "parser": "angular" }
    }
  ]
}
```

- [ ] **Step 4: Write `packages/prettier-config/package.json`**

```json
{
  "name": "@treinberger/harness-prettier-config",
  "version": "0.1.0",
  "description": "Shared Prettier config for Angular harness projects",
  "license": "MIT",
  "main": "index.json",
  "files": ["index.json"],
  "exports": {
    ".": "./index.json"
  },
  "peerDependencies": {
    "prettier": ">=3"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/treinberger/angular-harness.git",
    "directory": "packages/prettier-config"
  }
}
```

- [ ] **Step 5: Write `packages/prettier-config/README.md`**

```markdown
# @treinberger/harness-prettier-config

## Usage

In your project's `package.json`:

    "prettier": "@treinberger/harness-prettier-config"

To override per project, create `.prettierrc.mjs`:

    import base from '@treinberger/harness-prettier-config' with { type: 'json' };
    export default { ...base, printWidth: 120 };
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `pnpm test`
Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/prettier-config
git commit -m "feat: add @treinberger/harness-prettier-config"
```

---

### Task 4: `@treinberger/harness-eslint-config` (with Sheriff boundaries)

**Files:**
- Create: `packages/eslint-config/package.json`, `packages/eslint-config/index.mjs`, `packages/eslint-config/README.md`
- Test: `packages/eslint-config/test/lint.test.ts`, `packages/eslint-config/test/fixture/bad.component.ts`, `packages/eslint-config/test/fixture/tsconfig.json`

**Interfaces:**
- Produces: `defineHarnessEslintConfig(options: { prefix?: string; boundaries?: boolean; legacyDirs?: string[]; extraTs?: object[]; extraTemplate?: object[] }): FlatConfig[]` — default export is `defineHarnessEslintConfig()` with prefix `app` and `boundaries: false` (the scaffolded consumer config passes `boundaries: true` because the schematic also scaffolds `sheriff.config.ts`). `legacyDirs` is the brownfield ratchet: for the listed directories, modernization rules are downgraded to warnings so existing code lints green while new code stays strict.

- [ ] **Step 1: Install dependencies for the package**

First create a minimal `packages/eslint-config/package.json` so the filter resolves:

```json
{
  "name": "@treinberger/harness-eslint-config",
  "version": "0.1.0",
  "description": "Shared ESLint flat config for Angular harness projects",
  "license": "MIT",
  "type": "module",
  "main": "index.mjs",
  "files": ["index.mjs"],
  "exports": { ".": "./index.mjs" },
  "peerDependencies": {
    "eslint": ">=9",
    "typescript": ">=5.6"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/treinberger/angular-harness.git",
    "directory": "packages/eslint-config"
  }
}
```

Run:
```bash
pnpm add -D --filter @treinberger/harness-eslint-config eslint @eslint/js typescript-eslint angular-eslint eslint-config-prettier @softarc/eslint-plugin-sheriff typescript
```

After `pnpm add` resolves versions, move `@eslint/js`, `typescript-eslint`, `angular-eslint`, `eslint-config-prettier`, and `@softarc/eslint-plugin-sheriff` from `devDependencies` to `dependencies` in this package.json (they are runtime deps of the shared config; `eslint` and `typescript` stay peers).

- [ ] **Step 2: Write the failing test**

`packages/eslint-config/test/fixture/bad.component.ts`:

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'wrongprefix-widget',
  template: '<p>{{ title }}</p>',
})
export class WidgetComponent {
  title = 'hello';
}
```

`packages/eslint-config/test/fixture/tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "preserve",
    "moduleResolution": "bundler",
    "experimentalDecorators": true
  },
  "include": ["*.ts"]
}
```

`packages/eslint-config/test/lint.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { ESLint } from "eslint";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";
import { defineHarnessEslintConfig } from "../index.mjs";

const here = dirname(fileURLToPath(import.meta.url));
const fixtureDir = join(here, "fixture");

async function lintFixture(): Promise<string[]> {
  const eslint = new ESLint({
    cwd: fixtureDir,
    overrideConfigFile: true,
    overrideConfig: defineHarnessEslintConfig({ prefix: "app" }),
  });
  const results = await eslint.lintFiles([join(fixtureDir, "bad.component.ts")]);
  return results.flatMap((r) => r.messages.map((m) => m.ruleId ?? ""));
}

describe("harness-eslint-config", () => {
  it("flags a component selector with the wrong prefix", async () => {
    expect(await lintFixture()).toContain("@angular-eslint/component-selector");
  });

  it("flags a component without OnPush change detection", async () => {
    expect(await lintFixture()).toContain(
      "@angular-eslint/prefer-on-push-component-change-detection",
    );
  });

  it("includes a sheriff config block only when boundaries are enabled", () => {
    const withBoundaries = defineHarnessEslintConfig({ boundaries: true });
    const withoutBoundaries = defineHarnessEslintConfig({ boundaries: false });
    const hasSheriffRule = (configs: object[]) =>
      configs.some((c) =>
        JSON.stringify(Object.keys((c as { rules?: object }).rules ?? {})).includes("sheriff"),
      );
    expect(hasSheriffRule(withBoundaries)).toBe(true);
    expect(hasSheriffRule(withoutBoundaries)).toBe(false);
  });

  it("downgrades modernization rules to warnings inside legacyDirs", () => {
    const configs = defineHarnessEslintConfig({ legacyDirs: ["src/app/legacy"] }) as Array<{
      files?: string[];
      rules?: Record<string, unknown>;
    }>;
    const legacyTs = configs.find((c) =>
      (c.files ?? []).includes("src/app/legacy/**/*.ts"),
    );
    const legacyHtml = configs.find((c) =>
      (c.files ?? []).includes("src/app/legacy/**/*.html"),
    );
    expect(legacyTs?.rules?.["@angular-eslint/prefer-on-push-component-change-detection"]).toBe(
      "warn",
    );
    expect(legacyTs?.rules?.["@angular-eslint/prefer-standalone"]).toBe("warn");
    expect(legacyHtml?.rules?.["@angular-eslint/template/prefer-control-flow"]).toBe("warn");
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm test`
Expected: FAIL — `defineHarnessEslintConfig` is not exported / `index.mjs` missing.

- [ ] **Step 4: Write `packages/eslint-config/index.mjs`**

```js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import angular from 'angular-eslint';
import prettier from 'eslint-config-prettier';
import sheriff from '@softarc/eslint-plugin-sheriff';

/**
 * Build the harness ESLint flat config.
 *
 * @param {object} [options]
 * @param {string} [options.prefix='app'] Angular selector prefix for this project.
 * @param {boolean} [options.boundaries=false] Enable Sheriff module-boundary
 *   enforcement. Requires a `sheriff.config.ts` in the project root
 *   (scaffolded by @treinberger/harness-schematics).
 * @param {string[]} [options.legacyDirs=[]] Brownfield ratchet: directories
 *   (no trailing slash or glob) of pre-harness code. Modernization rules are
 *   downgraded to warnings there. This list may only ever SHRINK — document
 *   it in docs/guidelines/ and remove entries as code is modernized.
 * @param {object[]} [options.extraTs=[]] Additional flat-config objects appended for *.ts files.
 * @param {object[]} [options.extraTemplate=[]] Additional flat-config objects appended for *.html files.
 * @returns {import('typescript-eslint').ConfigArray}
 */
export function defineHarnessEslintConfig({
  prefix = 'app',
  boundaries = false,
  legacyDirs = [],
  extraTs = [],
  extraTemplate = [],
} = {}) {
  return tseslint.config(
    {
      files: ['**/*.ts'],
      extends: [
        eslint.configs.recommended,
        ...tseslint.configs.strictTypeChecked,
        ...tseslint.configs.stylisticTypeChecked,
        ...angular.configs.tsRecommended,
        prettier,
      ],
      processor: angular.processInlineTemplates,
      languageOptions: {
        parserOptions: {
          projectService: true,
        },
      },
      rules: {
        '@angular-eslint/directive-selector': [
          'error',
          { type: 'attribute', prefix, style: 'camelCase' },
        ],
        '@angular-eslint/component-selector': [
          'error',
          { type: 'element', prefix, style: 'kebab-case' },
        ],
        '@angular-eslint/prefer-on-push-component-change-detection': 'error',
        '@angular-eslint/prefer-standalone': 'error',
        '@angular-eslint/no-empty-lifecycle-method': 'error',
        '@typescript-eslint/explicit-member-accessibility': [
          'error',
          { accessibility: 'no-public' },
        ],
        '@typescript-eslint/consistent-type-imports': 'error',
        '@typescript-eslint/no-unused-vars': [
          'error',
          { argsIgnorePattern: '^_', varsIgnorePattern: '^_' },
        ],
      },
    },
    ...(boundaries
      ? [
          {
            files: ['**/*.ts'],
            extends: [sheriff.configs.all],
          },
        ]
      : []),
    ...(legacyDirs.length
      ? [
          {
            files: legacyDirs.map((dir) => `${dir}/**/*.ts`),
            rules: {
              '@angular-eslint/prefer-on-push-component-change-detection': 'warn',
              '@angular-eslint/prefer-standalone': 'warn',
              '@typescript-eslint/explicit-member-accessibility': 'warn',
              '@typescript-eslint/consistent-type-imports': 'warn',
            },
          },
        ]
      : []),
    ...extraTs,
    {
      files: ['**/*.html'],
      extends: [
        ...angular.configs.templateRecommended,
        ...angular.configs.templateAccessibility,
      ],
      rules: {
        '@angular-eslint/template/prefer-control-flow': 'error',
      },
    },
    ...(legacyDirs.length
      ? [
          {
            files: legacyDirs.map((dir) => `${dir}/**/*.html`),
            rules: {
              '@angular-eslint/template/prefer-control-flow': 'warn',
            },
          },
        ]
      : []),
    ...extraTemplate,
  );
}

export default defineHarnessEslintConfig();
```

Version-drift notes (verify against installed majors, adjust test and config together if renamed):
- `@angular-eslint/prefer-on-push-component-change-detection` — rule id.
- `sheriff.configs.all` — the flat-config export of `@softarc/eslint-plugin-sheriff`; check the package README for the exact export name of the flat (ESLint 9) preset and use that.

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm test`
Expected: PASS (3 tests).

- [ ] **Step 6: Write `packages/eslint-config/README.md`**

```markdown
# @treinberger/harness-eslint-config

ESLint 9 flat config for Angular harness projects, including Sheriff
module-boundary enforcement.

## Usage

`eslint.config.mjs` in the consuming project:

    import { defineHarnessEslintConfig } from '@treinberger/harness-eslint-config';

    export default defineHarnessEslintConfig({
      prefix: 'myapp',
      boundaries: true, // requires sheriff.config.ts (scaffolded by ng add)
      // Brownfield only: pre-harness directories where modernization rules
      // are warnings, not errors. Ratchet: this list may only shrink.
      legacyDirs: [],
      extraTs: [
        // project-specific rule overrides go here and win over harness defaults
        { files: ['**/*.ts'], rules: { 'no-console': 'error' } },
      ],
    });

See `docs/adopting-existing-project.md` in the harness repo for the full
brownfield adoption flow.
```

- [ ] **Step 7: Commit**

```bash
git add packages/eslint-config pnpm-lock.yaml
git commit -m "feat: add @treinberger/harness-eslint-config with sheriff boundaries"
```

---

### Task 5: `@treinberger/harness-testing`

**Files:**
- Create: `packages/testing/package.json`, `packages/testing/src/index.ts`, `packages/testing/src/playwright.ts`, `packages/testing/src/playwright-a11y.ts`, `packages/testing/src/testing-library.ts`, `packages/testing/tsconfig.json`, `packages/testing/README.md`
- Test: `packages/testing/test/playwright.test.ts`, `packages/testing/test/exports.test.ts`

**Interfaces:**
- Produces:
  - `defineHarnessPlaywrightConfig(options: { baseURL: string; webServerCommand: string; port: number } & Partial<PlaywrightTestConfig>): PlaywrightTestConfig`
  - `harnessTestProviders(): (Provider | EnvironmentProviders)[]` — zoneless providers for TestBed.
  - `renderWithHarness(component, options?)` — Angular Testing Library `render` with harness providers pre-applied (subpath `./testing-library`).
  - `expectNoA11yViolations(page, options?)` — axe-core scan for Playwright tests (subpath `./playwright-a11y`).

- [ ] **Step 1: Create package.json and install deps**

`packages/testing/package.json`:

```json
{
  "name": "@treinberger/harness-testing",
  "version": "0.1.0",
  "description": "Playwright config factory, axe a11y helper, Testing Library wrapper, zoneless TestBed defaults",
  "license": "MIT",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./playwright": { "types": "./dist/playwright.d.ts", "default": "./dist/playwright.js" },
    "./playwright-a11y": { "types": "./dist/playwright-a11y.d.ts", "default": "./dist/playwright-a11y.js" },
    "./testing-library": { "types": "./dist/testing-library.d.ts", "default": "./dist/testing-library.js" }
  },
  "scripts": {
    "build": "tsc -p tsconfig.json"
  },
  "peerDependencies": {
    "@angular/core": ">=21",
    "@playwright/test": ">=1.45",
    "@axe-core/playwright": ">=4",
    "@testing-library/angular": ">=17"
  },
  "peerDependenciesMeta": {
    "@angular/core": { "optional": true },
    "@playwright/test": { "optional": true },
    "@axe-core/playwright": { "optional": true },
    "@testing-library/angular": { "optional": true }
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/treinberger/angular-harness.git",
    "directory": "packages/testing"
  }
}
```

Run:
```bash
pnpm add -D --filter @treinberger/harness-testing typescript @playwright/test @angular/core @axe-core/playwright @testing-library/angular
```

(If `@testing-library/angular` pulls peer warnings for missing Angular packages, add `@angular/common @angular/compiler @angular/platform-browser` as dev deps of this package too — they are needed only for type-checking.)

- [ ] **Step 2: Write `packages/testing/tsconfig.json`**

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src",
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Write the failing tests**

`packages/testing/test/playwright.test.ts` (note: extensionless `../src/...` imports so vitest resolves the `.ts` sources directly):

```ts
import { describe, expect, it } from "vitest";
import { defineHarnessPlaywrightConfig } from "../src/playwright";

describe("defineHarnessPlaywrightConfig", () => {
  it("wires baseURL and webServer from the required options", () => {
    const config = defineHarnessPlaywrightConfig({
      baseURL: "http://localhost:4200",
      webServerCommand: "pnpm start",
      port: 4200,
    });
    expect(config.use?.baseURL).toBe("http://localhost:4200");
    expect(config.webServer).toMatchObject({ command: "pnpm start", port: 4200 });
  });

  it("lets project overrides win", () => {
    const config = defineHarnessPlaywrightConfig({
      baseURL: "http://localhost:4200",
      webServerCommand: "pnpm start",
      port: 4200,
      retries: 5,
    });
    expect(config.retries).toBe(5);
  });

  it("is CI-aware by default", () => {
    const config = defineHarnessPlaywrightConfig({
      baseURL: "http://localhost:4200",
      webServerCommand: "pnpm start",
      port: 4200,
    });
    expect(config.forbidOnly).toBe(!!process.env["CI"]);
  });
});
```

`packages/testing/test/exports.test.ts`:

```ts
import { describe, expect, it } from "vitest";

describe("harness-testing exports", () => {
  it("exposes renderWithHarness", async () => {
    const mod = await import("../src/testing-library");
    expect(typeof mod.renderWithHarness).toBe("function");
  });

  it("exposes expectNoA11yViolations", async () => {
    const mod = await import("../src/playwright-a11y");
    expect(typeof mod.expectNoA11yViolations).toBe("function");
  });

  it("exposes zoneless test providers", async () => {
    const mod = await import("../src/index");
    expect(mod.harnessTestProviders().length).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `pnpm test`
Expected: FAIL — cannot resolve `../src/playwright` etc.

- [ ] **Step 5: Write `packages/testing/src/playwright.ts`**

```ts
import type { PlaywrightTestConfig } from "@playwright/test";
import { devices } from "@playwright/test";

export interface HarnessPlaywrightOptions extends Partial<PlaywrightTestConfig> {
  baseURL: string;
  webServerCommand: string;
  port: number;
}

export function defineHarnessPlaywrightConfig(
  options: HarnessPlaywrightOptions,
): PlaywrightTestConfig {
  const { baseURL, webServerCommand, port, ...overrides } = options;
  const isCI = !!process.env["CI"];
  return {
    testDir: "./e2e",
    fullyParallel: true,
    forbidOnly: isCI,
    retries: isCI ? 2 : 0,
    reporter: isCI ? [["github"], ["html", { open: "never" }]] : [["list"]],
    use: {
      baseURL,
      trace: "on-first-retry",
      screenshot: "only-on-failure",
    },
    projects: [{ name: "chromium", use: { ...devices["Desktop Chrome"] } }],
    webServer: {
      command: webServerCommand,
      port,
      reuseExistingServer: !isCI,
      timeout: 120_000,
    },
    ...overrides,
  };
}
```

- [ ] **Step 6: Write `packages/testing/src/playwright-a11y.ts`**

```ts
import type { Page } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

export interface A11yOptions {
  /** axe rule ids to skip, with a documented reason at the call site. */
  disableRules?: string[];
}

/** Run an axe-core scan on the current page; throw with a readable summary on violations. */
export async function expectNoA11yViolations(
  page: Page,
  options: A11yOptions = {},
): Promise<void> {
  let builder = new AxeBuilder({ page });
  if (options.disableRules?.length) {
    builder = builder.disableRules(options.disableRules);
  }
  const results = await builder.analyze();
  if (results.violations.length > 0) {
    const summary = results.violations
      .map((v) => `${v.id} [${v.impact ?? "n/a"}]: ${v.help} (${v.nodes.length} nodes)`)
      .join("\n");
    throw new Error(`Accessibility violations:\n${summary}`);
  }
}
```

- [ ] **Step 7: Write `packages/testing/src/testing-library.ts`**

```ts
import type { Type } from "@angular/core";
import {
  render,
  type RenderComponentOptions,
  type RenderResult,
} from "@testing-library/angular";
import { harnessTestProviders } from "./index.js";

/** Angular Testing Library render with harness defaults (zoneless) pre-applied. */
export async function renderWithHarness<T>(
  component: Type<T>,
  options: RenderComponentOptions<T> = {},
): Promise<RenderResult<T>> {
  return render(component, {
    ...options,
    providers: [...harnessTestProviders(), ...(options.providers ?? [])],
  });
}
```

- [ ] **Step 8: Write `packages/testing/src/index.ts`**

```ts
import type { EnvironmentProviders, Provider } from "@angular/core";
import { provideZonelessChangeDetection } from "@angular/core";

export { defineHarnessPlaywrightConfig } from "./playwright.js";
export type { HarnessPlaywrightOptions } from "./playwright.js";

/**
 * Default providers for TestBed in harness projects: zoneless change detection.
 * Usage: TestBed.configureTestingModule({ providers: [...harnessTestProviders()] })
 */
export function harnessTestProviders(): (Provider | EnvironmentProviders)[] {
  return [provideZonelessChangeDetection()];
}
```

If the installed `@angular/core` exports the zoneless provider under a different name (it was `provideExperimentalZonelessChangeDetection` before stabilization), use the stable name the installed major exposes and note it in the README.

- [ ] **Step 9: Run tests, then build**

Run: `pnpm test && pnpm --filter @treinberger/harness-testing build`
Expected: all tests PASS; `dist/` contains `index.*`, `playwright.*`, `playwright-a11y.*`, `testing-library.*`.

- [ ] **Step 10: Write `packages/testing/README.md`**

```markdown
# @treinberger/harness-testing

## Playwright

`playwright.config.ts`:

    import { defineHarnessPlaywrightConfig } from '@treinberger/harness-testing/playwright';

    export default defineHarnessPlaywrightConfig({
      baseURL: 'http://localhost:4200',
      webServerCommand: 'pnpm start',
      port: 4200,
    });

## Accessibility in e2e

    import { expectNoA11yViolations } from '@treinberger/harness-testing/playwright-a11y';

    test('home page is accessible', async ({ page }) => {
      await page.goto('/');
      await expectNoA11yViolations(page);
    });

## Unit tests (Vitest via @angular/build:unit-test)

    import { renderWithHarness } from '@treinberger/harness-testing/testing-library';

    const { getByRole } = await renderWithHarness(MyComponent, {
      inputs: { title: 'hello' },
    });

Or plain TestBed:

    import { harnessTestProviders } from '@treinberger/harness-testing';

    TestBed.configureTestingModule({ providers: [...harnessTestProviders()] });
```

- [ ] **Step 11: Commit**

```bash
git add packages/testing pnpm-lock.yaml
git commit -m "feat: add @treinberger/harness-testing with playwright, a11y, and testing-library helpers"
```

---

### Task 6: Core guidelines (Layer 0) and config schema

**Files:**
- Create: `guidelines/00-layering.md`, `guidelines/01-architecture.md`, `guidelines/02-angular-style.md`, `guidelines/03-testing.md`, `guidelines/04-git-workflow.md`, `guidelines/05-dependencies.md`, `guidelines/06-agentic-development.md`, `schemas/harness.config.schema.json`

**Interfaces:**
- Produces: the layering contract every consuming project follows, and the JSON schema that `harness.config.json` validates against. The schematic (Task 8) and `harness doctor` (Task 7) consume the schema's shape.

- [ ] **Step 1: Write `guidelines/00-layering.md`**

```markdown
# Guideline Layering Model

The harness uses two layers of guidelines. Both humans and coding agents MUST
apply them in this order:

| Layer | Location | Owner | Wins on conflict |
|-------|----------|-------|------------------|
| 0 — Core | `angular-harness` repo, `guidelines/` (version-pinned via `harnessVersion` in `harness.config.json`) | harness maintainer | no |
| 1 — Project | consuming repo, `docs/guidelines/` | project team | **yes** |

## Rules

1. Core guidelines apply to every harness project by default.
2. A project may override any core guideline by stating the override
   explicitly in a file under `docs/guidelines/`, including a one-line
   rationale ("**Overrides core:** <rule> — because <reason>").
3. Silent divergence is a defect: if project practice contradicts core and
   no override is documented, fix the practice or document the override.
4. Enforcement-first: where a rule is enforced by tooling (ESLint, Sheriff,
   `harness doctor`, CI), the tool is the source of truth. Overriding an
   enforced rule means overriding it at the documented extension point
   (`extraTs`, `sheriff.config.ts` depRules, prettier spread, tsconfig
   overrides) — never by disabling the shared config wholesale.
5. Coding agents (Claude/CLAUDE.md): read core guidelines first, then all
   files in `docs/guidelines/`; on conflict, the project file wins.
```

- [ ] **Step 2: Write `guidelines/01-architecture.md`**

```markdown
# Architecture

- Standalone components only. No NgModules in new code.
- Feature-first structure: `src/app/features/<feature>/` holds components,
  services, and routes of one feature. Shared, feature-agnostic code lives in
  `src/app/shared/` (presentational) and `src/app/core/` (singletons:
  interceptors, auth, config).
- **Boundaries are enforced by Sheriff** (`sheriff.config.ts`):
  features must not import from other features; `shared` imports nothing
  app-internal; `core` may use `shared`. Changing the dependency rules is a
  project-level architecture decision — document it in `docs/guidelines/`.
- Routing: lazy-load every feature via `loadChildren`/`loadComponent`.
- State: component-local state with signals; cross-component state in
  injectable signal stores (plain services exposing `signal`/`computed`).
  Introduce a state library only via a documented project override.
- HTTP: all API access behind typed service classes in the owning feature
  (or `core/api/` when shared). Components never call `HttpClient` directly.
```

- [ ] **Step 3: Write `guidelines/02-angular-style.md`**

```markdown
# Angular Style

- `ChangeDetectionStrategy.OnPush` on every component (enforced by ESLint).
- Signals-first: `signal`, `computed`, `input()`, `output()`, `model()`.
  Do not add new `@Input()`/`@Output()` decorators or RxJS-based component
  state where a signal suffices. RxJS remains appropriate for event streams
  and HTTP composition; convert at the edge with `toSignal`/`toObservable`.
- Native control flow (`@if`, `@for` with `track`, `@switch`) — no
  `*ngIf`/`*ngFor` in new templates (enforced by ESLint).
- Zoneless change detection (`provideZonelessChangeDetection()`) for new
  projects.
- `inject()` over constructor injection in new code.
- Selector prefix comes from `harness.config.json` → `prefix`.
- File naming: current Angular style-guide naming as generated by the CLI.
  One top-level declarable per file.
- Templates > ~15 lines go into separate `.html` files.
```

- [ ] **Step 4: Write `guidelines/03-testing.md`**

```markdown
# Testing

- Unit tests: Vitest via `@angular/build:unit-test`. Co-located
  `*.spec.ts` next to the code under test.
- Component tests use Angular Testing Library via `renderWithHarness()`
  from `@treinberger/harness-testing/testing-library` — query by role/label,
  interact through the DOM, assert observable behavior. Plain TestBed with
  `harnessTestProviders()` for services and non-DOM logic.
- E2E: Playwright with `defineHarnessPlaywrightConfig`. Smoke-test every
  route; deeper flows for business-critical paths. Every smoke test calls
  `expectNoA11yViolations(page)` — accessibility regressions fail CI.
- TDD is the default working mode: red → green → refactor. A bugfix starts
  with a failing test reproducing the bug.
- Coverage is a signal, not a gate; do not chase numbers with trivial tests.
```

- [ ] **Step 5: Write `guidelines/04-git-workflow.md`**

```markdown
# Git Workflow

- Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`,
  `refactor:`) — enforced locally by commitlint via lefthook.
- Pre-commit hook runs prettier + eslint on staged files (lefthook).
  Bypass (`LEFTHOOK=0`) only for WIP commits on private branches.
- Small, frequent commits; each commit leaves the build green.
- Branch names: `feat/<slug>`, `fix/<slug>`, `chore/<slug>`.
- `main` is protected; changes land via PR with green CI.
- CI must run the reusable workflow
  `treinberger/angular-harness/.github/workflows/angular-ci.yml`.
```

- [ ] **Step 6: Write `guidelines/05-dependencies.md`**

```markdown
# Dependencies

- Renovate manages updates. Every project's `renovate.json` extends the
  shared preset `github>treinberger/angular-harness//renovate/default.json`.
- Non-major updates arrive grouped weekly; majors get individual PRs and are
  reviewed by a human. Angular majors are upgraded deliberately (with
  `ng update`), never auto-merged.
- Harness packages (`@treinberger/harness-*`) are grouped into one PR;
  after merging, update `harnessVersion` in `harness.config.json` and read
  the harness changelog for guideline changes.
- The lockfile is committed. CI installs with `--frozen-lockfile`.
- `pnpm audit --prod --audit-level=high` runs in CI and blocks on findings;
  temporary exceptions require a documented override in `docs/guidelines/`.
- Adding a runtime dependency is an architecture decision: prefer platform
  and Angular built-ins; justify new dependencies in the PR description.
```

- [ ] **Step 7: Write `guidelines/06-agentic-development.md`**

```markdown
# Agentic Development

Rules for coding agents (Claude Code and similar) working in harness projects.

- Entry point is the project's `CLAUDE.md`. It pins the guideline reading
  order: core guidelines (Layer 0) first, then `docs/guidelines/` (Layer 1);
  project wins on conflict.
- Before claiming any task complete, run the project verify skill
  (`.claude/skills/verify-project/`): lint, unit tests, build, doctor —
  and e2e when the change touches routing, templates, or user flows.
- Agents follow TDD like humans: failing test first, then implementation.
- Agents never weaken enforcement to get green: no disabling ESLint rules
  inline without a comment explaining why, no editing `sheriff.config.ts`
  dep rules, no loosening tsconfig — such changes require an explicit
  human decision recorded in `docs/guidelines/`.
- Agents keep commits conventional and small; hooks (lefthook) stay enabled.
- When an agent detects a conflict between layers or between a guideline
  and tooling, it surfaces the conflict instead of silently picking a side.
```

- [ ] **Step 8: Write `schemas/harness.config.schema.json`**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://raw.githubusercontent.com/treinberger/angular-harness/main/schemas/harness.config.schema.json",
  "title": "Harness project configuration",
  "type": "object",
  "additionalProperties": false,
  "required": ["harnessVersion", "prefix"],
  "properties": {
    "$schema": { "type": "string" },
    "harnessVersion": {
      "type": "string",
      "description": "Version of the harness core guidelines this project follows (git tag of angular-harness, e.g. 'v0.1.0')."
    },
    "prefix": {
      "type": "string",
      "pattern": "^[a-z][a-z0-9]*$",
      "description": "Angular selector prefix for this project."
    },
    "guidelinesDir": {
      "type": "string",
      "default": "docs/guidelines",
      "description": "Directory holding Layer-1 project guidelines."
    },
    "boundaries": {
      "type": "boolean",
      "default": true,
      "description": "Whether Sheriff module-boundary enforcement is enabled."
    },
    "ci": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "nodeVersion": { "type": "string", "default": "22" },
        "e2e": { "type": "boolean", "default": true },
        "audit": { "type": "boolean", "default": true }
      }
    }
  }
}
```

- [ ] **Step 9: Validate the schema**

Run:
```bash
echo '{"harnessVersion":"v0.1.0","prefix":"demo","boundaries":true}' > /tmp/hc.json
npx --yes ajv-cli validate -s schemas/harness.config.schema.json -d /tmp/hc.json
```
Expected: `/tmp/hc.json valid`.

- [ ] **Step 10: Commit**

```bash
git add guidelines schemas
git commit -m "docs: add layered core guidelines and harness.config schema"
```

---

### Task 7: `@treinberger/harness-cli` (`harness doctor`)

**Files:**
- Create: `packages/cli/package.json`, `packages/cli/tsconfig.json`, `packages/cli/src/doctor.ts`, `packages/cli/bin/harness.mjs`, `packages/cli/README.md`
- Test: `packages/cli/test/doctor.test.ts`

**Interfaces:**
- Consumes: the file layout scaffolded by Task 8 and the `harness.config.json` shape from Task 6.
- Produces: `runDoctor(root: string): DoctorFinding[]` and a `harness` bin with a `doctor` command — exit 0 when compliant, exit 1 with a finding list when not. Consuming projects get a `doctor` npm script; CI (Task 10) runs it.

- [ ] **Step 1: Create package.json and tsconfig**

`packages/cli/package.json`:

```json
{
  "name": "@treinberger/harness-cli",
  "version": "0.1.0",
  "description": "Compliance doctor for harness projects: detects config drift",
  "license": "MIT",
  "type": "module",
  "main": "dist/doctor.js",
  "types": "dist/doctor.d.ts",
  "bin": { "harness": "./bin/harness.mjs" },
  "files": ["dist", "bin"],
  "scripts": {
    "build": "tsc -p tsconfig.json"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/treinberger/angular-harness.git",
    "directory": "packages/cli"
  }
}
```

`packages/cli/tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src",
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

Run: `pnpm add -D --filter @treinberger/harness-cli typescript`

- [ ] **Step 2: Write the failing test**

`packages/cli/test/doctor.test.ts`:

```ts
import { describe, expect, it, beforeEach, afterEach } from "vitest";
import { mkdtempSync, mkdirSync, readFileSync, rmSync, writeFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { runDoctor } from "../src/doctor";

let root: string;

function write(path: string, content: string): void {
  const full = join(root, path);
  mkdirSync(join(full, ".."), { recursive: true });
  writeFileSync(full, content);
}

/** Lay down a fully compliant project skeleton. */
function compliantProject(): void {
  write(
    "harness.config.json",
    JSON.stringify({ harnessVersion: "v0.1.0", prefix: "demo", boundaries: true }),
  );
  write("eslint.config.mjs", "export default [];");
  write("sheriff.config.ts", "export const config = {};");
  write("CLAUDE.md", "# CLAUDE.md");
  write("lefthook.yml", "pre-commit:");
  write("commitlint.config.mjs", "export default {};");
  write("renovate.json", "{}");
  write("playwright.config.ts", "export default {};");
  write("docs/guidelines/README.md", "# Project Guidelines");
  write(".github/workflows/ci.yml", "name: CI");
  write("tsconfig.json", JSON.stringify({ extends: "@treinberger/harness-tsconfig/base.json" }));
  write(
    "package.json",
    JSON.stringify({
      name: "demo",
      prettier: "@treinberger/harness-prettier-config",
      scripts: { doctor: "harness doctor" },
      devDependencies: {
        "@treinberger/harness-tsconfig": "^0.1.0",
        "@treinberger/harness-prettier-config": "^0.1.0",
        "@treinberger/harness-eslint-config": "^0.1.0",
        "@treinberger/harness-testing": "^0.1.0",
        "@treinberger/harness-cli": "^0.1.0",
      },
    }),
  );
}

describe("runDoctor", () => {
  beforeEach(() => {
    root = mkdtempSync(join(tmpdir(), "doctor-"));
  });
  afterEach(() => {
    rmSync(root, { recursive: true, force: true });
  });

  it("passes on a compliant project", () => {
    compliantProject();
    const findings = runDoctor(root);
    expect(findings.every((f) => f.ok)).toBe(true);
  });

  it("fails when harness.config.json is missing", () => {
    compliantProject();
    rmSync(join(root, "harness.config.json"));
    const failed = runDoctor(root).filter((f) => !f.ok);
    expect(failed.map((f) => f.check)).toContain("harness.config.json exists and is valid");
  });

  it("fails when tsconfig does not extend the harness base", () => {
    compliantProject();
    write("tsconfig.json", JSON.stringify({ compilerOptions: {} }));
    const failed = runDoctor(root).filter((f) => !f.ok);
    expect(failed.map((f) => f.check)).toContain("tsconfig extends harness base");
  });

  it("fails on version drift between harness packages", () => {
    compliantProject();
    const pkgPath = join(root, "package.json");
    const pkg = JSON.parse(readFileSync(pkgPath, "utf8"));
    pkg.devDependencies["@treinberger/harness-testing"] = "^0.2.0";
    writeFileSync(pkgPath, JSON.stringify(pkg));
    const failed = runDoctor(root).filter((f) => !f.ok);
    expect(failed.map((f) => f.check)).toContain("harness package versions aligned");
  });

  it("does not require playwright config when e2e is disabled", () => {
    compliantProject();
    write(
      "harness.config.json",
      JSON.stringify({
        harnessVersion: "v0.1.0",
        prefix: "demo",
        boundaries: true,
        ci: { e2e: false },
      }),
    );
    rmSync(join(root, "playwright.config.ts"));
    expect(runDoctor(root).every((f) => f.ok)).toBe(true);
  });

  it("does not require sheriff config when boundaries are disabled", () => {
    compliantProject();
    write(
      "harness.config.json",
      JSON.stringify({ harnessVersion: "v0.1.0", prefix: "demo", boundaries: false }),
    );
    rmSync(join(root, "sheriff.config.ts"));
    expect(runDoctor(root).every((f) => f.ok)).toBe(true);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm test`
Expected: FAIL — cannot resolve `../src/doctor`.

- [ ] **Step 4: Write `packages/cli/src/doctor.ts`**

```ts
import { existsSync, readFileSync } from "node:fs";
import { join } from "node:path";

export interface DoctorFinding {
  check: string;
  ok: boolean;
  detail: string;
}

interface HarnessConfig {
  harnessVersion?: string;
  prefix?: string;
  boundaries?: boolean;
  ci?: { e2e?: boolean; audit?: boolean; nodeVersion?: string };
}

const ALWAYS_REQUIRED_FILES = [
  "eslint.config.mjs",
  "CLAUDE.md",
  "lefthook.yml",
  "commitlint.config.mjs",
  "renovate.json",
  "docs/guidelines/README.md",
  ".github/workflows/ci.yml",
];

const HARNESS_PACKAGES = [
  "@treinberger/harness-tsconfig",
  "@treinberger/harness-prettier-config",
  "@treinberger/harness-eslint-config",
  "@treinberger/harness-testing",
  "@treinberger/harness-cli",
];

function readJson(root: string, file: string): unknown | undefined {
  const path = join(root, file);
  if (!existsSync(path)) return undefined;
  try {
    return JSON.parse(readFileSync(path, "utf8"));
  } catch {
    return undefined;
  }
}

export function runDoctor(root: string): DoctorFinding[] {
  const findings: DoctorFinding[] = [];
  const add = (check: string, ok: boolean, detail = ""): void => {
    findings.push({ check, ok, detail });
  };

  // 1. harness.config.json exists, parses, has required fields
  const config = readJson(root, "harness.config.json") as HarnessConfig | undefined;
  const configOk =
    !!config &&
    typeof config.harnessVersion === "string" &&
    typeof config.prefix === "string" &&
    /^[a-z][a-z0-9]*$/.test(config.prefix);
  add(
    "harness.config.json exists and is valid",
    configOk,
    configOk ? "" : "missing, unparseable, or missing harnessVersion/prefix",
  );

  // 2. required files
  const required = [...ALWAYS_REQUIRED_FILES];
  if (config?.boundaries !== false) required.push("sheriff.config.ts");
  if (config?.ci?.e2e !== false) required.push("playwright.config.ts");
  for (const file of required) {
    add(`required file: ${file}`, existsSync(join(root, file)), "missing");
  }

  // 3. tsconfig extends harness base
  const tsconfig = readJson(root, "tsconfig.json") as { extends?: string } | undefined;
  add(
    "tsconfig extends harness base",
    tsconfig?.extends === "@treinberger/harness-tsconfig/base.json",
    `found extends: ${String(tsconfig?.extends)}`,
  );

  // 4. package.json wiring
  const pkg = readJson(root, "package.json") as
    | { prettier?: string; devDependencies?: Record<string, string>; scripts?: Record<string, string> }
    | undefined;
  add(
    "prettier config registered",
    pkg?.prettier === "@treinberger/harness-prettier-config" ||
      existsSync(join(root, ".prettierrc.mjs")),
    "package.json prettier key not set and no .prettierrc.mjs override",
  );
  add(
    "doctor script present",
    pkg?.scripts?.["doctor"] === "harness doctor",
    "add \"doctor\": \"harness doctor\" to scripts",
  );

  // 5. harness package versions aligned
  const deps = pkg?.devDependencies ?? {};
  const versions = new Set(
    HARNESS_PACKAGES.filter((name) => deps[name] !== undefined).map((name) => deps[name]),
  );
  add(
    "harness package versions aligned",
    versions.size <= 1,
    `found versions: ${[...versions].join(", ")}`,
  );

  return findings;
}
```

- [ ] **Step 5: Write `packages/cli/bin/harness.mjs`**

```js
#!/usr/bin/env node
import { runDoctor } from "../dist/doctor.js";

const [, , command] = process.argv;

if (command !== "doctor") {
  console.error("Usage: harness doctor");
  process.exit(2);
}

const findings = runDoctor(process.cwd());
for (const f of findings) {
  console.log(`${f.ok ? "✔" : "✘"} ${f.check}${f.ok || !f.detail ? "" : ` — ${f.detail}`}`);
}
const failed = findings.filter((f) => !f.ok).length;
if (failed > 0) {
  console.error(`\nharness doctor: ${failed} check(s) failed.`);
  process.exit(1);
}
console.log("\nharness doctor: all checks passed.");
```

- [ ] **Step 6: Run tests, then build**

Run: `pnpm test && pnpm --filter @treinberger/harness-cli build`
Expected: all doctor tests PASS; `dist/doctor.js` exists.

- [ ] **Step 7: Write `packages/cli/README.md`**

```markdown
# @treinberger/harness-cli

    pnpm exec harness doctor

Checks a harness project for config drift: required files, tsconfig/prettier
wiring, aligned harness package versions, conditional checks driven by
`harness.config.json` (`boundaries`, `ci.e2e`). Exit 1 on any failed check —
CI runs this on every push.
```

- [ ] **Step 8: Commit**

```bash
git add packages/cli pnpm-lock.yaml
git commit -m "feat: add harness doctor CLI detecting config drift"
```

---

### Task 8: `@treinberger/harness-schematics` (`ng add` + migration path)

**Files:**
- Create: `packages/schematics/package.json`, `packages/schematics/tsconfig.json`, `packages/schematics/collection.json`, `packages/schematics/migrations.json`, `packages/schematics/copy-assets.mjs`, `packages/schematics/src/ng-add/index.ts`, `packages/schematics/src/ng-add/schema.json`, `packages/schematics/src/ng-add/files/…` (template tree, Step 4), `packages/schematics/README.md`
- Test: `packages/schematics/test/ng-add.test.ts`

**Interfaces:**
- Consumes: package names from Tasks 2–7; schema from Task 6; reusable workflow name from Task 10 (`angular-ci.yml` — fixed here, implemented there).
- Produces: `ng add @treinberger/harness-schematics --prefix=<prefix>` scaffolding the full consumer layout, plus an (initially empty) `ng update` migrations collection.

- [ ] **Step 1: Create package.json, tsconfig, copy script; install deps**

`packages/schematics/package.json`:

```json
{
  "name": "@treinberger/harness-schematics",
  "version": "0.1.0",
  "description": "ng add schematic that wires an Angular project into the harness",
  "license": "MIT",
  "schematics": "./collection.json",
  "ng-update": {
    "migrations": "./migrations.json"
  },
  "files": ["collection.json", "migrations.json", "dist"],
  "scripts": {
    "build": "tsc -p tsconfig.json && node copy-assets.mjs"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/treinberger/angular-harness.git",
    "directory": "packages/schematics"
  }
}
```

`packages/schematics/tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "commonjs",
    "moduleResolution": "node",
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/ng-add/files/**"]
}
```

`packages/schematics/copy-assets.mjs`:

```js
import { cpSync } from "node:fs";
cpSync("src/ng-add/files", "dist/ng-add/files", { recursive: true });
cpSync("src/ng-add/schema.json", "dist/ng-add/schema.json");
```

Run:
```bash
pnpm add --filter @treinberger/harness-schematics @angular-devkit/schematics @angular-devkit/core @schematics/angular
pnpm add -D --filter @treinberger/harness-schematics typescript
```

- [ ] **Step 2: Write `collection.json` and `migrations.json`**

`packages/schematics/collection.json`:

```json
{
  "$schema": "../../node_modules/@angular-devkit/schematics/collection-schema.json",
  "schematics": {
    "ng-add": {
      "description": "Wire this Angular project into the treinberger harness",
      "factory": "./dist/ng-add/index#ngAdd",
      "schema": "./dist/ng-add/schema.json"
    }
  }
}
```

`packages/schematics/migrations.json` (empty now; future harness majors add `ng update` migrations here, e.g. rewriting configs when an extension point changes):

```json
{
  "$schema": "../../node_modules/@angular-devkit/schematics/collection-schema.json",
  "schematics": {}
}
```

- [ ] **Step 3: Write `src/ng-add/schema.json`**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "HarnessNgAdd",
  "title": "Harness ng-add options",
  "type": "object",
  "properties": {
    "prefix": {
      "type": "string",
      "description": "Angular selector prefix for this project.",
      "default": "app"
    },
    "e2e": {
      "type": "boolean",
      "description": "Scaffold Playwright e2e setup.",
      "default": true
    },
    "boundaries": {
      "type": "boolean",
      "description": "Scaffold Sheriff module-boundary enforcement.",
      "default": true
    }
  },
  "required": []
}
```

- [ ] **Step 4: Write the template file tree `src/ng-add/files/`**

Files use schematics template syntax (`<%= prefix %>`). Every file carries a `.template` suffix in the source tree, stripped during rendering (Step 6), so the monorepo's own tooling never lints placeholders. Create:

`src/ng-add/files/harness.config.json.template`:

```json
{
  "$schema": "https://raw.githubusercontent.com/treinberger/angular-harness/main/schemas/harness.config.schema.json",
  "harnessVersion": "<%= harnessVersion %>",
  "prefix": "<%= prefix %>",
  "boundaries": <%= boundaries %>
}
```

`src/ng-add/files/eslint.config.mjs.template`:

```js
import { defineHarnessEslintConfig } from '@treinberger/harness-eslint-config';

export default defineHarnessEslintConfig({
  prefix: '<%= prefix %>',
  boundaries: <%= boundaries %>,
  // Brownfield ratchet (see docs/adopting-existing-project.md in the harness
  // repo): pre-harness directories where modernization rules are warnings.
  // This list may only ever shrink.
  legacyDirs: [],
  // Project-specific rule overrides win over harness defaults:
  extraTs: [],
  extraTemplate: [],
});
```

`src/ng-add/files/sheriff.config.ts.template` (only when `boundaries`):

```ts
import { SheriffConfig } from '@softarc/sheriff-core';

export const config: SheriffConfig = {
  enableBarrelLess: true,
  modules: {
    'src/app/core': ['core'],
    'src/app/shared': ['shared'],
    'src/app/features/<feature>': ['feature:<feature>'],
  },
  depRules: {
    root: ['core', 'shared', 'feature:*'],
    'feature:*': ['shared', 'core'],
    core: ['shared'],
    shared: [],
  },
};
```

(The `<feature>` placeholders here are Sheriff's own syntax, not schematic template variables — no `<%= %>`.)

`src/ng-add/files/lefthook.yml.template`:

```yaml
pre-commit:
  commands:
    format:
      glob: "*.{ts,html,css,scss,json,md,yml}"
      run: pnpm exec prettier --check {staged_files}
    lint:
      glob: "*.{ts,html}"
      run: pnpm exec eslint --no-warn-ignored {staged_files}
commit-msg:
  commands:
    commitlint:
      run: pnpm exec commitlint --edit {1}
```

`src/ng-add/files/commitlint.config.mjs.template`:

```js
export default { extends: ['@commitlint/config-conventional'] };
```

`src/ng-add/files/renovate.json.template`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>treinberger/angular-harness//renovate/default.json"]
}
```

`src/ng-add/files/docs/guidelines/README.md.template`:

```markdown
# Project Guidelines (Layer 1)

Project-specific rules for this repository. They EXTEND and, where explicitly
stated, OVERRIDE the harness core guidelines
(https://github.com/treinberger/angular-harness/tree/<%= harnessVersion %>/guidelines).

Override format — always include the rationale:

> **Overrides core:** <core rule> — because <reason>.

Add one markdown file per topic (e.g. `api-conventions.md`, `ui-patterns.md`).
```

`src/ng-add/files/CLAUDE.md.template`:

```markdown
# CLAUDE.md

This project uses the treinberger Angular harness
(https://github.com/treinberger/angular-harness, version <%= harnessVersion %>).

## Guidelines — read in this order

1. Core guidelines (Layer 0):
   https://github.com/treinberger/angular-harness/tree/<%= harnessVersion %>/guidelines
   Summary: standalone + signals + OnPush + zoneless, native control flow,
   feature-first structure with Sheriff-enforced boundaries, TDD with Vitest
   + Testing Library, Playwright e2e with axe a11y checks,
   Conventional Commits.
2. Project guidelines (Layer 1): `docs/guidelines/` in this repo.
   **On conflict, project guidelines win.**

## Commands

- Dev server: `pnpm start`
- Unit tests: `pnpm test`
- Lint: `pnpm lint`
- E2E: `pnpm e2e`
- Compliance: `pnpm doctor`

## Definition of done

Run the verify skill (`.claude/skills/verify-project/`) before claiming any
task complete: lint + test + build + doctor, plus e2e for user-facing changes.

## Hard rules

- Selector prefix: `<%= prefix %>` (see `harness.config.json`).
- Never disable the shared ESLint/Prettier/tsconfig presets wholesale, never
  edit `sheriff.config.ts` dep rules to silence a boundary error — extend via
  the documented extension points; escalate real conflicts to a human.
- If `docs/guidelines/legacy-baseline.md` exists: never ADD to the legacy
  baseline or to `legacyDirs`; when touching a legacy file, upgrade it to
  the full standard (boy-scout rule).
- Every bugfix starts with a failing test.
```

`src/ng-add/files/.claude/skills/verify-project/SKILL.md.template`:

```markdown
---
name: verify-project
description: Verify this project is green before claiming work complete - lint, unit tests, build, harness doctor, and e2e for user-facing changes
---

# Verify Project

Run in this order; stop and fix on the first failure:

1. `pnpm lint` — ESLint (incl. Sheriff boundaries) + Prettier
2. `pnpm test` — unit tests (Vitest)
3. `pnpm build` — production build (includes bundle budgets)
4. `pnpm doctor` — harness compliance
5. If the change touches routes, templates, or user flows: `pnpm e2e`

Only report success after every applicable step passes. Quote the failing
output verbatim when reporting a failure.
```

`src/ng-add/files/.github/workflows/ci.yml.template`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  harness-ci:
    uses: treinberger/angular-harness/.github/workflows/angular-ci.yml@main
    with:
      node-version: '22'
      e2e: <%= e2e %>
```

`src/ng-add/files/playwright.config.ts.template` (only when `e2e`):

```ts
import { defineHarnessPlaywrightConfig } from '@treinberger/harness-testing/playwright';

export default defineHarnessPlaywrightConfig({
  baseURL: 'http://localhost:4200',
  webServerCommand: 'pnpm start',
  port: 4200,
});
```

`src/ng-add/files/e2e/smoke.spec.ts.template` (only when `e2e`):

```ts
import { test, expect } from '@playwright/test';
import { expectNoA11yViolations } from '@treinberger/harness-testing/playwright-a11y';

test('home page renders and is accessible', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('body')).toBeVisible();
  await expectNoA11yViolations(page);
});
```

- [ ] **Step 5: Write the failing schematic test**

`packages/schematics/test/ng-add.test.ts`:

```ts
import { describe, expect, it, beforeEach } from "vitest";
import { SchematicTestRunner } from "@angular-devkit/schematics/testing";
import { Tree } from "@angular-devkit/schematics";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const collectionPath = join(here, "..", "collection.json");

const DEFAULT_OPTIONS = { prefix: "demo", e2e: true, boundaries: true };

function baseAppTree(): Tree {
  const tree = Tree.empty();
  tree.create(
    "/package.json",
    JSON.stringify({ name: "demo", version: "0.0.0", scripts: {}, devDependencies: {} }),
  );
  tree.create(
    "/angular.json",
    JSON.stringify({
      version: 1,
      projects: {
        demo: {
          root: "",
          architect: {
            build: {
              configurations: {
                production: {
                  budgets: [{ type: "initial", maximumWarning: "500kB", maximumError: "1MB" }],
                },
              },
            },
          },
        },
      },
    }),
  );
  tree.create("/tsconfig.json", JSON.stringify({ compilerOptions: {} }));
  return tree;
}

describe("ng-add", () => {
  let runner: SchematicTestRunner;

  beforeEach(() => {
    runner = new SchematicTestRunner("harness", collectionPath);
  });

  it("scaffolds all harness files", async () => {
    const tree = await runner.runSchematic("ng-add", DEFAULT_OPTIONS, baseAppTree());
    for (const file of [
      "/harness.config.json",
      "/eslint.config.mjs",
      "/sheriff.config.ts",
      "/lefthook.yml",
      "/commitlint.config.mjs",
      "/renovate.json",
      "/docs/guidelines/README.md",
      "/CLAUDE.md",
      "/.claude/skills/verify-project/SKILL.md",
      "/.github/workflows/ci.yml",
      "/playwright.config.ts",
      "/e2e/smoke.spec.ts",
    ]) {
      expect(tree.exists(file), `${file} should exist`).toBe(true);
    }
  });

  it("injects the prefix into rendered files", async () => {
    const tree = await runner.runSchematic("ng-add", DEFAULT_OPTIONS, baseAppTree());
    expect(tree.readText("/eslint.config.mjs")).toContain("prefix: 'demo'");
    expect(JSON.parse(tree.readText("/harness.config.json")).prefix).toBe("demo");
  });

  it("extends the harness tsconfig", async () => {
    const tree = await runner.runSchematic("ng-add", DEFAULT_OPTIONS, baseAppTree());
    const tsconfig = JSON.parse(tree.readText("/tsconfig.json"));
    expect(tsconfig.extends).toBe("@treinberger/harness-tsconfig/base.json");
  });

  it("wires package.json: deps, prettier key, scripts", async () => {
    const tree = await runner.runSchematic("ng-add", DEFAULT_OPTIONS, baseAppTree());
    const pkg = JSON.parse(tree.readText("/package.json"));
    for (const dep of [
      "@treinberger/harness-tsconfig",
      "@treinberger/harness-prettier-config",
      "@treinberger/harness-eslint-config",
      "@treinberger/harness-testing",
      "@treinberger/harness-cli",
      "@softarc/sheriff-core",
      "lefthook",
      "@commitlint/cli",
      "@commitlint/config-conventional",
      "@testing-library/angular",
      "@playwright/test",
      "@axe-core/playwright",
    ]) {
      expect(pkg.devDependencies[dep], `${dep} should be a devDependency`).toBeDefined();
    }
    expect(pkg.prettier).toBe("@treinberger/harness-prettier-config");
    expect(pkg.scripts.doctor).toBe("harness doctor");
    expect(pkg.scripts.prepare).toBe("lefthook install");
    expect(pkg.scripts.e2e).toBe("playwright test");
  });

  it("skips playwright and e2e deps when e2e is false", async () => {
    const tree = await runner.runSchematic(
      "ng-add",
      { ...DEFAULT_OPTIONS, e2e: false },
      baseAppTree(),
    );
    expect(tree.exists("/playwright.config.ts")).toBe(false);
    expect(tree.exists("/e2e/smoke.spec.ts")).toBe(false);
    const pkg = JSON.parse(tree.readText("/package.json"));
    expect(pkg.devDependencies["@axe-core/playwright"]).toBeUndefined();
    expect(pkg.scripts.e2e).toBeUndefined();
  });

  it("skips sheriff when boundaries is false", async () => {
    const tree = await runner.runSchematic(
      "ng-add",
      { ...DEFAULT_OPTIONS, boundaries: false },
      baseAppTree(),
    );
    expect(tree.exists("/sheriff.config.ts")).toBe(false);
    expect(tree.readText("/eslint.config.mjs")).toContain("boundaries: false");
    const pkg = JSON.parse(tree.readText("/package.json"));
    expect(pkg.devDependencies["@softarc/sheriff-core"]).toBeUndefined();
  });
});
```

Run: `pnpm --filter @treinberger/harness-schematics build && pnpm test`
Expected: FAIL — factory not written yet (the build fails first; that is the red state).

- [ ] **Step 6: Write `src/ng-add/index.ts`**

```ts
import {
  Rule,
  SchematicContext,
  Tree,
  apply,
  applyTemplates,
  chain,
  filter,
  forEach,
  mergeWith,
  move,
  url,
} from "@angular-devkit/schematics";
import { NodePackageInstallTask } from "@angular-devkit/schematics/tasks";

const HARNESS_VERSION = "v0.1.0";

interface NgAddOptions {
  prefix: string;
  e2e: boolean;
  boundaries: boolean;
}

const CORE_DEV_DEPS: Record<string, string> = {
  "@treinberger/harness-tsconfig": "^0.1.0",
  "@treinberger/harness-prettier-config": "^0.1.0",
  "@treinberger/harness-eslint-config": "^0.1.0",
  "@treinberger/harness-testing": "^0.1.0",
  "@treinberger/harness-cli": "^0.1.0",
  lefthook: "^1",
  "@commitlint/cli": "^19",
  "@commitlint/config-conventional": "^19",
  "@testing-library/angular": "^17",
};

const BOUNDARY_DEV_DEPS: Record<string, string> = {
  "@softarc/sheriff-core": "^0.18",
};

const E2E_DEV_DEPS: Record<string, string> = {
  "@playwright/test": "^1.45",
  "@axe-core/playwright": "^4",
};

// Version note: the ranges above are floors known at plan time; at
// implementation time, set each range to the major that `pnpm add` resolves.

const E2E_FILES = ["playwright.config.ts.template", "e2e/smoke.spec.ts.template"];
const BOUNDARY_FILES = ["sheriff.config.ts.template"];

function scaffoldFiles(options: NgAddOptions): Rule {
  return mergeWith(
    apply(url("./files"), [
      filter((path) => {
        const rel = path.startsWith("/") ? path.slice(1) : path;
        if (!options.e2e && E2E_FILES.includes(rel)) return false;
        if (!options.boundaries && BOUNDARY_FILES.includes(rel)) return false;
        return true;
      }),
      applyTemplates({
        prefix: options.prefix,
        e2e: options.e2e,
        boundaries: options.boundaries,
        harnessVersion: HARNESS_VERSION,
      }),
      // strip the .template suffix
      forEach((entry) => {
        if (entry.path.endsWith(".template")) {
          return {
            content: entry.content,
            path: entry.path.replace(/\.template$/, "") as typeof entry.path,
          };
        }
        return entry;
      }),
      move("/"),
    ]),
  );
}

function extendTsconfig(): Rule {
  return (tree: Tree) => {
    const raw = tree.readText("/tsconfig.json");
    const tsconfig = JSON.parse(raw) as Record<string, unknown>;
    tsconfig["extends"] = "@treinberger/harness-tsconfig/base.json";
    tree.overwrite("/tsconfig.json", JSON.stringify(tsconfig, null, 2) + "\n");
    return tree;
  };
}

function updatePackageJson(options: NgAddOptions): Rule {
  return (tree: Tree) => {
    const pkg = JSON.parse(tree.readText("/package.json")) as {
      scripts?: Record<string, string>;
      devDependencies?: Record<string, string>;
      prettier?: string;
    };
    pkg.devDependencies = {
      ...pkg.devDependencies,
      ...CORE_DEV_DEPS,
      ...(options.boundaries ? BOUNDARY_DEV_DEPS : {}),
      ...(options.e2e ? E2E_DEV_DEPS : {}),
    };
    pkg.prettier = "@treinberger/harness-prettier-config";
    pkg.scripts = {
      ...pkg.scripts,
      lint: "eslint . && prettier --check .",
      doctor: "harness doctor",
      prepare: "lefthook install",
      ...(options.e2e ? { e2e: "playwright test" } : {}),
    };
    tree.overwrite("/package.json", JSON.stringify(pkg, null, 2) + "\n");
    return tree;
  };
}

export function ngAdd(options: NgAddOptions): Rule {
  return (tree: Tree, context: SchematicContext) => {
    context.addTask(new NodePackageInstallTask());
    return chain([scaffoldFiles(options), extendTsconfig(), updatePackageJson(options)])(
      tree,
      context,
    );
  };
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `pnpm --filter @treinberger/harness-schematics build && pnpm test`
Expected: all schematic tests PASS (root vitest `include` pattern `packages/*/test/**/*.test.ts` already matches).

- [ ] **Step 8: Write `packages/schematics/README.md`**

```markdown
# @treinberger/harness-schematics

## Usage (in a fresh `ng new` project)

    # one-time: point the @treinberger scope at GitHub Packages
    echo "@treinberger:registry=https://npm.pkg.github.com" >> .npmrc

    ng add @treinberger/harness-schematics --prefix=myapp

Scaffolds: `harness.config.json`, `eslint.config.mjs`, `sheriff.config.ts`,
`lefthook.yml` + commitlint, `renovate.json`, `CLAUDE.md` +
`.claude/skills/verify-project/`, `docs/guidelines/`, CI workflow,
Playwright config with an a11y smoke test (unless `--e2e=false`); extends
`tsconfig.json`, registers the shared Prettier config, installs all deps.

Flags: `--prefix=<p>` (default `app`), `--e2e=false`, `--boundaries=false`.

## Updates

Future harness majors ship `ng update @treinberger/harness-schematics`
migrations (see `migrations.json`).
```

- [ ] **Step 9: Commit**

```bash
git add packages/schematics pnpm-lock.yaml
git commit -m "feat: add ng-add schematic scaffolding full harness setup"
```

---

### Task 9: Shared Renovate preset

**Files:**
- Create: `renovate/default.json`, `renovate.json`

**Interfaces:**
- Produces: preset referenced by consumers as `github>treinberger/angular-harness//renovate/default.json` (already wired into the Task 8 template).

- [ ] **Step 1: Write `renovate/default.json` (the shared preset)**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:best-practices",
    ":semanticCommits",
    "schedule:weekly"
  ],
  "packageRules": [
    {
      "groupName": "all non-major dependencies",
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "groupName": "harness packages",
      "matchPackageNames": ["@treinberger/harness-*"]
    },
    {
      "groupName": "angular",
      "matchPackageNames": ["@angular/*", "@angular-devkit/*", "@schematics/angular"],
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "matchPackageNames": ["@angular/*", "@angular-devkit/*", "@schematics/angular"],
      "matchUpdateTypes": ["major"],
      "enabled": false,
      "description": "Angular majors are upgraded deliberately via ng update, never by Renovate."
    }
  ]
}
```

- [ ] **Step 2: Write `renovate.json` (this repo's own config)**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>treinberger/angular-harness//renovate/default.json"]
}
```

- [ ] **Step 3: Validate**

Run: `npx --yes renovate-config-validator renovate/default.json renovate.json`
Expected: `Config validated successfully`.

- [ ] **Step 4: Commit**

```bash
git add renovate renovate.json
git commit -m "feat: add shared renovate preset for harness projects"
```

Note: the Renovate GitHub App must be installed on the account and granted access to this repo for consumers to resolve the private preset.

---

### Task 10: Reusable GitHub Actions workflow for consuming projects

**Files:**
- Create: `.github/workflows/angular-ci.yml`

**Interfaces:**
- Consumes: consuming projects' scripts `lint`, `test`, `build`, `doctor`, `e2e` (set up by Task 8).
- Produces: `workflow_call` workflow referenced as `treinberger/angular-harness/.github/workflows/angular-ci.yml@main` (later `@v1` once tagged).

- [ ] **Step 1: Write `.github/workflows/angular-ci.yml`**

```yaml
name: Angular CI (reusable)

on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "22"
      e2e:
        type: boolean
        default: true
      audit:
        type: boolean
        default: true

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: pnpm
          registry-url: https://npm.pkg.github.com
      - run: pnpm install --frozen-lockfile
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - run: pnpm doctor
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
      - name: Audit production dependencies
        if: ${{ inputs.audit }}
        run: pnpm audit --prod --audit-level=high

  e2e:
    if: ${{ inputs.e2e }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: pnpm
          registry-url: https://npm.pkg.github.com
      - run: pnpm install --frozen-lockfile
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

Notes:
- `secrets.GITHUB_TOKEN` reaches package reads only for repos owned by the same account (`treinberger`); cross-owner consumers need a PAT secret (documented in Task 12).
- Bundle-size budgets are enforced by `pnpm build` itself (Angular budgets in `angular.json` — the CLI generates them; tighten per project).
- Renovate's `config:best-practices` pins action digests in consuming repos over time; tags here are the floor.

- [ ] **Step 2: Verify syntax**

Run: `npx --yes action-validator .github/workflows/angular-ci.yml || npx --yes yaml-lint .github/workflows/angular-ci.yml`
Expected: no syntax errors (action-validator preferred; yaml-lint is the fallback if the validator is unavailable).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/angular-ci.yml
git commit -m "feat: add reusable angular-ci workflow with doctor, audit, and e2e"
```

---

### Task 11: Example consumer app (integration proof)

**Files:**
- Create: `examples/demo-app/` (generated by Angular CLI, then wired to the harness)
- Modify: root `package.json`, `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: everything from Tasks 2–10 via the `ng add` schematic, using workspace-local packages.

- [ ] **Step 1: Generate the app**

Run (from repo root):
```bash
pnpm dlx @angular/cli@latest new demo-app --directory examples/demo-app \
  --style=css --ssr=false --skip-git --package-manager=pnpm --defaults
```
Expected: fresh Angular app in `examples/demo-app`.

- [ ] **Step 2: Wire workspace packages**

In `examples/demo-app/package.json`, add:

```json
"devDependencies": {
  "@treinberger/harness-tsconfig": "workspace:*",
  "@treinberger/harness-prettier-config": "workspace:*",
  "@treinberger/harness-eslint-config": "workspace:*",
  "@treinberger/harness-testing": "workspace:*",
  "@treinberger/harness-cli": "workspace:*",
  "@treinberger/harness-schematics": "workspace:*"
}
```

Run: `pnpm install`

- [ ] **Step 3: Run the schematic against the example app**

Run (from `examples/demo-app`, after building schematics and cli):
```bash
pnpm --filter @treinberger/harness-cli build
pnpm --filter @treinberger/harness-schematics build
pnpm exec ng g @treinberger/harness-schematics:ng-add --prefix=demo
pnpm install
```
Expected: full consumer layout created (all files from the Task 8 test list); `tsconfig.json` extended; `package.json` wired. (Using `ng g <collection>:ng-add` instead of `ng add` avoids re-installing from the registry — packages are already linked via workspace.) After scaffolding, change the six `^0.1.0` harness dep entries the schematic wrote back to `workspace:*` so the example keeps using local packages.

- [ ] **Step 4: Prove the toolchain end-to-end**

Run (from `examples/demo-app`):
```bash
pnpm lint
pnpm test
pnpm build
pnpm doctor
```
Expected: all green. Fix any friction **in the packages, not the example** (the example is the integration test). Common issues to expect and fix at the source: missing peer deps in the eslint-config package; sheriff flat-config export name; zoneless provider naming vs. installed Angular major; prettier fighting generated code (run `pnpm exec prettier --write .` once and commit); doctor's version-alignment check vs. `workspace:*` entries (all six are the identical string, so the check passes — if not, teach doctor to treat `workspace:*` as aligned).

- [ ] **Step 5: Prove boundary enforcement (Sheriff)**

Create two features and an illegal cross-feature import:

```bash
mkdir -p src/app/features/alpha src/app/features/beta
```

`src/app/features/alpha/alpha.service.ts`:

```ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AlphaService {
  greet(): string {
    return 'alpha';
  }
}
```

`src/app/features/beta/beta.service.ts`:

```ts
import { Injectable, inject } from '@angular/core';
import { AlphaService } from '../alpha/alpha.service';

@Injectable({ providedIn: 'root' })
export class BetaService {
  private readonly alpha = inject(AlphaService);
}
```

Run: `pnpm lint`
Expected: a Sheriff dependency-rule error on the `beta → alpha` import. Then delete `beta.service.ts`'s illegal import (make `greet` local), re-run, expect green. Keep both services as regression fixtures with a legal shape.

- [ ] **Step 6: Prove the override mechanism**

Add to `examples/demo-app/eslint.config.mjs`:

```js
extraTs: [{ files: ['**/*.ts'], rules: { 'no-console': 'error' } }],
```

Add a `console.log('x');` to `src/app/app.ts`, run `pnpm lint`, expect a `no-console` error; remove the `console.log`, keep the override. Add `examples/demo-app/docs/guidelines/logging.md`:

```markdown
# Logging

**Overrides core:** none — extension only.

`console.*` is forbidden in application code (enforced via ESLint
`no-console`). Use the `Logger` service in `core/logging/` once it exists.
```

- [ ] **Step 7: Prove the testing helpers**

Replace the generated `src/app/app.spec.ts` with:

```ts
import { App } from './app';
import { renderWithHarness } from '@treinberger/harness-testing/testing-library';

describe('App', () => {
  it('renders', async () => {
    const { container } = await renderWithHarness(App);
    expect(container).toBeTruthy();
  });
});
```

Run: `pnpm test` — expect green. Then run the scaffolded a11y smoke e2e:

```bash
pnpm exec playwright install --with-deps chromium
pnpm e2e
```
Expected: `e2e/smoke.spec.ts` passes including `expectNoA11yViolations`. If the default Angular starter page has axe violations, fix the page (it is our template baseline), not the check.

- [ ] **Step 8: Exclude the example's consumer CI and wire root CI**

The scaffolded consumer CI must not run inside this monorepo:

```bash
rm -rf examples/demo-app/.github
```

Add to root `package.json` scripts:

```json
"build:packages": "pnpm -r --filter './packages/*' run --if-present build",
"verify:example": "pnpm --filter demo-app lint && pnpm --filter demo-app test && pnpm --filter demo-app build && pnpm --filter demo-app doctor"
```

Update the `.github/workflows/ci.yml` verify job — builds must run **before** `pnpm test`, because the schematic tests load `collection.json`, which points into `dist/`:

```yaml
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm run build:packages
      - run: pnpm test
      - run: pnpm run verify:example
```

- [ ] **Step 9: Verify everything**

Run: `pnpm run build:packages && pnpm test && pnpm run verify:example`
Expected: all green.

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat: add demo-app proving ng-add, boundaries, overrides, testing helpers"
```

---

### Task 12: Release tooling and adoption docs

**Files:**
- Create: `.changeset/config.json`, `.github/workflows/release.yml`, `docs/consuming-a-project.md`, `docs/adopting-existing-project.md`
- Modify: root `package.json`, `README.md`

**Interfaces:**
- Produces: versioned releases of all packages to GitHub Packages via Changesets; a step-by-step adoption guide.

- [ ] **Step 1: Set up Changesets**

Run: `pnpm add -D -w @changesets/cli && pnpm changeset init`

Edit `.changeset/config.json`:

```json
{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "access": "restricted",
  "baseBranch": "main",
  "fixed": [["@treinberger/harness-*"]],
  "ignore": ["demo-app"]
}
```

(`fixed` keeps all harness packages on one version — doctor's alignment check assumes this.)

Add to root `package.json` scripts:

```json
"release": "pnpm run build:packages && pnpm changeset publish"
```

- [ ] **Step 2: Write `.github/workflows/release.yml`**

```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: write
  packages: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
          registry-url: https://npm.pkg.github.com
      - run: pnpm install --frozen-lockfile
      - uses: changesets/action@v1
        with:
          publish: pnpm release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 3: Write `docs/consuming-a-project.md`**

```markdown
# Adopting the harness in a new Angular project

> For an EXISTING codebase, follow
> [adopting-existing-project.md](adopting-existing-project.md) instead.

## Prerequisites

- Node 22+, pnpm 9+
- Read access to GitHub Packages for the `treinberger` scope. Locally,
  authenticate once:

      npm login --registry=https://npm.pkg.github.com
      # username: your GitHub username, password: a PAT with read:packages

- The Renovate GitHub App installed on the repo (for automated updates).

## Steps

1. `pnpm dlx @angular/cli@latest new my-app --package-manager=pnpm`
2. `cd my-app && echo "@treinberger:registry=https://npm.pkg.github.com" >> .npmrc`
3. `ng add @treinberger/harness-schematics --prefix=myapp`
4. `pnpm exec lefthook install` (also runs automatically via the `prepare` script)
5. Review the generated `CLAUDE.md` and `docs/guidelines/README.md`;
   add project-specific guidelines as separate markdown files.
6. Tag your features in `sheriff.config.ts` as they appear (the scaffolded
   `features/<feature>` placeholder covers the default layout).
7. `pnpm doctor` — must be green before the first commit.
8. Commit. CI runs via the reusable harness workflow.

## Project-specific overrides — extension points

| Concern | How to override |
|---------|-----------------|
| Guidelines | `docs/guidelines/*.md`, marked with "**Overrides core:** …" |
| ESLint | `extraTs` / `extraTemplate` in `eslint.config.mjs` |
| Boundaries | `modules`/`depRules` in `sheriff.config.ts` (document the decision) |
| Prettier | replace the `prettier` package.json key with a `.prettierrc.mjs` spreading the base |
| tsconfig | `compilerOptions` in the project `tsconfig.json` (wins over base) |
| Playwright | extra options passed to `defineHarnessPlaywrightConfig` |
| A11y checks | `disableRules` per call, with a comment explaining why |
| Renovate | additional `packageRules` after the `extends` entry |
| Hooks | edit `lefthook.yml`; `LEFTHOOK=0 git commit` for WIP on private branches |
| CI | inputs of the reusable workflow (`e2e`, `audit`, `node-version`); extra jobs alongside `harness-ci` |

## Updating

Renovate opens a grouped PR for `@treinberger/harness-*` bumps. After
merging: update `harnessVersion` in `harness.config.json` to the matching
harness tag, read the changelog for guideline changes, and run
`ng update @treinberger/harness-schematics` when the changelog says a
migration ships. `pnpm doctor` verifies the result.

## Note on CI package access

`secrets.GITHUB_TOKEN` can read the harness packages only when the consuming
repo belongs to the same owner (`treinberger`). Otherwise create a PAT with
`read:packages` and pass it as a secret.
```

- [ ] **Step 4: Write `docs/adopting-existing-project.md`**

```markdown
# Adopting the harness in an EXISTING Angular project

The harness is not greenfield-only. An existing codebase adopts the same
packages and the same `ng add` — plus a ratchet so existing code lints green
while all new code is held to the full standard.

**Principles**

- Never weaken the harness presets globally; confine relaxations to
  explicitly listed legacy scope.
- Every relaxation is documented and may only ever shrink ("ratchet").
- Prefer running the official Angular migrations over relaxing rules.

## Step 0 — Preconditions

- Working tree clean, CI (or at least build + tests) green before you start.
- Angular ≥ 21. If older, upgrade first, one major at a time:
  `ng update @angular/cli@<major> @angular/core@<major>` — commit per major.
- Switch the repo to pnpm if it isn't already (`corepack enable pnpm`,
  reinstall, commit the lockfile).

## Step 1 — ng add (same as greenfield)

    echo "@treinberger:registry=https://npm.pkg.github.com" >> .npmrc
    ng add @treinberger/harness-schematics --prefix=<existing-prefix>

Use the selector prefix the codebase ALREADY uses — check `angular.json`
(`projects.*.prefix`). Changing the prefix of an existing app is a separate,
deliberate refactor.

If the project has an existing `eslint.config.*`, `.prettierrc*`,
`.husky/`, or CI workflow, the scaffold does not delete them: port any
project-specific rules into the harness extension points (`extraTs`,
`docs/guidelines/`), then delete the old files. Husky is replaced by
lefthook; commit hooks must not run twice.

## Step 2 — Modernize with official Angular migrations

Run each, review the diff, commit separately:

    ng generate @angular/core:standalone      # NgModules → standalone (3 modes; run all)
    ng generate @angular/core:control-flow    # *ngIf/*ngFor → @if/@for
    ng generate @angular/core:inject          # constructor DI → inject()
    ng generate @angular/core:signal-inputs   # @Input() → input()
    ng generate @angular/core:output-migration # @Output() → output()

(Names as of Angular 21 — check `ng generate @angular/core: --help` for the
installed major.) These migrations remove most violations for free.

## Step 3 — TypeScript strictness

`pnpm build` after extending the harness tsconfig. If the error count is
manageable, fix forward. If not, relax ONLY the newly-failing flags in the
project `tsconfig.json` (it wins over the base), e.g.:

    "compilerOptions": {
      "exactOptionalPropertyTypes": false,
      "noUncheckedIndexedAccess": false
    }

Record each relaxation in `docs/guidelines/legacy-baseline.md` (see Step 6).
`strict: true` itself is non-negotiable — if the codebase doesn't compile
under `strict`, fix that first; it predates the harness.

## Step 4 — ESLint ratchet

    pnpm lint

For directories with bulk violations that the migrations couldn't fix, list
them in `eslint.config.mjs`:

    legacyDirs: ['src/app/legacy-billing', 'src/app/admin'],

Result: modernization rules are warnings there (visible, not blocking);
everywhere else they are errors. Do NOT list `src/app` wholesale — pick the
actual offending subtrees, keep the strict surface as large as possible.

## Step 5 — Boundaries (Sheriff)

Map the REAL existing structure in `sheriff.config.ts` `modules`, even if it
is not core/shared/features. Start with `depRules` that describe today's
legal dependencies (lint must pass), then tighten rule by rule as you
untangle. If the structure is a ball of mud, set `boundaries: false` in
`harness.config.json` + `eslint.config.mjs`, document it as an override, and
introduce Sheriff per-subtree later. A wrong-but-green boundary config is
worse than none.

## Step 6 — Write the baseline document

`docs/guidelines/legacy-baseline.md`:

    # Legacy Baseline (ratchet — entries may only be removed)

    **Overrides core:** strict tsconfig flags and modernization lint rules
    are relaxed for the scopes below — because this code predates the
    harness (adopted <date>).

    | Relaxation | Scope | Exit criterion |
    |------------|-------|----------------|
    | exactOptionalPropertyTypes: false | whole project | error count 0 under flag |
    | legacyDirs: src/app/admin | admin feature | feature modernized |

    Boy-scout rule: any PR touching a legacy file upgrades it to the full
    standard and shrinks this table when its scope is done.

## Step 7 — Tests, e2e, doctor

- Existing Karma/Jasmine tests: migrate the runner to Vitest via
  `@angular/build:unit-test` (jasmine-style APIs largely map); new tests use
  `renderWithHarness`.
- Add the scaffolded a11y smoke e2e. If existing pages have axe violations,
  either fix them now or pass `disableRules` with a comment and an entry in
  the baseline table.
- `pnpm doctor` must be green — doctor checks wiring, not code style, so
  legacy code doesn't affect it.

## Step 8 — Done criteria

- `pnpm lint && pnpm test && pnpm build && pnpm doctor` green.
- CI runs the reusable harness workflow.
- `legacy-baseline.md` exists, is referenced from `CLAUDE.md`
  (agents must respect the ratchet), and has an exit criterion per row.
```

- [ ] **Step 5: Replace root `README.md`**

```markdown
# angular-harness

State-of-the-art development harness for Angular web projects: shared tooling
presets, enforced architecture boundaries, layered guidelines (core +
per-project overrides), agentic-development support, compliance doctor,
automated dependency management, and reusable CI.

| Package | Purpose |
|---------|---------|
| `@treinberger/harness-tsconfig` | strict TS base config |
| `@treinberger/harness-prettier-config` | shared Prettier config |
| `@treinberger/harness-eslint-config` | ESLint flat-config factory + Sheriff boundaries |
| `@treinberger/harness-testing` | Playwright factory, axe a11y helper, Testing Library wrapper, zoneless TestBed providers |
| `@treinberger/harness-cli` | `harness doctor` — config-drift detection |
| `@treinberger/harness-schematics` | `ng add` onboarding + `ng update` migrations |

Also in this repo: core guidelines (`guidelines/`), shared Renovate preset
(`renovate/default.json`), reusable CI workflow
(`.github/workflows/angular-ci.yml`).

- **Adopt in a new project:** [docs/consuming-a-project.md](docs/consuming-a-project.md)
- **Adopt in an existing project:** [docs/adopting-existing-project.md](docs/adopting-existing-project.md)
- **Guidelines & layering model:** [guidelines/](guidelines/)
- **Implementation plan:** [docs/plans/](docs/plans/)
```

- [ ] **Step 6: Create the first changeset and verify**

Run:
```bash
pnpm changeset   # select all six packages, minor, message: "initial harness release"
pnpm run build:packages && pnpm test && pnpm lint
```
Expected: changeset file created; all green.

- [ ] **Step 7: Commit and tag**

```bash
git add -A
git commit -m "feat: add changesets release pipeline and adoption docs"
git push
git tag v0.1.0 && git push --tags
```

After the release workflow publishes, update consuming docs/templates if any resolved version differs from `0.1.0`.

---

## Self-Review Checklist (for the implementer, after Task 12)

1. **Fresh-project smoke test:** In a directory outside this repo, follow `docs/consuming-a-project.md` literally against the published packages. Every command must work as written.
2. **Override proof:** In that fresh project, add an ESLint override via `extraTs`, a boundary change via `sheriff.config.ts`, and a guideline override via `docs/guidelines/` — all must take effect / be documented.
3. **Enforcement proof:** A cross-feature import, a missing OnPush, a bad commit message, and a deleted required file must each be caught (Sheriff, ESLint, commitlint, doctor respectively).
4. **CI proof:** Push the fresh project to a private repo under `treinberger`; the reusable workflow must pass end-to-end including doctor, audit, and the a11y e2e smoke test.
5. **Brownfield proof:** Take (or synthesize) an NgModule-based Angular app with `*ngIf` templates and constructor DI; follow `docs/adopting-existing-project.md`. After the migrations plus a `legacyDirs` entry for one deliberately unmigrated directory, `pnpm lint` must be green with warnings confined to that directory, and `legacy-baseline.md` must exist.
6. **Naming consistency:** grep the repo for `@treinberger/harness-` — every reference must match the six package names exactly.
7. **No placeholders:** grep shipped files for `TODO`, `TBD`, `FIXME` — none may remain.

## Known risks / decisions already made

- **Registry = GitHub Packages** (restricted access, free for private use, auth via existing GitHub credentials). Alternative (public npmjs) rejected: harness conventions are internal.
- **Sheriff over eslint-plugin-boundaries / Nx**: Sheriff is purpose-built for Angular standalone architectures, config-light, and needs no workspace migration. Nx rejected as harness-wide default: too invasive for single-app repos.
- **Version drift:** exact dependency majors (angular-eslint, typescript-eslint, sheriff, Playwright, Testing Library, commitlint) are resolved at implementation time; plan steps call out the known rename risks (`prefer-on-push` rule id, sheriff flat-config export, zoneless provider name).
- **`@main` workflow reference** in scaffolded CI until `v1` tag exists; switch the template to `@v1` in a follow-up changeset after the first stable tag.
- **Renovate preset from a private repo** requires the Renovate app to have access to `angular-harness`; without it, consumers see a preset-resolution warning — grant access, don't inline the preset.
- **Storybook / visual regression** deliberately out of scope for v1 (YAGNI for current project sizes); revisit when a design-system package emerges.
- **pnpm catalogs** rejected for now: `fixed` changesets versioning plus doctor's alignment check already prevent drift with less machinery.
- **Windows** is out of scope; harness assumes macOS/Linux dev machines and ubuntu CI runners.
