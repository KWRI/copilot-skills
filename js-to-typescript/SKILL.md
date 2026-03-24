---
name: js-to-typescript
description: Migrates legacy JavaScript (or Flow-typed JS) to modern TypeScript. Use this skill when asked to convert JS to TS, migrate a file or directory to TypeScript, migrate a specific feature or module to TypeScript, add TypeScript to a project, replace PropTypes with TypeScript interfaces, or modernize JavaScript code with strong typing. Supports both full-repo migration and scoped/feature-by-feature migration.
---

# JavaScript → TypeScript Migration Skill

Perform a thorough migration from legacy JavaScript to modern TypeScript. This skill supports two modes — choose based on the user's request, or ask if it's unclear:

| Mode | When to use |
|------|-------------|
| **Full-repo** | User wants the entire codebase migrated |
| **Scoped** | User names a specific feature, directory, component, or set of files to migrate first |

**If the user's request includes a feature name, path, or component name → use Scoped Mode (Step 1-S).**
**If the user asks to migrate everything or doesn't specify a scope → use Full-Repo Mode (Step 1-F).**

Both modes share Steps 2–15. Scoped mode adds boundary-management rules that keep the unmigrated JS working alongside the new TS.

---

## Step 1-S — Scoped Migration: Target Assessment

> Use this step instead of Step 1-F when the user specifies a feature, directory, or set of files.

### A. Clarify the scope

If the user gave a feature name (e.g. "the auth module", "the Button component", "the payments feature") but not a path, locate it first:

```bash
# Search by directory name
find src -type d -iname "*<feature>*" 2>/dev/null

# Search by file name pattern
find src -type f \( -name "*.js" -o -name "*.jsx" \) -ipath "*<feature>*" 2>/dev/null

# Search by export name
grep -rl "export.*<FeatureName>\|function <FeatureName>" src/ --include="*.js" --include="*.jsx" 2>/dev/null
```

Confirm the resolved path(s) with the user before proceeding if there is any ambiguity.

### B. Map the target files

```bash
TARGET="src/features/<feature>"  # substitute the resolved path

# All JS/JSX files in scope
find "$TARGET" -name "*.js" -o -name "*.jsx" 2>/dev/null

# Already-TS files (may already be partially migrated)
find "$TARGET" -name "*.ts" -o -name "*.tsx" 2>/dev/null
```

### C. Map inbound and outbound dependencies

Understanding the dependency boundary is critical — it determines what needs stub types vs. what can be fully typed.

```bash
# What does this feature import FROM outside itself? (outbound deps)
grep -rh "^import\|require(" "$TARGET" --include="*.js" --include="*.jsx" 2>/dev/null \
  | grep -v "from '\." \
  | sort -u

# What imports FROM this feature? (inbound consumers — still in JS)
grep -rl "from '.*<feature>\|require('.*<feature>" src/ \
  --include="*.js" --include="*.jsx" 2>/dev/null \
  | grep -v "$TARGET"
```

Report these two lists to the user:
- **Outbound deps** — packages or other modules this feature depends on (need `@types/*` or stubs)
- **Inbound consumers** — JS files that import from this feature (will consume the new TS exports; may need declaration stubs if not migrated yet)

### D. Assess the target files

For each file in scope, note:
- File name and type (utility, component, hook, context, test)
- PropTypes present? (`grep -l "PropTypes" ...`)
- Flow types present? (`grep -l "@flow" ...`)
- CommonJS? (`grep -l "require(" ...`)
- Approximate line count (`wc -l`)

Report a complexity estimate:
- **Simple** — pure functions, no JSX, clear inputs/outputs
- **Moderate** — React components with props/state, some async
- **Complex** — dynamic types, HOCs, render props, heavily mocked in tests

### E. Ask the user

1. Should tests for this feature be migrated in the same pass, or deferred?
2. Preferred strictness for this scope: **strict** or **lenient** (can tighten later)?
3. Are there other features that should be migrated together because they share types?

Then proceed to **Step 1-S-Boundary** before moving on to Step 2.

---

### Step 1-S-Boundary — Establish the JS/TS Boundary

When only part of the repo is migrated, you must protect the unmigrated JS consumers so they keep working.

#### Ensure `allowJs: true` in `tsconfig.json`

