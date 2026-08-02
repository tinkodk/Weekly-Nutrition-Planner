# Weekly Nutrition Planner — CLAUDE.md

## Project Overview

Hebrew/English bilingual web app that turns a list of purchased groceries into a balanced weekly meal plan for a busy student. Given the products a user bought this week, it picks recipes from a curated database that maximize use of what's already owned, and outputs a 7-day × 4-meal plan (breakfast/lunch/dinner/snack) with full recipes and a complementary shopping list for missing ingredients.

## Architecture

**Single-file app**: Everything lives in `index.html` — HTML, CSS, and JS in one file. No build tools, no npm, no bundler.

**Data persistence**: `localStorage` under key `nutriplan_v1`. All state changes call `saveData()`. `loadData()` provides an empty default (`{lang, products: [], plan: null}`) — there is no seed data, since grocery lists change weekly and aren't meant to be committed like the wedding app's guest list. Git is used to host/back up the *code*, not the weekly product list.

**Hosting**: static site pushed to a public GitHub repo, served via GitHub Pages so it's reachable via URL from any computer. The files also live locally under OneDrive (`PersonalCode/Food/`).

## Data Model

- `CATEGORIES` — food-group taxonomy (protein, dairy, grain, legume, veg, fruit, fat, sweet, other)
- `PRODUCTS` — catalog of ~48 common grocery items, each `{id, he, en, cat}`
- `RECIPES` — ~30 recipes, each `{id, mealType, prepMin, name:{he,en}, ingredients:[{id, qty:{he,en}}], instructions:{he:[...],en:[...]}, nutrition:{he,en}}`. Ingredients reference `PRODUCTS` ids for matching.
- `db` (persisted state): `{lang, products: [{catalogId|null, category, label|null}], plan: [{breakfast:{recipeId,missing[]}, lunch:..., dinner:..., snack:...} × 7]}`

## Menu Engine (`generatePlan()`)

For each of 7 days × 4 meal slots, scores every candidate recipe of that meal type by:
- ingredient coverage (fraction of the recipe's ingredients the user already owns)
- a recency penalty so the same recipe isn't picked on consecutive/nearby days (variety)
- small random jitter so "Regenerate" produces a different plan

Missing ingredients per recipe are collected across the week into the shopping list (`renderShoppingList()`), counted by how many recipes need each item.

Nutrition values are pre-authored approximate ranges per recipe (not computed from ingredients) — this is intentional per product spec (general balance guidance, not exact macro tracking).

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
