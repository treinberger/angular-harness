# Angular Project Harness — Implementation Plan

> **For agentic workers:** Implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Each task ends with a passing verification step and a commit. Do not start a task before the previous one is committed. If a `superpowers:subagent-driven-development` or `superpowers:executing-plans` skill is available in your environment, use it; otherwise execute the tasks sequentially in order.

**Goal:** Build a reusable development harness for all future Angular web projects: shared, versioned tooling presets (TypeScript, ESLint, Prettier, testing), a layered guidelines system where every consuming project can add project-specific rules on top of harness-wide core rules, an `ng add` schematic that wires a project up in one command, and reusable CI workflows.

**Architecture:** A pnpm monorepo (`angular-harness`) that publishes small, focused npm packages under the `@treinberger` scope to GitHub Packages. Consuming projects install the packages and run `ng add @treinberger/harness-schematics`, which scaffolds config files, a `harness.config.json`, a layered guidelines directory, a `CLAUDE.md` for agentic work, and a CI workflow that calls this repo's reusable GitHub Actions workflow. Guidelines follow a two-layer model: **core** guidelines live in this repo and are referenced by version; **project** guidelines live in each consuming repo and override core on conflict.

**Tech Stack:** Angular ≥ 21 (standalone, signals, zoneless), TypeScript strict, Node 22 LTS, pnpm, ESLint 9 flat config via `angular-eslint` + `typescript-eslint`, Prettier 3, Vitest (unit, via `@angular/build:unit-test`), Playwright (e2e), Angular Schematics, GitHub Actions reusable workflows, Changesets for versioning/publishing.

## Global Constraints

- Node `>=22.12`, pnpm `>=9`. Use `pnpm` for everything; never `npm install` inside the monorepo.
- All published packages are scoped `@treinberger/*` and published to GitHub Packages (`npm.pkg.github.com`).
- Package names: `@treinberger/harness-tsconfig`, `@treinberger/harness-prettier-config`, `@treinberger/harness-eslint-config`, `@treinberger/harness-testing`, `@treinberger/harness-schematics`.
- All code, docs, comments, and commit messages in **English**.
- Commit style: Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`).
- TypeScript `strict: true` everywhere, including the harness's own source.
- ESM-first: config packages ship `.mjs`/JSON; the schematics package ships CJS (required by `@angular-devkit/schematics`).
- Do not pin exact dependency versions in this plan; install with `pnpm add` (latest) and let the lockfile pin. Peer ranges use the major that `pnpm add` resolves at implementation time.
- Every task ends with green verification (`pnpm test` / documented manual check) before committing.
- Angular-facing opinions baked into presets: standalone components only, `ChangeDetectionStrategy.OnPush` enforced, signals-first, native control flow (`@if`/`@for`), zoneless change detection in tests.

---

## Repository layout (target state)

```
angular-harness/
├── package.json                  # private root, pnpm workspace
├── pnpm-workspace.yaml
├── vitest.config.mts             # runs all package tests
├── .github/workflows/
│   ├── ci.yml                    # CI for this repo itself
│   ├── release.yml               # changesets publish to GitHub Packages
│   └── angular-ci.yml            # REUSABLE workflow consumed by projects
├── packages/
│   ├── tsconfig/                 # @treinberger/harness-tsconfig
│   ├── prettier-config/          # @treinberger/harness-prettier-config
│   ├── eslint-config/            # @treinberger/harness-eslint-config
│   ├── testing/                  # @treinberger/harness-testing
│   └── schematics/               # @treinberger/harness-schematics
├── guidelines/                   # LAYER 0: core guidelines (versioned here)
│   ├── 00-layering.md
│   ├── 01-architecture.md
│   ├── 02-angular-style.md
│   ├── 03-testing.md
│   └── 04-git-workflow.md
├── schemas/
│   └── harness.config.schema.json
├── docs/
│   ├── plans/                    # this plan
│   └── consuming-a-project.md    # how a project adopts the harness
└── examples/
    └── demo-app/                 # consuming Angular app, integration proof
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