This is non-negotiable for scoped migration. The TypeScript compiler must be able to see both `.js` and `.ts` files simultaneously.

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false
  }
}
```

`checkJs: false` means JS files are imported without being type-checked — this prevents a flood of errors from unmigrated files.

#### Verify `include` is broad enough

`tsconfig.json`'s `include` must cover both the migrated TS files and the remaining JS files:

```json
{
  "include": ["src/**/*"]
}
```

Do **not** narrow `include` to only `.ts` files — that would break `allowJs`.

#### For inbound JS consumers of the new TS module

JS files that `import` from the newly migrated TS module will work automatically as long as `allowJs: true` and the build tool resolves `.ts` extensions. Verify the build tool's `resolve.extensions` includes `.ts`/`.tsx` (see Step 4). No other changes are needed in the consuming JS files.

#### For outbound TS imports of unmigrated JS modules

When the newly migrated TS file imports from a JS module that has no types:

```ts
// TypeScript will infer the type as `any` for JS modules — that's acceptable during migration.
// If you need accurate types for an internal JS module, create a co-located stub:
```

Create `src/utils/legacyHelper.d.ts` alongside `src/utils/legacyHelper.js`:

```ts
// Minimal stub — replace with real types when that module is migrated
export declare function doThing(input: string): string;
export declare const CONFIG: Record<string, unknown>;
```

> Only create stubs for JS modules that the newly migrated TS code calls directly. Do not stub every unmigrated file — that would be premature work.

#### Track what still needs migration

After finishing a scoped pass, record what was migrated and what remains. Use a tracking comment at the top of `src/types/migration-status.d.ts` (create this file if it doesn't exist):

```ts
/**
 * MIGRATION STATUS
 * ----------------
 * Migrated (TS):
 *   - src/features/auth/**
 *   - src/components/Button.*
 *
 * Remaining (JS):
 *   - src/features/payments/**  ← next target
 *   - src/utils/legacyHelper.js ← stub created
 *   - src/pages/**
 */
```

This file serves as a living record visible to the whole team.

---

## Step 1-F — Full-Repo Migration: Assessment

> Use this step when migrating the entire codebase at once.

Before touching any files, build a full picture of the codebase.

### A. Detect project shape (run in parallel)

```bash
# Package manager
ls pnpm-lock.yaml yarn.lock package-lock.json 2>/dev/null | head -1

# Existing TS setup
cat tsconfig.json 2>/dev/null || echo "NO TSCONFIG"

# File counts
find src -name "*.js" -o -name "*.jsx" | wc -l
find src -name "*.ts" -o -name "*.tsx" | wc -l

# Flow type usage
grep -rl "@flow" src/ 2>/dev/null | wc -l

# PropTypes usage
grep -rl "PropTypes" src/ 2>/dev/null | wc -l

# CommonJS vs ESM
grep -rl "require(" src/ --include="*.js" 2>/dev/null | wc -l
grep -rl "^import " src/ --include="*.js" 2>/dev/null | wc -l
```

### B. Inspect build tooling

Check for and read (in parallel, if they exist):
- `webpack.config.js` / `rspack.*.js` / `vite.config.js` / `rollup.config.js`
- `babel.config.js` / `.babelrc`
- `jest.config.js` / `jest.setup.js`
- `package.json` (scripts, dependencies, devDependencies)
- `.eslintrc*`

### C. Summarize findings

Report to the user before proceeding:
- Number of `.js` / `.jsx` files to migrate
- Number already `.ts` / `.tsx`
- Whether Flow types are present
- Whether PropTypes are in use
- Build tool in use
- Whether a `tsconfig.json` already exists
- Estimated complexity: **Low** (< 20 files), **Medium** (20–100 files), **Large** (100+ files)

Ask the user:
1. Should migration be **all at once** or **incremental** (file by file, starting with leaf modules)?
2. Preferred strictness: **strict** (`"strict": true`) or **lenient** (`noImplicitAny: false` to start)?
3. Any directories or files to **exclude** from migration?

---

## Step 2 — Install TypeScript & Type Definitions

### A. Install core packages

```bash
# pnpm
pnpm add -D typescript @types/node

# npm
npm install -D typescript @types/node

# yarn
yarn add -D typescript @types/node
```

### B. Install framework-specific types (install whichever apply)

```bash
# React
pnpm add -D @types/react @types/react-dom

# Testing
pnpm add -D @types/jest @types/testing-library__react

# Node utilities commonly needed
pnpm add -D @types/lodash @types/classnames @types/uuid

