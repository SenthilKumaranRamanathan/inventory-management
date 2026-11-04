---
name: saas-ui-redesign
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a left vertical sidebar navigation, consistent spacing, and a polished professional look. Use this skill when asked to restyle, modernize, or re-layout a Vue 3 app's shell, navigation, or global styles.
---

# SaaS-Style UI Redesign for Vue 3

Use this skill to turn a basic Vue 3 layout (typically a top navigation bar over a page container) into a modern SaaS-style interface: a vertical sidebar on the left, a clean content column, a consistent spacing and color system, and a professional finish.

This skill changes presentation only. Do not change backend code, API calls, data logic, or routes. Keep every existing page and navigation target working.

## Step 1: Audit before changing anything

Read first. Make no edits until the audit is done, then summarize what you found.

1. Find the root layout component (usually `App.vue`) and the router file (usually `main.js` or `router/index.js`). List every route path and its nav label.
2. Find where navigation labels come from. If the app uses i18n, reuse the existing keys instead of hardcoding text. Note any label that is hardcoded and not translated.
3. Find the global styles. Note hardcoded colors, fonts, container widths, and whether CSS variables already exist.
4. List shared class names that pages reuse (page headers, cards, stat tiles, tables, badges). These are your normalization targets.
5. Find components coupled to the old header. Look for:
   - `position: sticky` or `fixed` with a hardcoded `top` value equal to the old header height
   - dropdowns that open downward and right-aligned (`top: calc(100% + ...)`, `right: 0`)
   - selectors that depend on the old nav container structure
6. Read the project's own instructions (CLAUDE.md, README, design notes) for palette, fonts, icon rules, and conventions. Project rules override the defaults in this skill.

## Step 2: Define design tokens

Put tokens in one place, on `:root` in the global stylesheet. Replace hardcoded values with these variables as you touch styles.

```css
:root {
  /* Color: reuse the project's palette if it has one */
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-accent: #2563eb;
  --color-accent-soft: #eff6ff;
  --color-success: #059669;
  --color-warning: #d97706;
  --color-danger: #dc2626;

  /* Spacing: 4px base scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* Shape and depth */
  --radius-sm: 6px;
  --radius-md: 10px;
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);

  /* Layout */
  --sidebar-width: 15rem;
  --content-max-width: 1400px;
}
```

Rules:
- Use only values from the spacing scale for padding, margin, and gap. Avoid one-off values like 13px or 0.7rem.
- Keep one font family. Use the font the project already uses.
- Keep at most two font weights for body UI (regular and semibold) plus bold for page titles.

## Step 3: Build the app shell

Target structure: a fixed sidebar on the left and a main column on the right.

```vue
<template>
  <div class="app-shell">
    <aside class="sidebar" aria-label="Primary">
      <div class="sidebar-brand">{{ t('nav.companyName') }}</div>
      <nav class="sidebar-nav">
        <router-link v-for="item in navItems" :key="item.path" :to="item.path" class="sidebar-link">
          {{ item.label }}
        </router-link>
      </nav>
      <div class="sidebar-footer">
        <!-- utility controls such as profile menu and language switcher -->
      </div>
    </aside>

    <div class="main-column">
      <!-- optional toolbar: filters and page actions -->
      <main class="content">
        <router-view />
      </main>
    </div>
  </div>
</template>
```

```css
.app-shell { display: grid; grid-template-columns: var(--sidebar-width) 1fr; min-height: 100vh; background: var(--color-bg); }
.sidebar { position: sticky; top: 0; height: 100vh; display: flex; flex-direction: column; background: var(--color-surface); border-right: 1px solid var(--color-border); padding: var(--space-4); }
.sidebar-nav { display: flex; flex-direction: column; gap: var(--space-1); margin-top: var(--space-6); flex: 1; overflow-y: auto; }
.sidebar-link { padding: var(--space-2) var(--space-3); border-radius: var(--radius-sm); color: var(--color-text-muted); text-decoration: none; font-weight: 500; }
.sidebar-link:hover { background: var(--color-bg); color: var(--color-text); }
.sidebar-link.router-link-exact-active { background: var(--color-accent-soft); color: var(--color-accent); }
.main-column { min-width: 0; }
.content { max-width: var(--content-max-width); margin: 0 auto; padding: var(--space-6) var(--space-8); }
```

Guidelines:
- Build the nav from a data array (`navItems`) with unique keys, not repeated hardcoded links. Take labels from the existing i18n keys.
- Use `router-link-exact-active` or the router's active class for the current page, and set `aria-current="page"` on it.
- Put the brand at the top, navigation in the middle, and utility controls (profile, language) at the bottom of the sidebar.
- Use text labels or inline SVG icons. Do not use emoji as icons unless the project explicitly allows it.
- Add `min-width: 0` to the main column so wide tables and charts scroll inside their own container instead of stretching the page.

## Step 4: Fix layout-coupled components

These break when the header goes away. Fix each one found in the audit.

- Sticky bars (for example a filter bar): move inside the main column and set `top: 0`. Remove any offset that equaled the old header height.
- Dropdowns anchored below a trigger: when the trigger is at the bottom of the sidebar, open the dropdown upward (`bottom: calc(100% + var(--space-2)); top: auto`) and align to the left (`left: 0; right: auto`). Check that it is not clipped by `overflow` on the sidebar.
- Selectors such as `.nav-container > .some-component`: rewrite them for the new structure. Search for the old class names to catch leftovers, then delete unused CSS.

## Step 5: Normalize shared components

Restyle the shared classes found in the audit using tokens, keeping the class names unchanged so individual pages need no edits.

- Page header: consistent title size, muted subtitle, `var(--space-6)` below.
- Cards and stat tiles: `var(--color-surface)` background, `1px solid var(--color-border)`, `var(--radius-md)`, `var(--shadow-sm)`, `var(--space-4)` or `var(--space-6)` padding.
- Tables: uppercase small muted header cells, row hover using `var(--color-bg)`, consistent cell padding.
- Badges: one shape, with color driven by status (success, warning, danger, info).
- Charts: keep the existing chart technology (for example custom SVG). Only update colors and spacing.

## Step 6: Responsive behavior

Work mobile-first. Below roughly 768px:

- Hide the sidebar off-canvas and show a menu button in a slim top bar.
- Toggle the sidebar as a drawer with a `ref` for open state. Close it after navigation and on Escape.
- Switch the shell grid to a single column.
- Wrap wide tables in a container with `overflow-x: auto`.
- Wrap transitions in `@media (prefers-reduced-motion: no-preference)`.

## Quality checklist

Before finishing, confirm each item:

- Every existing route is reachable from the sidebar, and the active page is highlighted.
- All colors, spacing, radii and shadows come from tokens. Search for leftover hardcoded hex values in the files you touched.
- Interactive elements have visible keyboard focus (`:focus-visible`) and sufficient color contrast.
- No horizontal page scroll at desktop or mobile widths.
- Loading and error states on every page still display correctly.
- No emoji in the UI. No console errors.

## Process rules

- Follow the project's Vue conventions: Composition API, scoped component styles, unique `v-for` keys, and no prop mutation.
- If the project requires a specialist agent for `.vue` changes, delegate those edits to it.
- Work in small steps (tokens, shell, coupled components, shared styles, responsive) and check the running app after each one.
- Verify visually in a browser at desktop and mobile widths. If browser automation is unavailable, tell the user what still needs a manual check.
- Do not commit, push, or add dependencies unless the user asks. Prefer plain CSS and the existing stack over adding a UI library.
