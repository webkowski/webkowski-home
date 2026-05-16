# Upgrade Plan: Astro 6.3 + Tailwind 4.3

## Overview

This document outlines all steps required to upgrade the project from the current stack to Astro 6.3 and Tailwind CSS 4.3, including all dependency and codebase changes.

**Current versions:**
| Package | Current | Target |
|---|---|---|
| `astro` | `^2.10.15` | `^6.3.x` |
| `tailwindcss` | `^3.0.24` | `^4.3.x` |
| `@astrojs/tailwind` | `^4.0.0` | **REMOVE** (replaced by `@tailwindcss/vite`) |
| `@astrojs/ts-plugin` | `^1.1.3` | **REMOVE** (bundled with Astro 4+) |
| `@tailwindcss/vite` | — | **ADD** (new Tailwind v4 Vite plugin) |

---

## Step 1 — Update `package.json` dependencies

### Remove:
- `@astrojs/tailwind` — deprecated, incompatible with Tailwind v4; replaced by the official Vite plugin
- `@astrojs/ts-plugin` — the TypeScript language server plugin has been bundled directly into Astro since v4, the standalone package is no longer needed

### Add:
- `@tailwindcss/vite` — the official Tailwind CSS v4 Vite plugin

### Updated `package.json` dependencies block:
```json
{
  "dependencies": {
    "astro": "^6.3.0",
    "tailwindcss": "^4.3.0",
    "@tailwindcss/vite": "^4.3.0"
  }
}
```

> **Note:** Verify the exact latest stable versions of `astro` and `@tailwindcss/vite` on npm before installing, as patch versions may differ.

> **Node.js requirement:** Astro 5+ requires Node.js `>=18.17.1` or `>=20.3.0`. Astro 6 likely requires Node.js 20+. Verify your runtime environment before upgrading.

---

## Step 2 — Update `astro.config.mjs`

**Current:** Uses the `@astrojs/tailwind` integration.

**Required change:** Replace the integration with the `@tailwindcss/vite` Vite plugin.

**Current file:**
```js
import { defineConfig } from 'astro/config';
import tailwind from '@astrojs/tailwind';

export default defineConfig({
  integrations: [tailwind()]
});
```

**New file:**
```js
import { defineConfig } from 'astro/config';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  vite: {
    plugins: [tailwindcss()]
  }
});
```

---

## Step 3 — Replace `tailwind.config.cjs` with CSS-based configuration

Tailwind v4 eliminates the JavaScript config file. All configuration (theme, content, plugins) moves into a CSS file using the `@import` and `@theme` directives.

### Action: Delete `tailwind.config.cjs`

The file is no longer used. Content scanning is now automatic by default in Tailwind v4.

### Action: Create `src/styles/global.css`

```css
@import "tailwindcss";

@theme {
  --font-family-sans: "Karla", sans-serif;
}
```

This replaces the `theme.fontFamily.sans` configuration from `tailwind.config.cjs`.

---

## Step 4 — Import the global CSS in `src/layouts/Layout.astro`

With Tailwind v4, the CSS must be explicitly imported. Add the import to the layout component's frontmatter.

**Add to `src/layouts/Layout.astro` frontmatter:**
```astro
---
import '../styles/global.css';
// ...existing code
---
```

---

## Step 5 — Fix internal Astro import in `src/layouts/Layout.astro`

**File:** `src/layouts/Layout.astro`, line 2

**Current:**
```ts
import {HTMLString} from "astro/dist/runtime/server";
```

**Problem:** This imports from an internal Astro distribution path (`astro/dist/...`). These internal paths are not part of the public API and have been removed/restructured across Astro versions. This will cause a build error in Astro 6.

**Fix:** Replace with the standard `string` type. The `Props` interface should use `string` for the `title` prop:

```ts
export interface Props {
  title: string;
}
```

The `HTMLString` type was an internal Astro type used to mark sanitized HTML. For a plain string title prop, `string` is the correct type.

