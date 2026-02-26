---
description: Redesign the Vue 3 app UI to a modern SaaS-style interface with a vertical sidebar
---

# SaaS Redesign — Vertical Sidebar Layout

Transform this Vue 3 application from a top navigation bar into a modern SaaS-style interface with a vertical sidebar. Work through the phases below in order.

---

## Phase 1 — Explore (read before changing anything)

Read these files to understand the current structure:
- `client/src/App.vue` — current layout, nav links, global styles
- `client/src/components/FilterBar.vue` — filter bar markup and styles

List all files in `client/src/components/` and note which ones are rendered inside `App.vue`. Identify:
1. The exact router-link paths and their label translations
2. Which components are in the sidebar area (LanguageSwitcher, ProfileMenu, etc.)
3. Which global CSS classes are used by view components and MUST be preserved

---

## Phase 2 — Target Layout

Replace the horizontal top nav with this structure:

```
┌──────────────────────────────────────────────────────────────┐
│  SIDEBAR (240px, fixed)  │  MAIN WRAPPER (flex: 1)           │
│ ─────────────────────── │ ─────────────────────────────────  │
│  [Logo]                  │  [FilterBar — sticky strip]        │
│  [App subtitle]          │ ─────────────────────────────────  │
│                          │                                    │
│  ── Nav ──               │  [<router-view />]                 │
│  • Overview              │                                    │
│  • Inventory             │                                    │
│  • Orders                │                                    │
│  • Finance               │                                    │
│  • Demand Forecast       │                                    │
│  • Reports               │                                    │
│  • Restocking            │                                    │
│                          │                                    │
│  ── (push to bottom) ──  │                                    │
│  [LanguageSwitcher]      │                                    │
│  [ProfileMenu]           │                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## Phase 3 — Design Tokens

Use these values everywhere. Do not invent new ones.

| Token | Value |
|---|---|
| Sidebar width | `240px` |
| Sidebar bg | `#ffffff` |
| Sidebar right border | `1px solid #e2e8f0` |
| Sidebar padding (x) | `0.875rem` |
| Content bg | `#f8fafc` |
| Nav item color (default) | `#64748b` |
| Nav item color (hover) | `#0f172a` |
| Nav item bg (hover) | `#f1f5f9` |
| Nav item color (active) | `#2563eb` |
| Nav item bg (active) | `#eff6ff` |
| Nav item active left border | `3px solid #2563eb` |
| Nav item left border (default) | `3px solid transparent` |
| Nav font size | `0.9rem` |
| Nav font weight | `500` |
| Nav item padding | `0.625rem 1rem` |
| Nav item border-radius | `6px` |
| Logo text size | `1.0625rem` |
| Logo font weight | `700` |
| Logo color | `#0f172a` |
| Subtitle font size | `0.75rem` |
| Subtitle color | `#94a3b8` |
| Section divider | `1px solid #f1f5f9` |
| Filter strip bg | `#ffffff` |
| Filter strip border | `1px solid #e2e8f0` |
| Main content padding | `1.5rem 2rem` |

---

## Phase 4 — Implementation

**MANDATORY RULE**: You MUST delegate all `.vue` file changes to the `vue-expert` subagent. Do not edit `.vue` files directly.

### What to delegate to vue-expert

Provide vue-expert with the full current content of `App.vue` and these instructions:

#### Template changes

Replace the entire `<template>` with:

```html
<template>
  <div class="app-shell">
    <!-- Vertical sidebar -->
    <aside class="sidebar">
      <div class="sidebar-header">
        <span class="sidebar-logo-text">{{ t('nav.companyName') }}</span>
        <span class="sidebar-subtitle">{{ t('nav.subtitle') }}</span>
      </div>

      <nav class="sidebar-nav">
        <!-- Replicate ALL existing router-links here as .nav-item elements -->
        <!-- Use :class="['nav-item', { active: $route.path === '/exact-path' }]" -->
        <!-- Keep ALL existing t() translation calls for link labels -->
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <!-- Right side: filter strip + content -->
    <div class="main-wrapper">
      <div class="filter-strip">
        <FilterBar />
      </div>
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- Keep ALL existing modals exactly as-is -->
    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />
    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>
```

Important: copy every router-link from the existing `<nav class="nav-tabs">` into `<nav class="sidebar-nav">`, converting each from:
```html
<router-link to="/path" :class="{ active: $route.path === '/path' }">Label</router-link>
```
to:
```html
<router-link to="/path" :class="['nav-item', { active: $route.path === '/path' }]">Label</router-link>
```

