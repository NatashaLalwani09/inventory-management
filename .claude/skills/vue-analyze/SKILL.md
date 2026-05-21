---
name: vue-analyze
description: Analyzes Vue 3 component structure and suggests optimizations for performance and code reuse. Use when asked to audit, analyze, or optimize Vue components.
---

# Vue Component Analysis Skill

When invoked, perform a structured analysis of the Vue 3 components in `client/src/`. Produce a prioritized report with specific file:line references and concrete fix examples.

## Step 1 — Inventory

Read every `.vue` file under `client/src/` (views + components). Also read the three existing composables: `useFilters.js`, `useI18n.js`, `useAuth.js`.

## Step 2 — Run All Six Checks

### Check A: Duplicated `computed` logic across views

Look for the same computed body appearing in multiple files. Known pattern in this codebase:

```js
// Repeated verbatim in Inventory.vue, Orders.vue, Spending.vue, Dashboard.vue
const currencySymbol = computed(() => { ... })
```

**Fix:** move into `useI18n` composable so every consumer gets it for free.

For each duplicate found, report:
- Which files contain it
- The exact lines
- The one-line composable addition that eliminates the copies

---

### Check B: `watch` that should be `watchEffect` or `computed`

Flag any `watch` whose callback does nothing except:
- Set another `ref` derived from the watched value, OR
- Reload data using the watched value directly

```js
// Anti-pattern: watch + manual reload
watch([selectedLocation, selectedCategory], () => {
  loadInventory()
})
// Better: watchEffect auto-tracks its own dependencies
watchEffect(() => {
  loadInventory()
})
```

Report file, line, and the rewritten snippet.

---

### Check C: `v-for` key quality

Flag any `v-for` using `:key="index"` (array index). Index keys cause Vue to reuse the wrong DOM node when items are reordered or deleted.

```html
<!-- Bad -->
<div v-for="(item, idx) in order.items" :key="idx">

<!-- Good — use a stable identity field -->
<div v-for="item in order.items" :key="item.sku">
```

Known instance: `Orders.vue:87`. Report all occurrences with a suggested stable key field.

---

### Check D: Repeated data-loading boilerplate

Every view follows this identical pattern:

```js
const loading = ref(true)
const error = ref(null)

const loadXxx = async () => {
  try {
    loading.value = true
    // ... fetch ...
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}
onMounted(loadXxx)
watch([...filters], loadXxx)
```

This is a strong signal for a `useAsyncData(fetchFn, deps)` composable:

```js
// composables/useAsyncData.js
export function useAsyncData(fetchFn, deps = []) {
  const data = ref(null)
  const loading = ref(true)
  const error = ref(null)

  const load = async () => {
    loading.value = true
    error.value = null
    try {
      data.value = await fetchFn()
    } catch (err) {
      error.value = err.message
    } finally {
      loading.value = false
    }
  }

  if (deps.length) watch(deps, load)
  onMounted(load)

  return { data, loading, error, reload: load }
}
```

Count how many views would be simplified and estimate lines saved.

---

### Check E: Large single-file components

Flag any `.vue` file where the `<script setup>` / `<script>` block exceeds ~150 lines. Large scripts are hard to test in isolation and usually contain logic that belongs in composables.

For each oversized file:
- Report the line count
- Identify which logical groups could be extracted (e.g., "filter logic", "modal state", "formatting helpers")
- Suggest composable names

---

### Check F: Props drilling / missing component extraction

Look for template patterns that repeat across multiple files — especially modal triggers, status badges, and table rows with the same structure. These are candidates for small reusable components.

Known patterns to check:
- Status badge `<span :class="...">{{ status }}</span>` — appears in Orders, Inventory, Dashboard
- Loading/error states — every view renders an identical spinner + error block
- "Create PO" button pattern in backlog/dashboard tables

For each, note the files involved and propose a component name + props interface.

---

## Step 3 — Report Format

Output a report with this structure:

```
## Vue Component Analysis

### Summary
| Check | Issues Found | Effort |
|-------|-------------|--------|
| A - Duplicated computed | N | Low |
| B - watch vs watchEffect | N | Low |
| C - v-for key quality | N | Low |
| D - Async data boilerplate | N files affected | Medium |
| E - Oversized components | N files | Medium |
| F - Missing component extraction | N patterns | Medium-High |

---

### A - Duplicated Computed Logic
[file:line for each instance, one-line fix]

### B - Watch Improvements
[file:line, before/after snippet]

### C - v-for Key Quality
[file:line, suggested stable key]

### D - useAsyncData Composable
[list of views, estimated lines saved, full composable code]

### E - Oversized Components
[file, line count, extraction suggestions]

### F - Reusable Component Candidates
[pattern name, files, proposed component interface]

---

### Quick Wins (fix in < 30 min)
[bullet list of Checks A-C findings - all low-effort]

### Bigger Refactors
[bullet list of Checks D-F findings with effort estimate]
```

## Constraints

- Reference specific `file:line` for every finding - never vague "somewhere in the codebase".
- Show before/after code for every suggestion.
- Do NOT auto-apply changes unless the user says "fix it" or "apply". Analysis only by default.
- Respect the existing composable structure (`useFilters`, `useI18n`, `useAuth`) - extend them, do not replace them.
- Keep the Design System in mind: no emojis in UI, slate/gray color tokens, SVG charts.
