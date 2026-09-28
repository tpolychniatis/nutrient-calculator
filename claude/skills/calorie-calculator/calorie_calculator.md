---
name: calorie-calculator
description: "Calculates calories and protein per 100g for a recipe, using a persistent, living ingredients database and recipe registry. Use this skill whenever the user gives a recipe (a list of ingredients with grams) and asks for its calorie or protein content, asks to 'calculate nutrition' for a dish, mentions adding an ingredient or recipe to their nutrition data, or references their 'ingredients table' or recipe registry. Also trigger if the user asks to look up an ingredient's nutrition facts, wants to update or review their nutrition database, or wants to see/list their saved recipes."
---

# Calorie Calculator

Computes calories/protein per 100g for a recipe using a living, file-based ingredient
database, filling gaps via web lookup (prioritizing ah.nl) with the user's confirmation
before anything is saved.

## Where the data lives

All data lives on the user's local machine, reachable through the `nutrition-filesystem`
MCP connector (NOT the general-purpose `filesystem` connector, which is scoped to an
unrelated folder — always use tools prefixed `nutrition-filesystem:`).

```
/Users/theodoros/claude/nutrition_data/
├── ingredients_table.csv     # the ingredient database
└── recipes/
    ├── <recipe-slug>.json    # one file per recipe
    └── ...
```

**`ingredients_table.csv` columns:** `ingredient,calories_per_100g,protein_per_100g,source`
- `ingredient`: lowercase-friendly name, matched case-insensitively
- `source`: where the values came from — `original spreadsheet (...)`, a URL, or similar. Every row must have one.

**Recipe JSON schema:**
```json
{
  "name": "recipe display name",
  "ingredients": [{"ingredient": "banana", "grams": 100.0}],
  "computed": {
    "total_grams": 100.0,
    "calories_per_100g": 93.0,
    "protein_per_100g": 1.0,
    "last_calculated": "YYYY-MM-DD"
  }
}
```
Slug the filename from the recipe name: lowercase, non-alphanumerics → `-` (e.g. "Choco Banana v2" → `choco-banana-v2.json`).

## Workflow

### 1. Load current data
Use `nutrition-filesystem:read_text_file` to read `ingredients_table.csv` at the start of
any calculation. Don't rely on memory of previous turns — always re-read, since the file
may have changed (e.g. edited by the user directly, or updated in an earlier session).

### 2. Match recipe ingredients
For each `{ingredient, grams}` in the recipe the user gives you, match against the table
case-insensitively. Allow near-matches (e.g. "eggs" vs "egg") but confirm with the user if
a match is ambiguous rather than silently guessing.

### 3. Handle missing ingredients
For each ingredient not found in the table:
1. Search **https://www.ah.nl/** first for the product and its nutrition label.
2. If not found there, do a general web search.
3. If still not found, tell the user plainly — do not estimate or guess a value.

Compile a lookup log for anything found: ingredient name, calories/100g, protein/100g,
and the source URL. Show this log to the user **before** writing anything to the CSV.

### 4. Get confirmation, then update the database
Only after the user confirms (they may confirm all, some, or give corrections):
- Append new rows to `ingredients_table.csv` via `nutrition-filesystem:write_file`
  (rewrite the full file with the new rows appended — the tool overwrites, it doesn't append in-place).
- Each new row's `source` column must be the URL you found it at.

### 5. Calculate the recipe
```
total_calories = sum(grams_i * calories_per_100g_i / 100)
total_protein  = sum(grams_i * protein_per_100g_i / 100)
total_grams    = sum(grams_i)
calories_per_100g = total_calories / total_grams * 100
protein_per_100g  = total_protein / total_grams * 100
```
Report both the per-100g figures and the totals for the full recipe as given.

### 6. Offer to save the recipe
Ask the user if they want this recipe saved to the registry. If yes, write a new file to
`nutrition_data/recipes/<slug>.json` following the schema above, with `last_calculated`
set to today's date.

## Notes
- Never overwrite `ingredients_table.csv` or a recipe file without having just read the
  latest version first in the same turn — avoids clobbering concurrent edits.
- If the `nutrition-filesystem` connector isn't available/connected, tell the user and ask
  them to check Claude Desktop's connector settings, rather than falling back to asking
  them to paste the CSV contents.