- [ ] **Step 9: Verify**

Run: `pnpm lint && pnpm test`
Expected: prettier passes; vitest reports "no test files found" — acceptable only for this task; every later task adds tests. If vitest exits non-zero on empty suite, add `"passWithNoTests": true` to the `test` block in `vitest.config.mts`.

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

### Task 4: `@treinberger/harness-eslint-config`

**Files:**
- Create: `packages/eslint-config/package.json`, `packages/eslint-config/index.mjs`, `packages/eslint-config/README.md`
- Test: `packages/eslint-config/test/lint.test.ts`, `packages/eslint-config/test/fixture/bad.component.ts`, `packages/eslint-config/test/fixture/tsconfig.json`

**Interfaces:**
- Produces: `defineHarnessEslintConfig(options: { prefix?: string; extraTs?: object[]; extraTemplate?: object[] }): FlatConfig[]` — default export is `defineHarnessEslintConfig()` with prefix `app`.

- [ ] **Step 1: Install dependencies for the package**

Run:
```bash
pnpm add -D --filter @treinberger/harness-eslint-config eslint @eslint/js typescript-eslint angular-eslint eslint-config-prettier typescript
```

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

After `pnpm add` resolves versions, move `@eslint/js`, `typescript-eslint`, `angular-eslint`, and `eslint-config-prettier` from `devDependencies` to `dependencies` in this package.json (they are runtime deps of the shared config; `eslint` and `typescript` stay peers).

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