---

## Step 6 — Remove `@astrojs/ts-plugin` from `tsconfig.json`

**File:** `tsconfig.json`

**Current:**
```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@astrojs/ts-plugin"
      }
    ]
  }
}
```

Since `@astrojs/ts-plugin` is bundled with Astro 4+, the package no longer needs to be installed separately, and the plugin entry in `tsconfig.json` can be removed (or left — it will be a no-op if the package is not installed, but removing it is cleaner).

**New file:**
```json
{
  "extends": "astro/tsconfigs/strict"
}
```

---

## Step 7 — Audit Tailwind utility classes for v4 renames

Tailwind v4 renamed several utility classes. The following classes used in the project should be audited:

### Classes that changed in Tailwind v4 (check and update if present):
| Old class (v3) | New class (v4) | Files to check |
|---|---|---|
| `shadow-sm` | `shadow-xs` | All `.astro` files |
| `shadow` | `shadow-sm` | All `.astro` files |
| `blur-sm` | `blur-xs` | All `.astro` files |
| `rounded-sm` | `rounded-xs` | All `.astro` files |
| `ring` (default 3px) | `ring` (default 1px) | All `.astro` files |

**Result after review:** The current pages (`index.astro`, `projects.astro`, `Layout.astro`) do not use the renamed utilities above. No class renames are required for the current codebase.

### Non-standard / invalid classes already present (not caused by migration, pre-existing):
These classes produce no styles in v3 or v4 — they are silently ignored. They are noted here for awareness but are not a migration blocker:
- `text-4sm` — (`Layout.astro:50`, `index.astro:161`, `projects.astro:150`) — not a Tailwind utility
- `text-text-2` — (multiple places in `index.astro`, `projects.astro`) — not a Tailwind utility
- `justify-left` — (`Layout.astro:50`) — not a Tailwind utility (likely intended as `justify-start`)

---

## Summary of all file changes

| File | Action | Reason |
|---|---|---|
| `package.json` | Update versions, remove 2 packages, add 1 | Dependency upgrade |
| `astro.config.mjs` | Replace integration with Vite plugin | Tailwind v4 integration change |
| `tailwind.config.cjs` | **Delete** | Tailwind v4 uses CSS-based config |
| `src/styles/global.css` | **Create new file** | Tailwind v4 entry point + theme config |
| `src/layouts/Layout.astro` | Add CSS import + fix `HTMLString` import | CSS must be explicitly imported; internal Astro path removed |
| `tsconfig.json` | Remove `@astrojs/ts-plugin` plugin entry | Plugin bundled into Astro 4+, package removed |

---

## Recommended upgrade sequence

1. Create a new branch (e.g., `upgrade/astro6-tailwind4`)
2. Update `package.json` (Step 1) and run `yarn install` / `npm install`
3. Update `astro.config.mjs` (Step 2)
4. Delete `tailwind.config.cjs` (Step 3)
5. Create `src/styles/global.css` (Step 3)
6. Update `src/layouts/Layout.astro` (Steps 4 and 5)
7. Update `tsconfig.json` (Step 6)
8. Run `yarn dev` / `npm run dev` and verify the site renders correctly
9. Run `yarn build` / `npm run build` and verify no build errors
10. Visual QA: confirm font (Karla) and all Tailwind styles render as expected
11. Merge to `development`, then `master`

---

## Risk notes

- **Jumping 4 major Astro versions (2 → 6)** is a significant leap. Check the official Astro migration guides for each major version between your start and end versions, as there may be additional breaking changes not listed here that affect features added in the future.
- The current project is small (2 pages, 1 layout, no content collections, no server-side rendering, no islands), which significantly reduces upgrade risk.
- **Verify exact stable versions** of `astro` (6.3.x) and `@tailwindcss/vite` on npm before running the install, as patch versions available at the time of this writing may differ.
