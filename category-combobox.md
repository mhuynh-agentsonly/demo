# Category Selector / Creator — Component Reference

**File:** `skills-center-admin-v1.html`  
**Used in:** Add Skill form (ASF) — the "Category" field (half-width, alongside the Skill Name field)

---

## Overview

A single-select combobox that lets admins either pick an existing category or type a new name to create one on the fly. It is **not** a native `<select>` — it's a custom dropdown built from a `<button>` trigger, a hidden `<input>`, and a manually rendered listbox.

---

## DOM Structure

```
div#asf-category-wrap (relative positioned container)
├── input#asf-category          (hidden; holds the committed value for form validation)
├── button#asf-category-trigger (visible trigger; opens/closes dropdown)
│     └── span#asf-category-label  ("Select category" placeholder or selected value)
└── div#asf-category-dropdown   (hidden until open)
      ├── div.p-2 (search row)
      │     ├── input#asf-category-search  (text input: "Search or create category…")
      │     └── div#asf-cat-counter        (live char counter, shown when typing)
      ├── div#asf-category-options[role="listbox"]  (rendered option buttons)
      └── div#asf-category-create-row      (hidden unless search has no exact match)
            └── button#asf-category-create-btn  ("Create <label>" CTA)
```

---

## State

| Variable | Type | Description |
|---|---|---|
| `CATEGORY_OPTIONS` | `string[]` | Mutable master list. Seeded with `['Coding', 'Medical', 'Language', 'AI/ML', 'Customer Service', 'Annotation']`. New categories are pushed here at runtime. |
| `selectedCategoryValue` | `string` | Currently committed selection. Empty string = nothing selected. |

---

## Functions

### `toggleCategoryCombobox()`
Called by `onclick` on `#asf-category-trigger`.  
- If dropdown is **hidden**: closes any open row menus (`closeAllRowMenus()`), shows the dropdown, sets `aria-expanded="true"`, clears the search input, calls `renderCategoryOptions('')`, then focuses the search field.  
- If dropdown is **visible**: hides it and sets `aria-expanded="false"`.

### `closeCategoryCombobox()`
Hides `#asf-category-dropdown` and resets `aria-expanded` to `"false"`. Called after a selection is committed or on Escape.

### `filterCategoryOptions(q: string)`
`oninput` handler on the search field. Trims whitespace and delegates to `renderCategoryOptions(q)`.

### `renderCategoryOptions(q: string)`
Core render function.  
1. Filters `CATEGORY_OPTIONS` case-insensitively against `q`.  
2. Renders each match as a `<button role="option">`. Selected option gets a checkmark SVG and `text-primary-700 font-medium` styling.  
3. Checks for an **exact match** (case-insensitive). If `q` is non-empty and no exact match exists, unhides `#asf-category-create-row` and sets the create-button label to the typed string.  
4. If `q.length > 30`, the create button is disabled (opacity 0.35) — enforces a 30-char max on new category names.

### `selectCategoryOption(cat: string)`
Commits a selection:  
1. Sets `selectedCategoryValue = cat`.  
2. Writes `cat` to `input#asf-category` (the hidden form field).  
3. Updates `#asf-category-label` text and removes the muted placeholder class.  
4. Calls `closeCategoryCombobox()`.  
5. Calls `checkFormValidity()` to re-evaluate whether the Publish / Save Draft buttons should be enabled.

### `createAndSelectCategory()`
Called by the "Create …" button (`onclick`) **and** by `handleCategorySearchKey` on Enter when no exact match exists.  
1. Reads the current value of `#asf-category-search`.  
2. If the string is not already in `CATEGORY_OPTIONS`, pushes it.  
3. Calls `selectCategoryOption(newCat)` — so creation and selection are atomic.  
> **Side-effect:** the new category persists in `CATEGORY_OPTIONS` for the lifetime of the page, making it immediately available in future opens of this combobox.

### `handleCategorySearchKey(e: KeyboardEvent)`
`onkeydown` handler on the search field.  
- **Escape** → `closeCategoryCombobox()`.  
- **Enter** → prevents default form submission, then either selects the exact-matching option or calls `createAndSelectCategory()` if the typed string is new.

### `updateCatCounter(input: HTMLInputElement)`  *(inline, via `oninput`)*
Shows `#asf-cat-counter` once the user starts typing; displays `{length}/30` to signal the character limit.

---

## Integration Points

- **Form validation** (`checkFormValidity`): reads `input#asf-category`'s value. An empty value keeps the Publish and Save Draft buttons disabled.
- **List-screen filter** (`renderListCategoryFilter`): a separate multi-select combobox (`#list-cat-wrap`) that filters the skills table. It derives its option set dynamically from `adminSkills[].category` — **not** from `CATEGORY_OPTIONS` — so seeded data and runtime-created categories only appear there after a skill is saved with that category.
- **Skills data model**: each skill object carries a `category: string` field. This field is what the list filter and the test-coverage matrix (`getTestCategoryMap`) read from.

---

## Accessibility

- Trigger uses `aria-haspopup="listbox"` and `aria-expanded` (toggled programmatically).  
- Dropdown uses `role="listbox"` + `aria-label`; each option uses `role="option"` + `aria-selected`.  
- Focus is moved to the search field on open (`setTimeout(() => search.focus(), 0)`).  
- Keyboard: Escape closes, Enter selects or creates.

---

## Constraints / Edge Cases

| Constraint | Detail |
|---|---|
| Max new-category length | 30 chars (create button disabled beyond this) |
| Case sensitivity | Filtering is case-insensitive; storage preserves the casing typed by the admin |
| Duplicate prevention | `createAndSelectCategory` checks `CATEGORY_OPTIONS.includes(newCat)` before pushing |
| Persistence | `CATEGORY_OPTIONS` is an in-memory array — new categories are lost on page reload |
| Single-select only | Only one category per skill; selecting a new option replaces the previous one |