#### Script changes

Keep the entire `<script>` section **completely unchanged**. Zero modifications.

#### Style changes

Replace the entire `<style>` section with a new one that:

1. Keeps all global utility classes **word-for-word** (`.badge`, `.badge.*`, `.card`, `.card-header`, `.card-title`, `.stat-card`, `.stat-card.*`, `.stat-value`, `.stat-label`, `.stats-grid`, `.page-header`, `.table-container`, `table`, `thead`, `th`, `td`, `tbody tr`, `.loading`, `.error`). These are used by view components and must not change.

2. Replaces all layout classes (`.app`, `.top-nav`, `.nav-container`, `.logo`, `.subtitle`, `.nav-tabs`, `.nav-tabs a`, `.nav-tabs a:hover`, `.nav-tabs a.active`, `.nav-tabs a.active::after`, `.main-content`) with the new sidebar layout classes.

New layout CSS to add (using the Phase 3 design tokens):

```css
/* ── Layout shell ────────────────────────────────── */
.app-shell {
  display: flex;
  min-height: 100vh;
}

/* ── Sidebar ─────────────────────────────────────── */
.sidebar {
  width: 240px;
  min-width: 240px;
  background: #ffffff;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  height: 100vh;
  z-index: 100;
  overflow-y: auto;
}

.sidebar-header {
  padding: 1.25rem 0.875rem 1rem;
  border-bottom: 1px solid #f1f5f9;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.sidebar-logo-text {
  font-size: 1.0625rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.02em;
  line-height: 1.2;
}

.sidebar-subtitle {
  font-size: 0.75rem;
  color: #94a3b8;
  font-weight: 400;
}

.sidebar-nav {
  flex: 1;
  padding: 0.75rem 0.875rem;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.nav-item {
  display: block;
  padding: 0.625rem 1rem;
  color: #64748b;
  text-decoration: none;
  font-size: 0.9rem;
  font-weight: 500;
  border-radius: 6px;
  border-left: 3px solid transparent;
  transition: color 0.15s ease, background-color 0.15s ease;
  white-space: nowrap;
}

.nav-item:hover {
  color: #0f172a;
  background: #f1f5f9;
}

.nav-item.active {
  color: #2563eb;
  background: #eff6ff;
  border-left-color: #2563eb;
}

.sidebar-footer {
  padding: 0.875rem;
  border-top: 1px solid #f1f5f9;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* ── Main wrapper ────────────────────────────────── */
.main-wrapper {
  margin-left: 240px;
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  min-width: 0;
}

.filter-strip {
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
  position: sticky;
  top: 0;
  z-index: 50;
}

.main-content {
  flex: 1;
  padding: 1.5rem 2rem;
  background: #f8fafc;
}
```

---

## Phase 5 — Visual Polish (apply after sidebar works)

After the sidebar is functional, make these additional improvements via vue-expert:

1. **FilterBar** (`client/src/components/FilterBar.vue`): Remove any top-specific padding/margin that assumed a top nav above it. Add `max-width: 100%` and ensure it fills the filter strip cleanly.

2. **Global card styles**: Ensure `.card` has `margin-bottom: 1.25rem` and `.stats-grid` has appropriate `margin-bottom: 1.5rem`. These should already exist; verify they do.

3. If the app has a `.nav-container > .language-switcher` or similar positional rules keyed to `.nav-container`, remove them — the sidebar footer handles positioning now.

---

## Phase 6 — Verify

After all changes are applied:

1. Use Playwright (`mcp__playwright__*`) to navigate to `http://localhost:3000` and take a screenshot
2. Confirm the sidebar is visible on the left at ~240px
3. Navigate to `/inventory`, `/orders`, `/restocking` — confirm active state highlights correctly
4. Confirm the filter bar appears as a sticky strip below the sidebar header level
5. Confirm page content renders in the right panel without horizontal overflow

If there are rendering issues, read the browser console errors via Playwright and fix them before finishing.

---

## Constraints — Do Not Break These

- **All view components stay unchanged** — only `App.vue` and possibly `FilterBar.vue`
- **All i18n calls preserved** — no hardcoded strings replacing `t()` calls
- **All modals preserved** — ProfileDetailsModal, TasksModal must still open/close
- **All global utility CSS classes preserved** — views depend on them
- **All routes preserved** — no route changes
