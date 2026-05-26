---
name: optimize-vue
description: Analyze Vue 3 components for performance issues and code-reuse opportunities. Read-only — emits a categorized report with file:line citations and suggested fixes. Use when the user asks to optimize, audit, or review Vue components (or runs /optimize-vue).
---

# Vue Optimization Analyzer

This skill scans the Vue 3 frontend for known antipatterns and extraction opportunities, then emits a categorized report. It does **not** modify any files — apply fixes via the `vue-expert` agent in a separate step if the user asks.

## Scope

**Default scan target:** every `.vue` file under `client/src/` (views, components) and `.js` under `client/src/composables/`.

**Arguments:**
- `/optimize-vue` — scan the whole `client/src/` tree (default)
- `/optimize-vue <path>` — scan only that file or directory (e.g., `/optimize-vue client/src/views/Orders.vue`)
- `/optimize-vue --since main` — scan only `.vue` and composable files changed vs `main`

## How to run the analysis (efficiently)

Reading every `.vue` file blindly wastes context. Instead:

1. **Resolve scope first.** If an argument was passed, glob from that path. If `--since main`, run `git diff --name-only main...HEAD -- 'client/src/**/*.vue' 'client/src/composables/*.js'`. Otherwise glob `client/src/**/*.{vue,js}`.

2. **Fast pattern sweep with Grep.** Run the regex sweeps from the "Detection patterns" section below across the scope. Only `Read` a file in full once Grep has found at least one hit in it.

3. **Cross-file checks last.** For code-reuse opportunities (duplicated formatters, filters, modal scaffolding) collect candidate snippets from the per-file reads, then group by similarity.

4. **Cite line numbers.** Every finding must include `file:line` for the reader to jump to. If a finding spans a range, cite the starting line.

## Detection patterns

The rules below match the patterns in `.claude/agents/vue-expert.md`. When in doubt about whether something is idiomatic, that file is the source of truth — this skill operationalizes the same rules into greppable checks.

### Performance — P1 (correctness / reactivity bugs)

| # | Check | Grep (ripgrep) starting point |
|---|---|---|
| P1.1 | `v-for` keyed by the loop index variable — should be a stable unique ID | Single self-contained regex with a backreference: `rg -nP 'v-for="\([^,]+,\s*(\w+)\)[^"]*"[^>]*:key="\1"' client/src`. Captures the index variable from the `v-for` declaration and asserts it's the value of `:key` on the same element. No fragile fixed allow-list, no dependency on a specific name (`idx`, `i`, `rowIdx`, etc. all caught). |
| P1.2 | Direct prop mutation — `props.X = ...`, `props.foo.bar = ...`, `props.items[0].field = ...`, `props.array.push(...)` | `rg -n '\bprops\.[a-zA-Z_.]+(\[[^\]]+\])?\s*(=[^=]\|\.push\(\|\.splice\(\|\.shift\(\|\.unshift\(\|\.pop\()' client/src`. The `=[^=]` form (not bare `=`) prevents false positives from `===`/`==` comparisons. The `[a-zA-Z_.]+` covers nested paths, the optional `\[…\]` covers indexed mutation. |
| P1.3 | Unvalidated date parsing — `new Date(x).getMonth()` (etc.) without an `isNaN(d.getTime())` guard | `rg -n 'new Date\([^)]+\)\.(getMonth\|getDate\|getFullYear\|getTime\|getDay\|getHours)' client/src` |
| P1.4 | Forgotten `.value` in `<script>` on a ref — `ref()` declared but used without `.value` in setup logic | Per-file inspection after Read — look for `const X = ref(` then later `if (X)` / `X.push(` / `X.length` without `.value` |
| P1.5 | Mixing Options API and Composition API in the same component | Look for both `export default { setup()` *and* sibling keys like `data()`, `methods:`, `computed:` |

These are **bugs**, not optimizations. Surface them at the top of the report.

### Performance — P2 (real speedups)

| # | Check | Heuristic |
|---|---|---|
| P2.1 | Template invokes a `methods:` function or arrow expression that returns derived data — should be `computed` (cached) | After Read, find `<template>` usages of functions defined in `setup()` whose body has no side effects and only reads reactive state. **Only flag if you're confident** — this check has a high false-positive rate (stateless formatters that take args legitimately stay as functions; `computed` only helps for parameterless, dependency-cached values). If in doubt, skip. |
| P2.2 | `v-if` on an element that toggles frequently (e.g., bound to filter / hover / tab state) — consider `v-show` to skip mount/unmount | Look for `v-if="<reactive flag>"` inside loops or near filter controls |
| P2.3 | Deep watcher on a large object where a `computed` would suffice | `rg -n 'watch\(' client/src` then inspect for `{ deep: true }` on large reactive objects |
| P2.4 | Inline arrow function passed as event handler inside `v-for` — recreates each render | `rg -nP '@(click\|input\|change)="\(\)\s*=>' client/src` |
| P2.5 | Big `<template>` block (>150 lines) — candidate for extraction into subcomponents | Per-file: count lines between `<template>` and `</template>` |
| P2.6 | `<script setup>` with top-level `await` and no `defineAsyncComponent`/`<Suspense>` boundary in the parent | After Read — grep for `^await ` at script top level |

### Performance — P3 (code health)

| # | Check | Heuristic |
|---|---|---|
| P3.1 | `<style>` block missing `scoped` — risks global CSS leakage. (Exception: `App.vue`, which is intentionally global per the design system in CLAUDE.md.) | `rg -nP '<style(?! scoped)' client/src --glob '*.vue'` (allow-list `App.vue`). Note: ripgrep has no built-in `vue` file type, so use `--glob '*.vue'` rather than `--type=vue` (the latter silently matches nothing). |
| P3.2 | Component over 400 lines total — split candidate | Per-file line count |
| P3.3 | Magic strings/numbers (warehouse names, status names, color hexes) hardcoded in `.vue` files that already exist as constants elsewhere | Cross-reference against existing constants and `vue-expert.md`'s "Common Values" list |