describe("harness-eslint-config", () => {
  it("flags a component selector with the wrong prefix", async () => {
    const eslint = new ESLint({
      cwd: fixtureDir,
      overrideConfigFile: true,
      overrideConfig: defineHarnessEslintConfig({ prefix: "app" }),
    });
    const results = await eslint.lintFiles([join(fixtureDir, "bad.component.ts")]);
    const ruleIds = results.flatMap((r) => r.messages.map((m) => m.ruleId));
    expect(ruleIds).toContain("@angular-eslint/component-selector");
  });

  it("flags a component without OnPush change detection", async () => {
    const eslint = new ESLint({
      cwd: fixtureDir,
      overrideConfigFile: true,
      overrideConfig: defineHarnessEslintConfig({ prefix: "app" }),
    });
    const results = await eslint.lintFiles([join(fixtureDir, "bad.component.ts")]);
    const ruleIds = results.flatMap((r) => r.messages.map((m) => m.ruleId));
    expect(ruleIds).toContain(
      "@angular-eslint/prefer-on-push-component-change-detection",
    );
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

/**
 * Build the harness ESLint flat config.
 *
 * @param {object} [options]
 * @param {string} [options.prefix='app'] Angular selector prefix for this project.
 * @param {object[]} [options.extraTs=[]] Additional flat-config objects appended for *.ts files.
 * @param {object[]} [options.extraTemplate=[]] Additional flat-config objects appended for *.html files.
 * @returns {import('typescript-eslint').ConfigArray}
 */
export function defineHarnessEslintConfig({
  prefix = 'app',
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
    ...extraTemplate,
  );
}

export default defineHarnessEslintConfig();
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `pnpm test`
Expected: PASS. If `prefer-on-push-component-change-detection` reports nothing, verify the rule name against the installed `angular-eslint` major (rule names occasionally move); adjust the test and config together.

- [ ] **Step 6: Write `packages/eslint-config/README.md`**

```markdown
# @treinberger/harness-eslint-config

ESLint 9 flat config for Angular harness projects.

## Usage

`eslint.config.mjs` in the consuming project:

    import { defineHarnessEslintConfig } from '@treinberger/harness-eslint-config';

    export default defineHarnessEslintConfig({
      prefix: 'myapp',
      extraTs: [
        // project-specific rule overrides go here and win over harness defaults
        { files: ['**/*.ts'], rules: { 'no-console': 'error' } },
      ],
    });
```

- [ ] **Step 7: Commit**

```bash
git add packages/eslint-config pnpm-lock.yaml
git commit -m "feat: add @treinberger/harness-eslint-config flat config factory"
```

---

### Task 5: `@treinberger/harness-testing`

**Files:**
- Create: `packages/testing/package.json`, `packages/testing/src/index.ts`, `packages/testing/src/playwright.ts`, `packages/testing/tsconfig.json`, `packages/testing/README.md`
- Test: `packages/testing/test/playwright.test.ts`

**Interfaces:**
- Produces:
  - `defineHarnessPlaywrightConfig(options: { baseURL: string; webServerCommand: string; port: number } & Partial<PlaywrightTestConfig>): PlaywrightTestConfig`
  - `harnessTestProviders(): (Provider | EnvironmentProviders)[]` — zoneless providers for `TestBed.configureTestingModule`.

- [ ] **Step 1: Create package.json and install deps**

`packages/testing/package.json`:

```json
{
  "name": "@treinberger/harness-testing",
  "version": "0.1.0",
  "description": "Shared Playwright config factory and Angular TestBed defaults",
  "license": "MIT",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "exports": {
    ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" },
    "./playwright": { "types": "./dist/playwright.d.ts", "default": "./dist/playwright.js" }
  },
  "scripts": {
    "build": "tsc -p tsconfig.json"
  },
  "peerDependencies": {
    "@angular/core": ">=21",
    "@playwright/test": ">=1.45"
  },
  "peerDependenciesMeta": {
    "@angular/core": { "optional": true },
    "@playwright/test": { "optional": true }
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
pnpm add -D --filter @treinberger/harness-testing typescript @playwright/test @angular/core
```

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

- [ ] **Step 3: Write the failing test**

`packages/testing/test/playwright.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { defineHarnessPlaywrightConfig } from "../src/playwright.js";

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

- [ ] **Step 4: Run test to verify it fails**

Run: `pnpm test`
Expected: FAIL — cannot resolve `../src/playwright.js`.

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

- [ ] **Step 6: Write `packages/testing/src/index.ts`**

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

- [ ] **Step 7: Run tests, then build**

Run: `pnpm test && pnpm --filter @treinberger/harness-testing build`
Expected: tests PASS; `dist/` contains `index.js`, `index.d.ts`, `playwright.js`, `playwright.d.ts`.

- [ ] **Step 8: Write `packages/testing/README.md`**

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

## Unit tests (Vitest via @angular/build:unit-test)

    import { harnessTestProviders } from '@treinberger/harness-testing';

    TestBed.configureTestingModule({
      providers: [...harnessTestProviders()],
    });
```

- [ ] **Step 9: Commit**

```bash
git add packages/testing pnpm-lock.yaml
git commit -m "feat: add @treinberger/harness-testing with playwright factory and zoneless test providers"
```

---

### Task 6: Core guidelines (Layer 0) and config schema

**Files:**
- Create: `guidelines/00-layering.md`, `guidelines/01-architecture.md`, `guidelines/02-angular-style.md`, `guidelines/03-testing.md`, `guidelines/04-git-workflow.md`, `schemas/harness.config.schema.json`
- Test: `packages/schematics/test/schema.test.ts` is added in Task 7 — here, validate the schema manually with `npx ajv` (Step 7).

**Interfaces:**
- Produces: the layering contract every consuming project follows, and the JSON schema that `harness.config.json` files validate against. The schematic (Task 7) copies/references these.

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
4. Tooling presets (tsconfig/eslint/prettier/testing packages) are the
   executable form of core guidelines. Projects override them via the
   documented extension points (`extraTs`, prettier spread, tsconfig
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
- Routing: lazy-load every feature via `loadChildren`/`loadComponent`.
- State: component-local state with signals; cross-component state in
  injectable signal stores (plain services exposing `signal`/`computed`).
  Introduce a state library only via a documented project override.
- HTTP: all API access behind typed service classes in the owning feature
  (or `core/api/` when shared). Components never call `HttpClient` directly.
- Dependency direction: `features → shared/core`. Features must not import
  from other features; extract into `shared`/`core` instead.
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
- TestBed setup uses `harnessTestProviders()` from
  `@treinberger/harness-testing` (zoneless).
- Test behavior through the public API/DOM, not implementation details.
  Prefer `fixture.nativeElement` queries and dispatching real events.
- E2E: Playwright with `defineHarnessPlaywrightConfig`. Smoke-test every
  route; deeper flows for business-critical paths.
- TDD is the default working mode: red → green → refactor. A bugfix starts
  with a failing test reproducing the bug.
- Coverage is a signal, not a gate; do not chase numbers with trivial tests.
```

- [ ] **Step 5: Write `guidelines/04-git-workflow.md`**

```markdown
# Git Workflow

- Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`,
  `refactor:`). Scope optional: `feat(auth): ...`.
- Small, frequent commits; each commit leaves the build green.
- Branch names: `feat/<slug>`, `fix/<slug>`, `chore/<slug>`.
- `main` is protected; changes land via PR with green CI.
- CI must run the reusable workflow
  `treinberger/angular-harness/.github/workflows/angular-ci.yml`.
```

- [ ] **Step 6: Write `schemas/harness.config.schema.json`**

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
    "ci": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "nodeVersion": { "type": "string", "default": "22" },
        "e2e": { "type": "boolean", "default": true }
      }
    }
  }
}
```

- [ ] **Step 7: Validate the schema**

Run:
```bash
echo '{"harnessVersion":"v0.1.0","prefix":"demo"}' > /tmp/hc.json
npx --yes ajv-cli validate -s schemas/harness.config.schema.json -d /tmp/hc.json
```
Expected: `/tmp/hc.json valid`.

- [ ] **Step 8: Commit**

```bash
git add guidelines schemas
git commit -m "docs: add layered core guidelines and harness.config schema"
```

---

### Task 7: `@treinberger/harness-schematics` (`ng add`)

**Files:**
- Create: `packages/schematics/package.json`, `packages/schematics/tsconfig.json`, `packages/schematics/collection.json`, `packages/schematics/src/ng-add/index.ts`, `packages/schematics/src/ng-add/schema.json`, `packages/schematics/src/ng-add/files/…` (template tree, listed in Step 4), `packages/schematics/README.md`
- Test: `packages/schematics/test/ng-add.test.ts`

**Interfaces:**
- Consumes: package names from Tasks 2–5; schema from Task 6; reusable workflow name from Task 8 (`angular-ci.yml` — fixed here, implemented there).
- Produces: `ng add @treinberger/harness-schematics --prefix=<prefix>` which scaffolds all harness files into a consuming project.

- [ ] **Step 1: Create package.json, tsconfig, and install deps**

`packages/schematics/package.json`:

```json
{
  "name": "@treinberger/harness-schematics",
  "version": "0.1.0",
  "description": "ng add schematic that wires an Angular project into the harness",
  "license": "MIT",
  "schematics": "./collection.json",
  "files": ["collection.json", "dist"],
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

Run:
```bash
pnpm add --filter @treinberger/harness-schematics @angular-devkit/schematics @angular-devkit/core @schematics/angular
pnpm add -D --filter @treinberger/harness-schematics typescript
```

`packages/schematics/copy-assets.mjs` (copies non-TS template files into dist):

```js
import { cpSync } from "node:fs";
cpSync("src/ng-add/files", "dist/ng-add/files", { recursive: true });
cpSync("src/ng-add/schema.json", "dist/ng-add/schema.json");
```

- [ ] **Step 2: Write `collection.json`**

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
    }
  },
  "required": []
}
```

- [ ] **Step 4: Write the template file tree `src/ng-add/files/`**

Files use the schematics template syntax (`<%= prefix %>`). Create:

`src/ng-add/files/harness.config.json.template`:

```json
{
  "$schema": "https://raw.githubusercontent.com/treinberger/angular-harness/main/schemas/harness.config.schema.json",
  "harnessVersion": "<%= harnessVersion %>",
  "prefix": "<%= prefix %>"
}
```

`src/ng-add/files/eslint.config.mjs.template`:

```js
import { defineHarnessEslintConfig } from '@treinberger/harness-eslint-config';

