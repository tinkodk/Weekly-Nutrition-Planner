# Weekly Nutrition Planner — CLAUDE.md

## Project Overview

Hebrew/English bilingual web app that turns a list of purchased groceries into a balanced weekly meal plan for a busy student. Given the products a user bought this week, it picks recipes from a curated database that maximize use of what's already owned, and outputs a 7-day × 4-meal plan (breakfast/lunch/dinner/snack) with full recipes and a complementary shopping list for missing ingredients.

## Architecture

**Single-file app**: Everything lives in `index.html` — HTML, CSS, and JS in one file. No build tools, no npm, no bundler.

**Data persistence**: `localStorage`, namespaced per profile (see Profiles below). All state changes call `saveData()`. There is no seed data, since grocery lists change weekly and aren't meant to be committed like the wedding app's guest list. Git is used to host/back up the *code*, not the weekly product list.

**Profiles (multi-person storage)**: this is a shared public app, so `db`/`syncConfig` are namespaced per named profile instead of one fixed key — `localStorage` keys are `nutriplan_v1::<name>` and `nutriplan_sync_v1::<name>`, with `nutriplan_profiles_v1` holding `{list:[names...], current:name}`. On first-ever visit (no profile picked yet) the app opens a **forced** name modal (`profileModal`, can't be dismissed until a name is entered) — if legacy un-namespaced `nutriplan_v1`/`nutriplan_sync_v1` data exists from before this feature, naming a profile migrates that data into the new key instead of losing it (`createProfile()`, first-profile-only branch). A 👤 dropdown in the header (`profileSelect`) lets the same browser switch between profiles or add a new one (`+ Add profile...` option, not forced, cancelable) — this is the "same computer, multiple people" case; the far more common case is each person on their own device with their own name, which is already isolated by browser but still gets a friendly name.

**Weekly history**: state is `{lang, weeks: [{id, label, createdAt, products, plan}], currentWeekId}` — each week keeps its own products and generated plan, so starting a new week (`newWeekBtn`) never deletes prior weeks. `currentWeek()` resolves the active week; the week `<select>` switches `db.currentWeekId`. `loadData()` migrates the older flat schema (`{lang, products, plan}`, no `weeks` array) into a single-week `weeks` array on first load, so no existing user data is lost.

**Hosting**: static site pushed to a public GitHub repo, served via GitHub Pages so it's reachable via URL from any computer. The files also live locally under OneDrive (`PersonalCode/Food/`).

**Cloud data sync**: same pattern as the wedding-planning app — a private GitHub Gist, not git commits. The user pastes a personal GitHub token (`gist` scope only) into the ☁️ Sync modal; `saveData()` debounces (3s) into `saveToGist()`, which PATCHes/POSTs the whole `db` object as `nutriplan-data.json` to a private Gist. `loadFromGist()` pulls it back down on another computer. Token + Gist ID are per-profile (see above), so each named person syncs to their own Gist — switching the profile dropdown also switches which Gist auto-save targets.

## Data Model

- `CATEGORIES` — food-group taxonomy (protein, dairy, grain, legume, veg, fruit, fat, sweet, other)
- `PRODUCTS` — catalog of ~48 common built-in grocery items, each `{id, he, en, cat}`. Fixed, not user-editable.
- `db.customProducts` — the user's **permanent personal catalog** of free-text items that didn't match `PRODUCTS`, each `{id, label, cat}`. Created via `findOrCreateCustomProduct()` on free-text add (default `cat:'other'`). These persist across weeks just like built-ins — `renderCatalogAccordion()` lists every custom product under its `cat` (including `'other'` → "Uncategorized" section) as a plain checkbox alongside the catalog items for that category, checked only if owned *this* week. Clicking the ✎ button next to a custom item opens `editCustomModal` to rename it, change its category, or delete it permanently (from `customProducts` and every week's `products`). Unchecking a custom item's checkbox just un-owns it for the current week — it does **not** delete the definition.
- `RECIPES` — ~30 recipes, each `{id, mealType, prepMin, name:{he,en}, ingredients:[{id, qty:{he,en}}], instructions:{he:[...],en:[...]}, nutrition:{he,en}}`. Ingredients reference `PRODUCTS` ids only (custom products never define new ingredients recipes can require).
- Each week: `{id, label, createdAt, products: [{catalogId, category, label:null} | {customId}], plan: [{breakfast:{recipeId,missing[]}, lunch:..., dinner:..., snack:...} × 7] | null}`

## Menu Engine (`generatePlan()`)

For each of 7 days × 4 meal slots, scores every candidate recipe of that meal type by:
- ingredient coverage (fraction of the recipe's ingredients the user already owns)
- a recency penalty so the same recipe isn't picked on consecutive/nearby days (variety)
- small random jitter so "Regenerate" produces a different plan

Missing ingredients per recipe are collected across the week into the shopping list (`renderShoppingList()`), counted by how many recipes need each item.

Nutrition values are pre-authored approximate ranges per recipe (not computed from ingredients) — this is intentional per product spec (general balance guidance, not exact macro tracking).

**Fuzzy custom-product coverage**: `isIngredientCovered(ingId, ownedCatalog, ownedCustomCats)` treats a recipe ingredient as covered if the user owns it exactly (catalog match) OR owns *any* custom product in that ingredient's category this week (`ownedCustomCategorySet()`). This is deliberately coarse — e.g. owning a custom "fat"-category item counts toward *every* fat ingredient (olive oil, tahini, avocado, ...), not just a specific one — chosen intentionally over exact free-text-to-ingredient matching, which isn't feasible.

## i18n / RTL

- `STRINGS` holds all UI chrome text as `{he, en}` (or `{he: fn, en: fn}` for pluralized/parametrized strings).
- `t(key)` returns the current-language string/function; `tf(obj)` returns `obj[db.lang]` for inline bilingual data (products, recipes).
- `applyStaticStrings()` re-renders all UI text and toggles `document.documentElement.dir`/`lang` — called on load and on language toggle.
- Language toggle does **not** regenerate the plan; it just re-renders existing state in the other language.

## Working in This Repo

- All changes go into `index.html` — edit HTML, CSS, and JS all in that one file
- No tests, no CI — verify by opening `index.html` (serve over local HTTP, not `file://`, since the in-app browser tool blocks local file navigation)
- To add a recipe: add to `RECIPES` with a unique id, ingredients referencing existing `PRODUCTS` ids (add new catalog entries first if needed)
- To add a UI string: add to `STRINGS`, wire it via `applyStaticStrings()` if it's static chrome text
