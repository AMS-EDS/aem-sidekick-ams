# Domain Injection via Build-time Replacement

**Date:** 2026-05-12  
**Status:** Approved

## Problem

Domain strings (`gov-aem`, `gov-aem.page`, `gov-aem.live`, etc.) are hard-coded in six source files. The extension must be distributable to customers whose domains are confidential and cannot be committed to the repository. Building a separate copy of the extension for each customer requires a clean, repeatable way to inject the correct domain at build time.

## Solution

Use the existing `@rollup/plugin-replace` plugin (already in the build pipeline) to replace the literal string `gov-aem` with the target domain prefix, derived from environment variables that are already sourced before other tool usage. No source files are modified; only `rollup.config.js` changes.

## Affected Files (source — not modified)

| File | What contains `gov-aem` |
|---|---|
| `src/extension/utils/admin.js:26` | `ADMIN_ORIGIN` constant — the Admin API endpoint |
| `src/extension/auth.js:65,90,112` | CORS regex filters, initiator domains, siteToken URL regex |
| `src/extension/app/utils/browser.js:94-96,247-252` | `previewSuffixes`/`liveSuffixes` arrays, `isErrorPage` host checks |
| `src/extension/app/store/app.js:1253` | `liveDomains` array in `switchEnv` |
| `src/extension/app/store/site.js:257,261` | Fallback `scriptUrl`, domain ternary in site store init |
| `src/extension/project.js:406` | `isValidHost` domain allowlist |

## Environment Variables

All customer `.env` files in `ams-eds-terraform/environments/` export:

```sh
export HLX_PROD_SERVER_HOST_PAGE="ent-aem.page"   # preview TLD
export HLX_PROD_SERVER_HOST_LIVE="ent-aem.live"   # live TLD
```

The domain prefix (`ent-aem`) is derived by stripping `.page` from `HLX_PROD_SERVER_HOST_PAGE`.

## rollup.config.js Change

Two additions only — nothing else in the file changes.

**At the top of the file (after imports):**

```js
const hlxPage = process.env.HLX_PROD_SERVER_HOST_PAGE;
const hlxLive = process.env.HLX_PROD_SERVER_HOST_LIVE;

if (!hlxPage || !hlxLive) {
  throw new Error(
    '\nDomain env vars not set.\nRun: source environments/<env-name>.env  before building.\n'
  );
}

const domainPrefix = hlxPage.replace(/\.page$/, '');
```

**Inside `commonPlugins()`, extend the existing `replace({...})` call:**

```js
replace({
  preventAssignment: true,
  'process.env.NODE_ENV': JSON.stringify('production'),
  'gov-aem': domainPrefix,  // ← added
}),
```

`preventAssignment: true` is a best-practice flag that prevents accidental replacements on the left-hand side of assignments and is safe to add to the existing call.

## The `site.js:261` Ternary

```js
const domain = previewHost?.endsWith('.aem.page') ? 'aem' : 'gov-aem';
```

After replacement becomes:

```js
const domain = previewHost?.endsWith('.aem.page') ? 'aem' : 'ent-aem';
```

`.endsWith('.aem.page')` only matches the upstream `*.aem.page` domain. For `*.ent-aem.page` the character preceding `aem.page` is `-` (not `.`), so the condition is `false` and the ternary correctly returns the customer domain prefix. This holds for all AMS domain prefixes.

## SharePoint Fix

The previous attempt showed the sidekick on preview sites but not in SharePoint. Root cause: `ADMIN_ORIGIN` was still pointing to `admin.gov-aem.page`. The extension background service worker calls the Admin API to match a SharePoint URL to a stored project (via `mountpoints`/`contentSourceUrl`). With the wrong admin origin, that lookup silently fails and the sidekick does not activate in SharePoint. On preview sites the sidekick content script can load from the site's own `/tools/sidekick/config.json`, bypassing the admin API — which is why it appeared there.

This implementation replaces `ADMIN_ORIGIN` (along with all other occurrences) in a single pass, closing that gap.

## What Does NOT Change

- All six source files — untouched
- `manifest.json` — untouched (`host_permissions` is already `https://*/*`)
- `adobems-aem` references in suffix arrays and `isValidHost` — intentionally left in place; they add no matching URLs in customer deployments and are harmless
- No new npm scripts, no new dependencies

## Build Workflow

```sh
# Production build for a specific customer environment
source environments/ent-aem.env && npm run build

# Dev/watch build (inherits domain injection via rollup.config-dev.js → rollup.config.js)
source environments/ent-aem.env && npm run build:dev
source environments/ent-aem.env && npm run build:watch
```

If the env file has not been sourced, the build fails immediately with a clear error message rather than silently producing a broken extension.

## Out of Scope

- Replacing `adobems-aem` occurrences (not needed)
- Runtime domain configuration (no UI, no storage key)
- Multi-domain builds in a single pass (one build per customer)
