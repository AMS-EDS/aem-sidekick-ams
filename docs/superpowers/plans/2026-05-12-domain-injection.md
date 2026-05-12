# Domain Injection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `rollup.config.js` to read domain env vars at build time and inject the customer-specific domain prefix into the built extension, replacing the hard-coded `gov-aem` strings throughout.

**Architecture:** The existing `@rollup/plugin-replace` call in `commonPlugins()` is extended with one new key-value pair: `'gov-aem' → domainPrefix`. The prefix is derived from `HLX_PROD_SERVER_HOST_PAGE` (strip `.page`). A fail-fast guard at the top of the config throws a clear error if the env vars are not set before building.

**Tech Stack:** Rollup, `@rollup/plugin-replace` (already a dependency), Node.js `process.env`

---

## File Map

| Action | File | Change |
|---|---|---|
| Modify | `rollup.config.js` | Add env var reading + guard at top; add `'gov-aem'` key to existing `replace()` call |
| Verify (no edit) | `src/extension/utils/admin.js` | Confirm `gov-aem` is replaced in dist output |
| Verify (no edit) | `src/extension/auth.js` | Confirm `gov-aem` is replaced in dist output |
| Verify (no edit) | `src/extension/app/utils/browser.js` | Confirm `gov-aem` is replaced in dist output |
| Verify (no edit) | `src/extension/app/store/app.js` | Confirm `gov-aem` is replaced in dist output |
| Verify (no edit) | `src/extension/app/store/site.js` | Confirm `gov-aem` is replaced in dist output |
| Verify (no edit) | `src/extension/project.js` | Confirm `gov-aem` is replaced in dist output |

No source files are modified. No test files require changes (the test suite runs against source, not dist, and uses `gov-aem` as a valid domain example in test cases — this is intentional and correct).

---

### Task 1: Establish baseline

Confirm the existing tests pass and a plain `npm run build` succeeds before touching anything.

**Files:** none

- [ ] **Step 1: Run the test suite**

```bash
npm test
```

Expected: all tests pass. If any are already failing, note them — they are pre-existing and not caused by this change.

- [ ] **Step 2: Run a baseline build (env vars NOT set)**

In a shell where you have NOT sourced any `.env` file:

```bash
npm run build
```

Expected: build succeeds (producing `dist/chrome/`) but with `gov-aem` still hard-coded in the output. This is the "before" state.

- [ ] **Step 3: Confirm gov-aem is present in the unmodified build output**

```bash
grep -r "gov-aem" dist/chrome/ --include="*.js" -l
```

Expected: several files listed (confirms the domain is currently baked in as-is).

---

### Task 2: Edit rollup.config.js

Add the env var guard and the domain replacement.

**Files:**
- Modify: `rollup.config.js`

- [ ] **Step 1: Open `rollup.config.js` and locate the imports block (lines 1–22)**

The file begins with copyright comment, then imports. The last import is:

```js
import sidekickManifestBuildPlugin from './build/build.js';
```

- [ ] **Step 2: Add the env var block immediately after the last import**

Insert these lines after `import sidekickManifestBuildPlugin from './build/build.js';` and before `function shared(browser, path = '') {`:

```js
const hlxPage = process.env.HLX_PROD_SERVER_HOST_PAGE;
const hlxLive = process.env.HLX_PROD_SERVER_HOST_LIVE;

if (!hlxPage || !hlxLive) {
  throw new Error(
    '\nDomain env vars not set.\nRun: source environments/<env-name>.env  before building.\n',
  );
}

const domainPrefix = hlxPage.replace(/\.page$/, '');
```

- [ ] **Step 3: Locate `commonPlugins()` (around line 40) and find the `replace({...})` call**

It currently looks like:

```js
replace({
  'process.env.NODE_ENV': JSON.stringify('production'),
}),
```

- [ ] **Step 4: Replace that call with the extended version**

```js
replace({
  preventAssignment: true,
  'process.env.NODE_ENV': JSON.stringify('production'),
  'gov-aem': domainPrefix,
}),
```

`preventAssignment: true` is a safe addition recommended by the plugin — it prevents replacements on the left-hand side of assignments, avoiding accidental overwrites.

- [ ] **Step 5: Verify the full `commonPlugins()` function looks like this**

```js
function commonPlugins() {
  return [
    /** Resolve bare module imports */
    nodeResolve(),
    /** Transform decorators with babel */
    babel({ babelHelpers: 'bundled' }),
    /** Minify JS, compile JS to a lower language target */
    esbuild({
      minify: true,
      target: ['chrome64'],
    }),
    replace({
      preventAssignment: true,
      'process.env.NODE_ENV': JSON.stringify('production'),
      'gov-aem': domainPrefix,
    }),
  ];
}
```