export default defineHarnessEslintConfig({
  prefix: '<%= prefix %>',
  // Project-specific rule overrides win over harness defaults:
  extraTs: [],
  extraTemplate: [],
});
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
   feature-first structure, TDD with Vitest, Playwright e2e,
   Conventional Commits.
2. Project guidelines (Layer 1): `docs/guidelines/` in this repo.
   **On conflict, project guidelines win.**

## Commands

- Dev server: `pnpm start`
- Unit tests: `pnpm test`
- Lint: `pnpm lint`
- E2E: `pnpm e2e`

## Hard rules

- Selector prefix: `<%= prefix %>` (see `harness.config.json`).
- Never disable the shared ESLint/Prettier/tsconfig presets wholesale;
  extend them via their documented extension points.
- Every bugfix starts with a failing test.
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

`src/ng-add/files/playwright.config.ts.template` (only written when `e2e` is true — handled in the factory):

```ts
import { defineHarnessPlaywrightConfig } from '@treinberger/harness-testing/playwright';

export default defineHarnessPlaywrightConfig({
  baseURL: 'http://localhost:4200',
  webServerCommand: 'pnpm start',
  port: 4200,
});
```

Note: name every template file with a `.template` suffix in the source tree and strip it during rendering (see the `rename` in Step 6). This prevents the monorepo's own tooling from linting template placeholders.

