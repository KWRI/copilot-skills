---
name: dependency-audit
description: Full dependency audit for a Node.js/npm/pnpm package. Use this skill when asked to audit dependencies, find unused packages, identify deprecated packages, find outdated packages, check for peer dependency issues, or clean up package.json.
---

# Dependency Audit Skill

Perform a thorough dependency audit of a Node.js project. Follow these steps in order.

## Step 1 — Detect the Package Manager

Check for lock files to determine the package manager:
- `pnpm-lock.yaml` → use `pnpm`
- `yarn.lock` → use `yarn`
- `package-lock.json` → use `npm`

All subsequent commands should use the detected package manager.

## Step 2 — Collect Raw Data (run in parallel)

Run all three of these simultaneously:

### A. Check for outdated packages (with deprecation info)

```bash
# For pnpm (preferred — includes isDeprecated field):
pnpm outdated --format json 2>/dev/null

# Fallback for npm/yarn:
npm outdated --json 2>/dev/null || yarn outdated --json 2>/dev/null
```

Parse the JSON output. For each package, note:
- `current` version
- `latest` version
- `isDeprecated: true` → flag as **deprecated**
- Major version gap (latest major > current major) → flag as **major update needed**
- Minor/patch gap only → flag as **safe to update**

### B. Check for unused packages

```bash
npx depcheck --ignore-bin-package --skip-missing 2>/dev/null
```

Collect the lists of unused `dependencies` and `devDependencies`.

### C. Check for peer dependency issues

```bash
# pnpm:
pnpm install --dry-run 2>&1 | grep -i "peer\|unmet\|warn" | head -50

# npm:
npm install --dry-run 2>&1 | grep -i "peer\|unmet\|warn" | head -50
```

Also run:
```bash
npx check-peer-dependencies 2>/dev/null || true
```

## Step 3 — Verify Depcheck Results (CRITICAL — avoid false positives)

Depcheck often misses config-file usage. For every package flagged as unused, verify before removing:

### Config files to check

Read these files (if they exist) to cross-reference flagged packages:

| File | Packages it may reference |
|------|--------------------------|
| `babel.config.js` / `.babelrc` | `@babel/preset-*`, `babel-plugin-*`, `babel-preset-*` |
| `.eslintrc` / `.eslintrc.json` / `.eslintrc.js` / `eslint.config.js` | `eslint-config-*`, `eslint-plugin-*` |
| `commitlint.config.js` | `@commitlint/config-*`, `commitlint-config-*` |
| `jest.config.js` / `jest.setup.js` / `setupTests.js` | `jest-*`, test utilities |
| `webpack.config.js` / `rspack.*.js` / `vite.config.js` | loaders, plugins |
| `postcss.config.js` | `postcss-*` plugins |
| `tsconfig.json` | `@types/*` packages |
| `.stylelintrc` | `stylelint-*` packages |

### Peer dependency chains

ESLint config packages (e.g. `eslint-config-next`, `eslint-config-airbnb`) declare peer deps on plugins. Even if the plugins aren't directly in `.eslintrc`, they're **required** if an extended config needs them. Check the peer deps of any extended eslint config before removing its plugins.

### Direct usage scan

For any remaining flagged package, search the source:
```bash
grep -r "from '[PACKAGE]'\|require('[PACKAGE]'" --include="*.js" --include="*.jsx" --include="*.ts" --include="*.tsx" src/ __tests__/ 2>/dev/null | head -5
```

If no results in source AND not found in any config file AND not a required peer dep → it is genuinely **unused**.

## Step 4 — Categorize All Findings

Build four clear lists before taking any action:

1. **Remove — Confirmed Unused**: Packages with no config, source, or peer dep references.
2. **⚠️ Deprecated**: Packages flagged `isDeprecated: true` by the registry. Note the recommended replacement if known.
3. **⬆️ Safe Updates** (minor/patch within same major): These can be updated with low risk.
4. **🔴 Major Version Updates Required**: These have breaking changes and need manual migration — list them but do NOT auto-update.

Present this categorization to the user before making changes, unless the user asked you to proceed automatically.

## Step 5 — Remove Unused Packages

For each confirmed unused package, remove it using the package manager:

```bash
# pnpm (handles deps and devDeps automatically):
pnpm remove <package1> <package2> ...

# npm:
npm uninstall <package1> <package2> ...

# yarn:
yarn remove <package1> <package2> ...
```

Remove runtime deps and devDeps in separate commands if you encounter errors (some package managers require `--save-dev` for devDeps).

### Also clean up `resolutions` / `overrides`

After removing packages, check if `package.json` has a `resolutions` or `overrides` block that pins a removed package. Remove those entries manually using the edit tool.

## Step 6 — Apply Safe Minor/Patch Updates

For packages where only a minor or patch update is available (same major version), update the version constraint in `package.json`:

- If the current constraint is a range like `^1.2.3` and latest is `1.5.0`, the range already covers it — run `pnpm update <package>` instead.
- If the version is pinned (no `^` or `~`), update the pin in `package.json` to the latest version within the same major.

Then reinstall:
```bash
pnpm install  # or npm install / yarn install
```

## Step 7 — Verify

Run the project's existing lint and/or test commands to confirm nothing is broken:

```bash
# Try these in order, use whichever exists:
pnpm lint 2>&1 | tail -5
pnpm test:ci 2>&1 | tail -20
npm run lint 2>&1 | tail -5
```

If lint or tests fail after your changes, investigate and revert only the specific package removal(s) that caused the failure. Use `git diff package.json` to review all changes made.

## Step 8 — Report

Provide a final summary with four sections:

### ✅ Removed (Unused)
List every removed package and the reason (e.g. "No imports found in source or config files").

### ⚠️ Deprecated (noted, not removed if still in use)
For each deprecated package still present in `package.json`:
- Current version
- Why it's deprecated (if known)
- Recommended replacement (if known)

### ⬆️ Updated
List packages that were updated and their old → new versions.

### 🔴 Major Updates — Manual Migration Required
List packages with major version gaps. Include:
- Current version → Latest version
- Any known breaking changes or migration guides (look these up if useful)

## Important Notes

- **Never remove** a package just because depcheck flagged it. Always verify against config files first.
- **Never auto-update** a major version — major bumps almost always have breaking API changes.
- **`moment`** is widely considered legacy (in favor of `date-fns` or `dayjs`), but if the project is actively using it, only flag it — don't remove it.
- **`enzyme`** is not maintained for React 18+. If found alongside React 18, flag it as deprecated.
- **`redux-devtools-extension`** is deprecated — the functionality is now built into Redux DevTools browser extension directly and `@redux-devtools/extension` is the maintained fork.
- **`babel-eslint`** is deprecated — the replacement is `@babel/eslint-parser`.
- **`next-plugin-transpile-modules`** is deprecated — the functionality was merged into Next.js core.
- **`react-google-maps`** (the old `^9.x` version) is deprecated — use `@react-google-maps/api` instead.