---

### Task 3: Verify the fail-fast guard

Confirm the build errors clearly when env vars are absent.

**Files:** none

- [ ] **Step 1: Run the build WITHOUT sourcing an env file**

In a clean shell (or unset the vars explicitly):

```bash
unset HLX_PROD_SERVER_HOST_PAGE HLX_PROD_SERVER_HOST_LIVE
npm run build
```

Expected: build exits with a non-zero code and prints:

```
Domain env vars not set.
Run: source environments/<env-name>.env  before building.
```

If you see this error, the guard works. Proceed.

---

### Task 4: Verify the domain is injected correctly

Build with a real env file and confirm `gov-aem` is gone from the output.

**Files:** none

- [ ] **Step 1: Source an env file and build**

```bash
source ../ams-eds-terraform/environments/ent-aem.env && npm run build
```

(Adjust the relative path if your terminal is not in the `aem-sidekick-ams` directory. The file is at `ams-eds-terraform/environments/ent-aem.env` relative to the `eds_tools` directory.)

Expected: build completes with no errors and `dist/chrome/` is populated.

- [ ] **Step 2: Confirm the target domain appears in the output**

```bash
grep -r "ent-aem" dist/chrome/ --include="*.js" -l
```

Expected: several files listed.

- [ ] **Step 3: Confirm gov-aem is NOT present in the output**

```bash
grep -r "gov-aem" dist/chrome/ --include="*.js"
```

Expected: no output (empty result). If any lines appear, check which file and which source location they came from — a match here means a `gov-aem` string was missed by the replace.

- [ ] **Step 4: Spot-check the ADMIN_ORIGIN value**

```bash
grep "admin\." dist/chrome/utils/admin.js
```

Expected output includes: `admin.ent-aem.page`

- [ ] **Step 5: Spot-check the isValidHost domain array**

```bash
grep "ent-aem" dist/chrome/project.js
```

Expected: one or more matches including the `isValidHost` allowlist entry.

- [ ] **Step 6: Spot-check auth.js CORS regex**

```bash
grep "ent-aem" dist/chrome/auth.js
```

Expected: matches for the CORS regex filter, initiator domains, and siteToken URL regex.

---

### Task 5: Run the test suite to confirm nothing regressed

The tests run against source files (not dist), so they are unaffected by the rollup change. This step confirms no accidental side-effects.

**Files:** none

- [ ] **Step 1: Run the full test suite**

```bash
npm test
```

Expected: same results as Task 1 Step 1 (all previously passing tests still pass).

---

### Task 6: Build with a second env file to confirm portability

Confirm the mechanism works for a different environment, not just `ent-aem`.

**Files:** none

- [ ] **Step 1: Source the gov-stage env and rebuild**

```bash
source ../ams-eds-terraform/environments/gov-stage.env && npm run build
```

- [ ] **Step 2: Confirm gov-stage domain in the output**

```bash
grep -r "gov-aem" dist/chrome/ --include="*.js" -l
```

Expected: several files listed — `gov-stage.env` sets `HLX_PROD_SERVER_HOST_PAGE=gov-aem.page`, so `domainPrefix` is `gov-aem` and the output correctly matches the original source. This proves the mechanism is wiring up env → build correctly (not just that we hardcoded the replacement).

- [ ] **Step 3: Confirm ent-aem does NOT appear in this build**

```bash
grep -r "ent-aem" dist/chrome/ --include="*.js"
```

Expected: no output.

---

### Task 7: Commit

- [ ] **Step 1: Stage the only modified file**

```bash
git add rollup.config.js
```

- [ ] **Step 2: Confirm only rollup.config.js is staged**

```bash
git diff --cached --name-only
```

Expected:
```
rollup.config.js
```

- [ ] **Step 3: Commit**

```bash
git commit -m "feat: inject customer domain via env vars at build time"
```

---

## Build Reference

For daily use, after this change:

```bash
# Build for ent-aem
source environments/ent-aem.env && npm run build

# Build for gov-stage
source environments/gov-stage.env && npm run build

# Dev/watch build (also requires env sourced)
source environments/ent-aem.env && npm run build:dev
source environments/ent-aem.env && npm run build:watch
```

The env file path assumes you're in the `aem-sidekick-ams` directory. Adjust if needed.