- [ ] **Step 5: Write the failing schematic test**

`packages/schematics/test/ng-add.test.ts`:

```ts
import { describe, expect, it, beforeEach } from "vitest";
import { SchematicTestRunner, UnitTestTree } from "@angular-devkit/schematics/testing";
import { Tree } from "@angular-devkit/schematics";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";

const here = dirname(fileURLToPath(import.meta.url));
const collectionPath = join(here, "..", "collection.json");

function baseAppTree(): Tree {
  const tree = Tree.empty();
  tree.create(
    "/package.json",
    JSON.stringify({ name: "demo", version: "0.0.0", scripts: {}, devDependencies: {} }),
  );
  tree.create("/angular.json", JSON.stringify({ version: 1, projects: { demo: { root: "" } } }));
  tree.create("/tsconfig.json", JSON.stringify({ compilerOptions: {} }));
  return tree;
}

describe("ng-add", () => {
  let runner: SchematicTestRunner;

  beforeEach(() => {
    runner = new SchematicTestRunner("harness", collectionPath);
  });

  it("scaffolds all harness files", async () => {
    const tree = await runner.runSchematic<{ prefix: string; e2e: boolean }>(
      "ng-add",
      { prefix: "demo", e2e: true },
      baseAppTree(),
    );
    for (const file of [
      "/harness.config.json",
      "/eslint.config.mjs",
      "/docs/guidelines/README.md",
      "/CLAUDE.md",
      "/.github/workflows/ci.yml",
      "/playwright.config.ts",
    ]) {
      expect(tree.exists(file), `${file} should exist`).toBe(true);
    }
  });

  it("injects the prefix into rendered files", async () => {
    const tree = await runner.runSchematic("ng-add", { prefix: "demo", e2e: true }, baseAppTree());
    expect(tree.readText("/eslint.config.mjs")).toContain("prefix: 'demo'");
    expect(JSON.parse(tree.readText("/harness.config.json")).prefix).toBe("demo");
  });

  it("extends the harness tsconfig", async () => {
    const tree = await runner.runSchematic("ng-add", { prefix: "demo", e2e: true }, baseAppTree());
    const tsconfig = JSON.parse(tree.readText("/tsconfig.json"));
    expect(tsconfig.extends).toBe("@treinberger/harness-tsconfig/base.json");
  });

  it("adds harness packages to devDependencies and sets the prettier key", async () => {
    const tree = await runner.runSchematic("ng-add", { prefix: "demo", e2e: true }, baseAppTree());
    const pkg = JSON.parse(tree.readText("/package.json"));
    expect(pkg.devDependencies["@treinberger/harness-eslint-config"]).toBeDefined();
    expect(pkg.devDependencies["@treinberger/harness-tsconfig"]).toBeDefined();
    expect(pkg.devDependencies["@treinberger/harness-testing"]).toBeDefined();
    expect(pkg.prettier).toBe("@treinberger/harness-prettier-config");
  });

  it("skips playwright when e2e is false", async () => {
    const tree = await runner.runSchematic("ng-add", { prefix: "demo", e2e: false }, baseAppTree());
    expect(tree.exists("/playwright.config.ts")).toBe(false);
  });
});
```