### Code reuse — R1 (extract to composables or utils)

| # | Check | How to detect |
|---|---|---|
| R1.1 | Same currency / number / date formatting expression repeated across files | Two-step: (1) `ls client/src/utils/` and grep for `export function format` to discover existing formatters. (2) `rg -n 'toLocaleString' client/src --glob '!**/utils/**'` to find raw call sites *excluding the utils directory itself* (otherwise the util's own implementation matches and gets falsely flagged). **If a util already exists**, frame the finding as "drift from existing util" — list the inline call sites and recommend replacing them with the existing helper (the util is the source of truth; one JPY bug fix would otherwise have to touch every site). **If no util exists**, frame as "extract a new helper" and suggest `client/src/utils/format.js` → `formatCurrency(value)`. |
| R1.2 | Repeated `loading`/`error`/`data` ref triplet with the same `try/catch` shape — candidate for a `useAsyncData` composable | After Read — look for the trio `const loading = ref(false); const error = ref(null); const data = ref(...)` |
| R1.3 | Inline filter logic that re-implements what `useFilters` already does | `rg -n 'selectedWarehouse\.value\|selectedCategory\.value\|filters\.warehouse' client/src` — if a `.vue` filters locally but doesn't import `useFilters`, flag |
| R1.4 | Identical or near-identical `computed` definitions across multiple files | After Read — collect computed names + body hashes, group |

### Code reuse — R2 (extract to components)

| # | Check | How to detect |
|---|---|---|
| R2.1 | Multiple files use the same modal scaffolding (`.modal-overlay > .modal-container > .modal-header`) — extract a `BaseModal` | Grep for the class names; if 3+ files match, flag |
| R2.2 | Repeated stat-card markup (`.stat-card > .stat-label + .stat-value`) — extract a `StatCard` component | Grep for `class="stat-card"`; if 4+ instances across files, flag |
| R2.3 | Repeated table layouts with similar `<thead>`/`<tbody>` structure | Per-file: compare table column counts and header text |

## Output format

Emit a single Markdown report to the chat (do not write a file unless the user asks). Use this exact structure so the user can scan it quickly:

````markdown
# Vue Optimization Report
**Scope:** <N files scanned> · <branch or path>
**Findings:** <X> performance · <Y> reuse · <Z> code health

---

## P1 — Bugs & reactivity issues (fix first)

### 1. <one-line title>
**File:** `client/src/views/Orders.vue:42`
**Pattern:**
```vue
<tr v-for="(order, index) in orders" :key="index">
```
**Why it matters:** Index keys cause Vue to reuse the wrong DOM nodes when the list reorders, leading to stale form state and broken transitions.
**Fix:** Use `order.id` as the key.

### 2. ...

## P2 — Performance opportunities

(same structure)

## P3 — Code health

(same structure)

## R1 — Logic extraction opportunities

### 1. Currency formatting duplicated across 3 files
**Files:**
- `client/src/views/Orders.vue:88`
- `client/src/views/Spending.vue:104`
- `client/src/views/Inventory.vue:67`

**Shared expression:**
```javascript
value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
```
**Suggested home:** `client/src/utils/format.js` → `formatCurrency(value)`.

## R2 — Component extraction opportunities

(same structure)

---

## Suggested next steps
1. Apply P1 fixes (bugs). For .vue changes, delegate to the `vue-expert` agent per the project's mandatory rule.
2. Pick 1–2 P2/R1 items with the highest payoff and tackle in a focused commit.
3. Hold R2 component extractions for a follow-up PR — they're larger refactors.
````

If there are zero findings in a category, write `_No findings._` under that heading rather than omitting it — that makes it easy to see at a glance what was checked.

## Constraints

- **MANDATORY: read-only.** Do not use `Edit`, `Write`, `NotebookEdit`, or any `git`-mutating Bash command for the duration of this skill. This skill is a read-and-report loop. If a finding looks trivially fixable, *still emit it as a finding* — do not silently apply it. Mutating `.vue` files outside the `vue-expert` agent also violates the project CLAUDE.md's `MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert`.
- **No dev server.** Do not start, restart, or hit the running app. Static analysis only.
- **Cite, don't paraphrase.** Every finding must include the actual offending snippet (verbatim from the file) and the file:line.
- **Don't moralize.** A finding is `pattern + location + concrete suggestion`, not "consider thinking about whether perhaps you might want to refactor."
- **Skip the bikeshed.** Things that aren't worth flagging: cosmetic indentation, single-word naming preferences, `var` vs `const` debates (project uses `const` throughout), missing trailing commas.
- **Respect the design system.** The CLAUDE.md mandates no emojis in UI and a specific slate/gray palette — don't suggest visual changes that contradict it.

## When to *not* run this skill

- The user wants a fix applied, not a report → use the `vue-expert` agent directly.
- The user is asking about backend code → this skill ignores `server/`.
- The repo isn't a Vue 3 project (no `client/src/**/*.vue`) → return early with "no Vue files found."

## Calibration check

Before emitting the report, sanity-check yourself:
- Did I cite at least one P1 if any v-for index keys exist? (They do — `vue-expert.md` lists them as the #1 thing to never do.)
- Did I check for the `useFilters` composable import in files that filter locally?
- Did I dedupe near-identical findings (e.g., 5 instances of the same date-format expression should be ONE R1 finding, not 5 P1 findings)?
- Is the report scannable in under 60 seconds? If it's longer than ~200 lines, prune the lowest-severity items into a final "Other minor findings" bullet list.