# Any other untyped packages — discover them with:
grep '"dependencies"' -A 200 package.json | grep -oP '"[^"]+":' | tr -d '":' | \
  xargs -I{} sh -c 'npm info @types/{} version 2>/dev/null && echo "AVAILABLE: @types/{}"' 2>/dev/null | grep AVAILABLE
```

> **Note:** Only install `@types/*` for packages that don't already ship their own types. Check `node_modules/<pkg>/index.d.ts` or the `"types"` field in the package's `package.json` first.

---

## Step 3 — Create or Update `tsconfig.json`

### A. If no `tsconfig.json` exists, generate one

```bash
npx tsc --init
```

### B. Apply the recommended configuration

Edit `tsconfig.json` to match this baseline (adjust `paths` to fit the project):

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "allowJs": true,
    "checkJs": false,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noImplicitReturns": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./build",
    "baseUrl": ".",
    "paths": {}
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "build", "dist", "coverage", "**/*.test.js", "**/*.spec.js"]
}
```

**Key flags explained:**
- `allowJs: true` — lets already-TS files import remaining `.js` files during incremental migration
- `checkJs: false` — avoids type errors on not-yet-migrated JS files
- `strict: true` — enables all strict checks; change to `false` temporarily for a lenient first pass
- `skipLibCheck: true` — prevents noise from third-party `.d.ts` issues

### C. If `tsconfig.json` already exists

Read it first, then only update fields that are incorrect or missing. Do **not** overwrite customizations.

---

## Step 4 — Update Build Tooling

### Babel

If `babel.config.js` / `.babelrc` exists and uses `@babel/preset-env` or `@babel/preset-react`, add TypeScript support:

```bash
pnpm add -D @babel/preset-typescript
```

Add to presets array in babel config:
```js
'@babel/preset-typescript'
```

> Babel strips types (no type checking at build time) — `tsc --noEmit` handles type checking separately. This is the recommended approach for most bundlers.

### webpack

If `webpack.config.js` exists, check for `ts-loader` or `babel-loader`:

```bash
# If using ts-loader directly:
pnpm add -D ts-loader

# If using babel-loader (preferred — faster):
# Just add @babel/preset-typescript to babel config (done above)
```

Add/update the `resolve.extensions` array in webpack config:
```js
resolve: {
  extensions: ['.ts', '.tsx', '.js', '.jsx', '.json']
}
```

Add TypeScript to the module rules if using `ts-loader`:
```js
{
  test: /\.(ts|tsx)$/,
  use: 'ts-loader',
  exclude: /node_modules/
}
```

### rspack

rspack has built-in TypeScript support via SWC. In `rspack.config.js`:
```js
resolve: {
  extensions: ['.ts', '.tsx', '.js', '.jsx', '.json']
}
```

No additional loader is needed unless type checking at build time is required.

### Vite

Vite handles TypeScript natively. Only ensure `tsconfig.json` is present.

### Jest

If `jest.config.js` exists, add TypeScript transform support:

```bash
pnpm add -D ts-jest
# OR (preferred with Babel):
pnpm add -D babel-jest  # already installed if babel is in use
```

Update `jest.config.js`:
```js
transform: {
  '^.+\\.(ts|tsx|js|jsx)$': 'babel-jest'  // if using Babel
  // OR
  '^.+\\.(ts|tsx)$': 'ts-jest'            // if using ts-jest
},
moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json', 'node'],
```

---

## Step 5 — Prioritize Migration Order

Migrate files in dependency order — **leaf modules first, entry points last**.

```bash
# Find files with fewest imports (good leaf candidates)
for f in $(find src -name "*.js" -o -name "*.jsx"); do
  echo "$(grep -c "^import\|require(" $f 2>/dev/null) $f"
done | sort -n | head -30
```

**Recommended order:**
1. Type definition files (create `src/types/` or `src/@types/`)
2. Utility / helper functions (pure functions, no JSX)
3. Constants and enums
4. Custom hooks
5. Context providers
6. Reusable UI components (leaf components first)
7. Container / page components
8. Entry points (`index.js`, `App.js`)

---

## Step 6 — Migrate Each File

For each file, follow this process:

### A. Rename the file

```bash
# .js → .ts (no JSX)
mv src/utils/formatDate.js src/utils/formatDate.ts

# .jsx → .tsx (contains JSX)
mv src/components/Button.jsx src/components/Button.tsx
```

### B. Convert CommonJS to ES module syntax (if needed)

```js
// BEFORE (CommonJS)
const React = require('react');
const { useState } = require('react');
module.exports = { MyComponent };

// AFTER (ESM)
import React, { useState } from 'react';
export { MyComponent };
```

### C. Add types to function signatures

```ts
// BEFORE
function formatDate(date, locale) {
  return date.toLocaleDateString(locale);
}

// AFTER
function formatDate(date: Date, locale: string = 'en-US'): string {
  return date.toLocaleDateString(locale);
}
```

### D. Convert PropTypes → TypeScript interfaces

```tsx
// BEFORE
import PropTypes from 'prop-types';

function Button({ label, onClick, disabled }) {
  return <button onClick={onClick} disabled={disabled}>{label}</button>;
}

Button.propTypes = {
  label: PropTypes.string.isRequired,
  onClick: PropTypes.func.isRequired,
  disabled: PropTypes.bool,
};

Button.defaultProps = {
  disabled: false,
};

// AFTER
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}

function Button({ label, onClick, disabled = false }: ButtonProps) {
  return <button onClick={onClick} disabled={disabled}>{label}</button>;
}
```

> **Note:** Keep `prop-types` in `package.json` if any `.js` files remain. Remove it only when 100% of files have been migrated.

### E. Convert Flow types → TypeScript types

```ts
// BEFORE (Flow)
// @flow
type User = { id: number, name: string };
function greet(user: User): string { ... }

// AFTER (TypeScript)
type User = { id: number; name: string };
function greet(user: User): string { ... }
```

Flow-to-TS differences to watch for:
- Flow uses `,` in object types; TypeScript uses `;`
- `?Type` (Flow nullable) → `Type | null | undefined` (TS)
- `mixed` → `unknown`
- `*` (existential type) → `any` (then refine)
- `$Keys<T>` → `keyof T`
- `$Values<T>` → `T[keyof T]`
- `$Exact<T>` → remove (TS interfaces are exact by default for assignments)
- `Class<T>` → `new (...args: any[]) => T`

### F. Add return types to React components

```tsx
// Function components
function MyComponent(props: MyProps): JSX.Element { ... }
// OR (preferred modern style)
const MyComponent = (props: MyProps): React.ReactElement => { ... }

// Components that may return null
const ConditionalComponent = (props: Props): React.ReactElement | null => { ... }
```

### G. Type event handlers

```tsx
// Mouse events
const handleClick = (e: React.MouseEvent<HTMLButtonElement>): void => { ... }

// Change events
const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => { ... }

// Form submit
const handleSubmit = (e: React.FormEvent<HTMLFormElement>): void => { ... }

// Keyboard events
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => { ... }
```

### H. Type useState, useRef, useContext

```ts
// useState
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);

// useRef
const inputRef = useRef<HTMLInputElement>(null);
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

// useContext
const ctx = useContext<UserContextType>(UserContext);
```

### I. Type API responses and async functions

```ts
// BEFORE
async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

// AFTER
interface User {
  id: number;
  name: string;
  email: string;
}

async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<User>;
}
```

---

## Step 7 — Handle Common Type Errors

### "Property does not exist on type"

```ts
// Option 1: Add the property to the interface
interface Config {
  timeout: number;
  retries?: number;  // was missing
}

// Option 2: Use type assertion (only when certain)
(config as ExtendedConfig).retries

// Option 3: Index signature (for dynamic keys)
interface Config {
  [key: string]: unknown;
}
```

### "Object is possibly null or undefined"

```ts
// Option 1: Optional chaining
const name = user?.profile?.name;

// Option 2: Non-null assertion (only when you are certain it's not null)
const name = user!.profile.name;

// Option 3: Type guard
if (user && user.profile) {
  const name = user.profile.name;
}
```

### "Argument of type X is not assignable to parameter of type Y"

```ts
// Narrow the type with a guard
function isString(val: unknown): val is string {
  return typeof val === 'string';
}

// OR use explicit cast when you know the type
const id = (event.target as HTMLInputElement).value;
```

### "Cannot find module" / "Could not find declaration file"

```bash
# Check if @types package exists
npm info @types/<package> version 2>/dev/null

# If no @types package, create a local declaration:
# src/@types/<package>.d.ts
declare module '<package>' {
  const value: any;
  export default value;
}
```

### Typed `Object.keys` / `Object.entries`

```ts
// Object.keys loses the key type — use this pattern:
const keys = Object.keys(obj) as Array<keyof typeof obj>;

// OR create a typed helper
function typedKeys<T extends object>(obj: T): Array<keyof T> {
  return Object.keys(obj) as Array<keyof T>;
}
```

---

## Step 8 — Create Shared Type Definitions

Organize shared types in a `src/types/` directory:

```
src/
  types/
    index.ts          ← re-exports all types
    api.ts            ← API response shapes
    models.ts         ← domain model types (User, Product, etc.)
    components.ts     ← shared component prop types
    env.d.ts          ← environment variable types
    global.d.ts       ← module augmentations, global declarations
```

### `src/types/env.d.ts` (for Vite/webpack env vars)

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
  // add more as needed
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### `src/types/global.d.ts` (for non-module assets)

```ts
declare module '*.svg' {
  import * as React from 'react';
  export const ReactComponent: React.FunctionComponent<React.SVGProps<SVGSVGElement>>;
  const src: string;
  export default src;
}

declare module '*.png' { const content: string; export default content; }
declare module '*.jpg' { const content: string; export default content; }
declare module '*.css' { const content: Record<string, string>; export default content; }
declare module '*.json' { const content: unknown; export default content; }
```

---

## Step 9 — Update ESLint Configuration

If `.eslintrc*` or `eslint.config.js` exists:

```bash
pnpm add -D @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

Update `.eslintrc.json` (or equivalent):

```json
{
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "project": "./tsconfig.json",
    "ecmaVersion": 2020,
    "sourceType": "module",
    "ecmaFeatures": { "jsx": true }
  },
  "plugins": ["@typescript-eslint"],
  "extends": [
    "plugin:@typescript-eslint/recommended"
  ],
  "rules": {
    "@typescript-eslint/no-explicit-any": "warn",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/explicit-function-return-type": "off",
    "@typescript-eslint/explicit-module-boundary-types": "off"
  },
  "overrides": [
    {
      "files": ["*.js", "*.jsx"],
      "rules": {
        "@typescript-eslint/no-var-requires": "off"
      }
    }
  ]
}
```

---

## Step 10 — Run Type Checking

```bash
# Type-check without emitting files
npx tsc --noEmit

# Type-check and show error count only
npx tsc --noEmit 2>&1 | tail -5
```

Triage the errors by category:
1. **Missing types** (parameters, return types) → add type annotations
2. **Null/undefined issues** → apply optional chaining or guards
3. **Module not found** → add `@types/*` or create `.d.ts` declarations
4. **Type mismatches** → refine interfaces or add union types

Fix errors in batches, starting with errors in shared utilities (fixing them often resolves downstream errors automatically).

---

## Step 11 — Update `package.json` Scripts

Add a type-check script if it doesn't exist:

```json
{
  "scripts": {
    "type-check": "tsc --noEmit",
    "type-check:watch": "tsc --noEmit --watch"
  }
}
```

If CI exists (`.github/workflows/*.yml`), verify that `type-check` is called or that the build step covers it.

---

## Step 12 — Migrate Tests

For each test file (`*.test.js` / `*.spec.js`):

1. Rename to `.test.ts` / `.test.tsx`
2. Add types to `describe` / `it` callbacks and mock functions:

```ts
// BEFORE
const mockFn = jest.fn();
mockFn.mockReturnValue('hello');

// AFTER
const mockFn = jest.fn<string, []>();
mockFn.mockReturnValue('hello');
```

3. Type `jest.spyOn` results:

```ts
const spy = jest.spyOn(obj, 'method') as jest.SpyInstance<ReturnType<typeof obj.method>>;
```

4. Use typed render helpers (React Testing Library):

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MyComponent } from './MyComponent';

test('renders correctly', () => {
  render(<MyComponent label="Test" onClick={jest.fn()} />);
  expect(screen.getByText('Test')).toBeInTheDocument();
});
```

---

## Step 13 — Verify Everything Works

Run all of these, in order:

```bash
# 1. Type check
npx tsc --noEmit 2>&1 | tail -10

# 2. Lint
pnpm lint 2>&1 | tail -10

# 3. Tests
pnpm test --passWithNoTests 2>&1 | tail -20

# 4. Build
pnpm build 2>&1 | tail -20
```

If any step fails, investigate the errors before declaring the migration complete.

### Scoped Migration: Extra Verification

After a scoped pass, run these additional checks to confirm the JS/TS boundary is healthy:

```bash
# Confirm no new TS errors were introduced in files OUTSIDE the migrated scope
npx tsc --noEmit 2>&1 | grep -v "^src/<migrated-path>" | grep "error TS" | wc -l
# Should be 0 (same count as before the migration)

# Confirm inbound JS consumers still resolve correctly
# Run the specific tests for the consuming JS modules, not just the migrated ones
pnpm test --testPathPattern="<consumer-feature>" 2>&1 | tail -10

# Confirm the feature's own tests pass
pnpm test --testPathPattern="<migrated-path>" 2>&1 | tail -10
```

Also verify that any `.d.ts` stubs you created are accurate by checking them against the actual JS files they describe:

```bash
# For each stub at src/utils/legacyHelper.d.ts, diff the exported names
grep "^export" src/utils/legacyHelper.d.ts
grep "^export\|module.exports" src/utils/legacyHelper.js
```

---

## Step 14 — Cleanup

Once all files are migrated and all checks pass:

> **Scoped migration:** Only perform cleanup steps that apply to the migrated scope. Do **not** remove `prop-types`, `flow-bin`, or `allowJs` until the entire repo is migrated.

### Remove PropTypes (if fully migrated)

```bash
pnpm remove prop-types
pnpm remove -D @types/prop-types
```

### Remove Flow tooling (if it was in use)

```bash
pnpm remove -D flow-bin flow-typed @babel/preset-flow
```

### Remove Flow type annotations from any stragglers

```bash
# Find remaining @flow comments
grep -rl "@flow" src/ 2>/dev/null
```

### Remove Babel's Flow preset from `babel.config.js` if present:

```js
// Remove: '@babel/preset-flow'
```

---

## Step 15 — Final Report

Provide a summary with these sections:

### ✅ Migrated Files
List each file renamed from `.js`/`.jsx` → `.ts`/`.tsx`.

### 🔧 Configuration Changes
List all config files modified (`tsconfig.json`, `babel.config.js`, `jest.config.js`, `.eslintrc`, `package.json`).

### 📦 Packages Added / Removed
List all `@types/*` packages added and `prop-types`/`flow-bin` packages removed.

### ⚠️ Remaining `any` Types
List every location where `any` was used as a placeholder. These should be tracked as tech debt and refined in follow-up work.

### 🔴 Files Skipped / Blocked
List any files that couldn't be fully migrated and why (e.g., complex dynamic types, third-party module without types).

### 🗺️ Migration Map (Scoped mode only)
If this was a scoped migration, include a status table showing progress across the full repo:

```
Feature / Directory              Status        Notes
-------------------------------  ------------  ---------------------------------
src/features/auth/**             ✅ Migrated   This pass
src/components/Button.*          ✅ Migrated   This pass
src/utils/legacyHelper.js        🔲 Stub only  .d.ts created, migrate next
src/features/payments/**         ⬜ Pending    Recommended next target
src/pages/**                     ⬜ Pending    Depends on payments migration
```

Update `src/types/migration-status.d.ts` to reflect the current state.

### 📋 Recommended Follow-up
- Enable `noUnusedLocals: true` and `noUnusedParameters: true` in `tsconfig.json` once the team is comfortable
- Replace `any` usages with proper types iteratively
- Add `type-check` to the CI pipeline if not already present
- Consider enabling `exactOptionalPropertyTypes: true` for stricter null safety
- **(Scoped)** Suggest the next logical feature/directory to migrate based on the dependency map

---

## Important Notes

- **Never remove `allowJs: true` until every `.js` file is migrated** — it allows mixed JS/TS during incremental migration.
- **Prefer `unknown` over `any`** — `unknown` forces you to narrow the type before use, making code safer.
- **Do not use `// @ts-ignore`** unless absolutely necessary. Prefer `// @ts-expect-error` with a comment explaining why.
- **`React.FC` is discouraged** for modern React — use explicit `(props: Props) => React.ReactElement` signatures instead.
- **`defaultProps` is deprecated** for function components — use default parameter values in destructuring.
- **Index signatures `[key: string]: any`** defeat the purpose of TypeScript. Only use them for truly dynamic objects, and prefer `Record<string, ValueType>` or `Map<string, ValueType>`.
- **Type imports** should use `import type { Foo } from './foo'` when the import is only used as a type — this is erased at compile time and avoids circular dependency issues.