Run: `pnpm --filter @treinberger/harness-schematics build && pnpm test`
Expected: FAIL — `dist/ng-add/index` not found (factory not written yet). The build itself fails first; that is the red state.

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
  mergeWith,
  move,
  url,
  forEach,
} from "@angular-devkit/schematics";
import { NodePackageInstallTask } from "@angular-devkit/schematics/tasks";

const HARNESS_VERSION = "v0.1.0";

interface NgAddOptions {
  prefix: string;
  e2e: boolean;
}

const HARNESS_DEV_DEPS: Record<string, string> = {
  "@treinberger/harness-tsconfig": "^0.1.0",
  "@treinberger/harness-prettier-config": "^0.1.0",
  "@treinberger/harness-eslint-config": "^0.1.0",
  "@treinberger/harness-testing": "^0.1.0",
};

function scaffoldFiles(options: NgAddOptions): Rule {
  return mergeWith(
    apply(url("./files"), [
      filter((path) => options.e2e || !path.endsWith("playwright.config.ts.template")),
      applyTemplates({
        prefix: options.prefix,
        e2e: options.e2e,
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
    pkg.devDependencies = { ...pkg.devDependencies, ...HARNESS_DEV_DEPS };
    pkg.prettier = "@treinberger/harness-prettier-config";
    pkg.scripts = {
      ...pkg.scripts,
      lint: "eslint . && prettier --check .",
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
Expected: all 5 schematic tests PASS. Add `"packages/schematics/test/**/*.test.ts"` to the root `vitest.config.mts` include list if not already matched.

- [ ] **Step 8: Write `packages/schematics/README.md`**

```markdown
# @treinberger/harness-schematics

## Usage (in a fresh `ng new` project)

    # one-time: point the @treinberger scope at GitHub Packages
    echo "@treinberger:registry=https://npm.pkg.github.com" >> .npmrc

    ng add @treinberger/harness-schematics --prefix=myapp

Scaffolds: `harness.config.json`, `eslint.config.mjs`, `CLAUDE.md`,
`docs/guidelines/`, CI workflow, Playwright config (unless `--e2e=false`),
extends `tsconfig.json`, registers the shared Prettier config, and installs
the harness packages.
```

- [ ] **Step 9: Commit**

```bash
git add packages/schematics vitest.config.mts pnpm-lock.yaml
git commit -m "feat: add ng-add schematic scaffolding harness setup into projects"
```

---

### Task 8: Reusable GitHub Actions workflow for consuming projects

**Files:**
- Create: `.github/workflows/angular-ci.yml`

**Interfaces:**
- Consumes: consuming projects' scripts `lint`, `test`, `build`, `e2e` (set up by Task 7).
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
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build

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

Note: `secrets.GITHUB_TOKEN` reaches package reads only for repos owned by the same account/org. Document this in Task 10's consuming guide; cross-org consumers need a PAT secret instead.

- [ ] **Step 2: Verify syntax**

Run: `npx --yes action-validator .github/workflows/angular-ci.yml || npx --yes yaml-lint .github/workflows/angular-ci.yml`
Expected: no syntax errors (action-validator preferred; yaml-lint is the fallback if the validator is unavailable).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/angular-ci.yml
git commit -m "feat: add reusable angular-ci workflow for consuming projects"
```

---

### Task 9: Example consumer app (integration proof)

**Files:**
- Create: `examples/demo-app/` (generated by Angular CLI, then wired to the harness)

**Interfaces:**
- Consumes: everything from Tasks 2–8 via the `ng add` schematic, using workspace-local packages.

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
  "@treinberger/harness-schematics": "workspace:*"
}
```

Run: `pnpm install`

- [ ] **Step 3: Run the schematic against the example app**

Run (from `examples/demo-app`, after building schematics):
```bash
pnpm --filter @treinberger/harness-schematics build
pnpm exec ng g @treinberger/harness-schematics:ng-add --prefix=demo
```
Expected: `harness.config.json`, `eslint.config.mjs`, `CLAUDE.md`, `docs/guidelines/`, `.github/workflows/ci.yml`, `playwright.config.ts` created; `tsconfig.json` extended; `package.json` updated. (Using `ng g <collection>:ng-add` instead of `ng add` avoids re-installing from the registry — packages are already linked via workspace.)

- [ ] **Step 4: Prove the toolchain end-to-end**

Run (from `examples/demo-app`):
```bash
pnpm lint
pnpm test
pnpm build
```
Expected: all green. Fix any friction **in the packages, not the example** (the example is the integration test). Common issues to expect and fix at the source: missing peer deps in the eslint-config package; `harnessTestProviders()` naming vs. installed Angular major; prettier fighting the generated code (run `pnpm exec prettier --write .` once and commit).

- [ ] **Step 5: Prove the override mechanism**

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

- [ ] **Step 6: Exclude the example from publishing and keep CI green**

Ensure `examples/demo-app/.github/` is deleted (the scaffolded consumer CI must not run inside this monorepo — it's only meaningful in a standalone repo):

```bash
rm -rf examples/demo-app/.github
```

Add example verification to root `package.json` scripts:

```json
"verify:example": "pnpm --filter demo-app lint && pnpm --filter demo-app test && pnpm --filter demo-app build"
```

And update the `.github/workflows/ci.yml` verify job — the build steps must run **before** `pnpm test`, because the schematic tests load `collection.json`, which points into `dist/`:

```yaml
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm --filter @treinberger/harness-testing build
      - run: pnpm --filter @treinberger/harness-schematics build
      - run: pnpm test
      - run: pnpm run verify:example
```

- [ ] **Step 7: Verify everything**

Run: `pnpm test && pnpm run verify:example`
Expected: all green.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: add demo-app example proving ng-add integration and override layering"
```

---

### Task 10: Release tooling and adoption docs

**Files:**
- Create: `.changeset/config.json`, `.github/workflows/release.yml`, `docs/consuming-a-project.md`, `README.md`
- Modify: root `package.json`

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
  "ignore": ["demo-app"]
}
```

Add to root `package.json` scripts:

```json
"release": "pnpm --filter './packages/*' build && pnpm changeset publish"
```

(`--filter './packages/*'` with `build` only runs where a build script exists — `testing` and `schematics`; config packages have nothing to build. If pnpm errors on packages without the script, use `--if-present`: `pnpm -r --filter './packages/*' run --if-present build`.)

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

## Prerequisites

- Node 22+, pnpm 9+
- Read access to GitHub Packages for the `treinberger` scope. Locally,
  authenticate once:

      npm login --registry=https://npm.pkg.github.com
      # username: your GitHub username, password: a PAT with read:packages

## Steps

1. `pnpm dlx @angular/cli@latest new my-app --package-manager=pnpm`
2. `cd my-app && echo "@treinberger:registry=https://npm.pkg.github.com" >> .npmrc`
3. `ng add @treinberger/harness-schematics --prefix=myapp`
4. Review the generated `CLAUDE.md` and `docs/guidelines/README.md`;
   add project-specific guidelines as separate markdown files.
5. Commit. CI runs via the reusable harness workflow.

## Project-specific overrides — extension points

| Concern | How to override |
|---------|-----------------|
| Guidelines | `docs/guidelines/*.md`, marked with "**Overrides core:** …" |
| ESLint | `extraTs` / `extraTemplate` in `eslint.config.mjs` |
| Prettier | replace the `prettier` package.json key with a `.prettierrc.mjs` spreading the base |
| tsconfig | `compilerOptions` in the project `tsconfig.json` (wins over base) |
| Playwright | extra options passed to `defineHarnessPlaywrightConfig` |
| CI | inputs of the reusable workflow; extra jobs alongside `harness-ci` |

## Updating

Bump the `@treinberger/harness-*` versions, run `pnpm install`, and update
`harnessVersion` in `harness.config.json` to the matching harness tag.
Read the changelog for guideline changes.

## Note on CI package access

`secrets.GITHUB_TOKEN` can read the harness packages only when the consuming
repo belongs to the same owner (`treinberger`). Otherwise create a PAT with
`read:packages` and pass it as a secret.
```

- [ ] **Step 4: Write root `README.md`**

```markdown
# angular-harness

Reusable development harness for Angular web projects: shared tooling
presets, layered guidelines (core + per-project overrides), one-command
project onboarding via `ng add`, and reusable CI.

| Package | Purpose |
|---------|---------|
| `@treinberger/harness-tsconfig` | strict TS base config |
| `@treinberger/harness-prettier-config` | shared Prettier config |
| `@treinberger/harness-eslint-config` | ESLint flat-config factory |
| `@treinberger/harness-testing` | Playwright factory + zoneless TestBed providers |
| `@treinberger/harness-schematics` | `ng add` onboarding schematic |

- **Adopt in a project:** see [docs/consuming-a-project.md](docs/consuming-a-project.md)
- **Guidelines & layering model:** see [guidelines/](guidelines/)
- **Implementation plan:** see [docs/plans/](docs/plans/)
```

This replaces the bootstrap README committed when the repo was created.

- [ ] **Step 5: Create the first changeset and verify**

Run:
```bash
pnpm changeset   # select all five packages, minor, message: "initial harness release"
pnpm test && pnpm lint
```
Expected: changeset file created; all green.

- [ ] **Step 6: Commit and tag**

```bash
git add -A
git commit -m "feat: add changesets release pipeline and adoption docs"
git push
git tag v0.1.0 && git push --tags
```

After the release workflow publishes, update consuming docs/templates if any resolved version differs from `0.1.0`.

---

## Self-Review Checklist (for the implementer, after Task 10)

1. **Fresh-project smoke test:** In a directory outside this repo, follow `docs/consuming-a-project.md` literally against the published packages. Every command must work as written.
2. **Override proof:** In that fresh project, add an ESLint override via `extraTs` and a guideline override via `docs/guidelines/` — both must take effect / be documented.
3. **CI proof:** Push the fresh project to a private repo under `treinberger`; the reusable workflow must pass.
4. **Naming consistency:** grep the repo for `@treinberger/harness-` — every reference must match the five package names exactly.
5. **No placeholders:** grep for `TODO`, `TBD`, `FIXME` — none may remain in shipped files.

## Known risks / decisions already made

- **Registry = GitHub Packages** (restricted access, free for private use, auth via existing GitHub credentials). Alternative (public npmjs) rejected: harness conventions are internal.
- **Version drift:** exact dependency majors (angular-eslint, typescript-eslint, Playwright) are resolved at implementation time; two plan steps (4.5, 5.6) call out the known rename risks (`prefer-on-push` rule id, zoneless provider name).
- **`@main` workflow reference** in scaffolded CI until `v1` tag exists; switch the template to `@v1` in a follow-up changeset after the first stable tag.
- **Windows** is out of scope; harness assumes macOS/Linux dev machines and ubuntu CI runners.
