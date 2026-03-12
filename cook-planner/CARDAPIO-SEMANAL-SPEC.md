# Cardápio Semanal — Project Spec

## Overview

Single-page HTML app (mobile-first) for weekly meal planning. Deployed via GitHub Pages. No backend, no frameworks — vanilla HTML/CSS/JS in one file. The app reads meal data dynamically from an `.xlsx` file stored in the same repository (parsed at runtime with SheetJS), lets the user build a weekly menu, generates a shopping list grouped by store, and exports two PDFs.

**Key workflow**: To add or change dishes, the user edits `data/cardapio_template.xlsx`, pushes to GitHub, and the app picks up the changes automatically — no code edits needed.

---

## Data Architecture: xlsx as Single Source of Truth

### File: `data/cardapio_template.xlsx`

Single sheet called "Cardápio" with 4 columns:

| Prato | Categoria | Ingrediente | Onde Comprar |
|-------|-----------|-------------|--------------|
| Franguinho empanado | Proteína | Frango Sassami | Swift |
| Cubos de frango com ervilha e milho | Proteína | Frango em cubos congelado | Swift |
| Cubos de frango com ervilha e milho | Proteína | Milho lata | Supermercado |
| ... | ... | ... | ... |

**Rules**:
- Each row = one ingredient of one dish
- A dish with 3 ingredients = 3 rows with the same dish name
- Ingredient names must be exactly consistent across dishes (for consolidation)
- Categories: `Proteína`, `Acompanhamento`, `Legume/Vegetal`

### Runtime parsing (SheetJS)

On app load:
1. Fetch `data/cardapio_template.xlsx` via relative URL
2. Parse with SheetJS (`XLSX.read()`)
3. Read the "Cardápio" sheet, skip header row
4. Build the internal data structure:

```javascript
// Parsed result shape (built from xlsx at runtime)
const MEAL_DATA = {
  dishes: {
    "Proteína": [
      {
        name: "Franguinho empanado",
        ingredients: [
          { item: "Frango Sassami", store: "Swift" }
        ]
      },
      // ...
    ],
    "Acompanhamento": [ /* ... */ ],
    "Legume/Vegetal": [ /* ... */ ]
  }
};
```

**Parsing logic**:
```
1. Read all rows from sheet
2. Group rows by column A (Prato name)
3. For each group, extract category from column B (same for all rows of that dish)
4. Collect ingredients: { item: column C, store: column D }
5. Organize into dishes object grouped by category
```

### Hardcoded config (in JS, not xlsx)

These rarely change and don't belong in the spreadsheet:

```javascript
const CONFIG = {
  // Pantry staples — "Confere se tem em casa" section
  pantryStaples: ["Azeite", "Manteiga", "Leite", "Cebola", "Alho", "Óleo", "Sal"],

  days: [
    "Segunda-feira", "Terça-feira", "Quarta-feira",
    "Quinta-feira", "Sexta-feira", "Sábado", "Domingo"
  ],

  dayAbbrev: {
    "Segunda-feira": "seg", "Terça-feira": "ter",
    "Quarta-feira": "qua", "Quinta-feira": "qui",
    "Sexta-feira": "sex", "Sábado": "sáb", "Domingo": "dom"
  }
};
```

---

## Design Theme: Botanical Garden

A fresh, organic, premium aesthetic inspired by botanical illustrations and natural textures. This should feel like a beautifully designed recipe book — warm, inviting, and professional.

### Color Palette

| Token             | Hex       | Usage                                      |
|--------------------|-----------|---------------------------------------------|
| `--fern-green`     | `#4a7c59` | Primary — headers, buttons, active states   |
| `--marigold`       | `#f9a620` | Accent — highlights, badges, CTAs           |
| `--terracotta`     | `#b7472a` | Secondary accent — alerts, delete actions   |
| `--cream`          | `#f5f3ed` | Background base                             |
| `--dark-soil`      | `#2c2c2c` | Text primary                                |
| `--warm-gray`      | `#8a8578` | Text secondary, borders                     |
| `--white`          | `#ffffff` | Cards, modals                               |
| `--light-sage`     | `#e8ede5` | Subtle section dividers, hover states       |

### Typography

- **Headers**: `"Playfair Display", Georgia, serif` — elegant, editorial feel
- **Body**: `"DM Sans", system-ui, sans-serif` — clean, modern readability
- Import both from Google Fonts
- Use generous line-height (1.6 for body, 1.2 for headers)
- Font sizes should feel comfortable on mobile (base 16px, scale up for headers)

### Visual Details

- Subtle paper/parchment texture on the cream background (CSS noise or subtle pattern)
- Cards with soft shadows (`0 2px 8px rgba(0,0,0,0.06)`) and rounded corners (`12px`)
- Botanical leaf/vine decorative SVG elements as subtle accents (section dividers, empty states)
- Smooth transitions on all interactive elements (300ms ease)
- Micro-animations: cards fade-in on load with stagger, buttons have gentle scale on hover
- Bottom sheet / modal for meal selection slides up with CSS animation
- No harsh borders — use shadow and background contrast for hierarchy

### Mobile-First Layout

- Max-width `480px` centered, with `16px` padding
- Touch-friendly tap targets (min `44px`)
- Horizontal scroll for the week days (snap scroll) OR vertical card stack
- Sticky header with app title and action buttons

---

## App Structure & Flow

### Screen 1: Weekly Planner (Main)

Seven day cards (Segunda to Domingo), each containing:

```
┌─────────────────────────────┐
│  SEGUNDA-FEIRA        🗑️    │
│                             │
│  🍽️ Almoço                  │
│  Proteína: [Strogonoff ▼]  │
│  Acompanhamento: [Arroz ▼] │
│  Legume: [Brócolis ▼]      │
│                             │
│  + Adicionar jantar         │ ← optional, collapsed by default
│                             │
└─────────────────────────────┘
```

- Each slot shows a `(+)` button when empty, and the dish name + `(x)` to remove when filled
- Clicking `(+)` opens a **bottom sheet modal** with dishes filtered by category
- The 🗑️ clears all selections for that day
- "+ Adicionar jantar" expands to show the same 3 slots for dinner
- If dinner is not set, it doesn't appear in PDFs/shopping list
- If dinner is not set, the weekly menu PDF shows "Sobra do almoço" for that day's dinner

### Loading State

While the xlsx is being fetched and parsed (usually < 1 second):
- Show a centered loading spinner or botanical-themed animation
- Message: "Carregando cardápio..."
- If fetch fails (offline, file missing), show friendly error with retry button

### Screen 2: Shopping List (Checklist)

Triggered by "Lista de Compras" button. Shows:

#### Section: "Confere se tem em casa"
Fixed checklist of pantry staples:
```
☐ Azeite  ☐ Manteiga  ☐ Leite  ☐ Cebola  ☐ Alho  ☐ Óleo  ☐ Sal
```

#### Section: Shopping list grouped by store
```
🏪 SUPERMERCADO
☐ Creme de leite (Strogonoff de frango — seg)
☐ Molho de tomate (Strogonoff — seg, Macarrão — qua)
☐ Macarrão (Macarrão molho vermelho — qua)

🥩 AÇOUGUE
☐ Kafta pronta congelada (Kafta — ter)
☐ Linguiça toscana (Linguiça toscana — qui)

🧊 SWIFT
☐ Frango em cubos congelado (Strogonoff — seg, Cubos de frango — qua)
☐ Frango Sassami (Franguinho empanado — sex)
```

**Consolidation logic**: Same ingredient name + same store → merge into one line, list all dishes and days in parentheses.

**Checked items**: strikethrough + dimmed opacity. State saved in localStorage.

**Buttons at bottom**:
- "Baixar PDF da Lista" — generates PDF with checkboxes
- "Voltar ao Cardápio"

### Bottom Sheet Modal (Meal Selection)

Slides up from bottom, max 60vh height, scrollable. Shows:
- Title: "Escolha a proteína" (or acompanhamento/legume)
- List of dishes in that category as tappable cards/pills
- Each dish card shows just the name
- Tap selects and closes the modal
- "X" or swipe down to dismiss

---

## PDF Generation

Use **jsPDF** + **jsPDF-AutoTable** from CDN for both PDFs.

### PDF 1: Cardápio da Semana

Title: "Cardápio da Semana" + date range (ex: "10–16 Mar 2026")

Table format:
| Dia | Almoço | Jantar |
|-----|--------|--------|
| Segunda | Strogonoff + Arroz + Brócolis | Sobra do almoço |
| Terça | Kafta + Macarrão + Milho e ervilha | Sopinha |

- Use Botanical Garden colors in the PDF header
- Clean, readable table

### PDF 2: Lista de Compras

Title: "Lista de Compras — Semana 10–16 Mar 2026"

Section 1: "Confere se tem em casa" with ☐ checkboxes

Section 2: Grouped by store:
```
SUPERMERCADO
☐ Creme de leite (Strogonoff de frango — seg)
☐ Molho de tomate (Strogonoff — seg, Macarrão — qua)

AÇOUGUE
☐ Kafta pronta congelada (Kafta — ter)
```

---

## Shopping List Consolidation Logic

```
1. For each day in the week:
   a. Get all selected dishes (lunch + dinner if set)
   b. For each dish, get its ingredients from MEAL_DATA
   c. For each ingredient, record: { item, store, dishName, dayAbbrev, meal }

2. Group by (item + store):
   a. Merge entries with same item + store
   b. Build context string: "DishA — seg, DishB — qua"

3. Group by store for display

4. Sort stores alphabetically, items within each store alphabetically
```

---

## localStorage Strategy

Save to localStorage on every user action:

```javascript
const STATE_KEY = "cardapio-semanal-state";

// State shape
{
  weekPlan: {
    "Segunda-feira": {
      lunch: { protein: "Strogonoff de frango", side: "Arroz branco", veggie: "Brócolis" },
      dinner: null  // or { protein: ..., side: ..., veggie: ... }
    },
    // ... other days
  },
  shoppingChecked: {
    "Creme de leite__Supermercado": true,
    "Kafta pronta (congelada)__Açougue": false
  },
  pantryChecked: {
    "Azeite": true,
    "Manteiga": false
  }
}
```

- Load state on page init
- Persist on every change
- "Novo Cardápio" button clears weekPlan and shoppingChecked but keeps pantryChecked

---

## Technical Requirements

- **Single HTML file** — all CSS and JS inline
- **Mobile-first** — optimized for 375px–430px, works on desktop too
- **External CDNs only**:
  - Google Fonts (Playfair Display, DM Sans)
  - SheetJS (`https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js`)
  - jsPDF (`https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js`)
  - jsPDF-AutoTable (`https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.8.2/jspdf.plugin.autotable.min.js`)
- **No framework** — vanilla JS, CSS custom properties, CSS animations
- **GitHub Pages compatible** — static file, no build step
- **Touch-optimized** — swipe gestures welcome but not required
- **Accessible** — semantic HTML, ARIA labels on interactive elements, good contrast ratios

---

## File Structure

```
cardapio-semanal/
├── index.html                        ← the entire app (single file)
├── data/
│   └── cardapio_template.xlsx        ← meal database (edit this to add/change dishes)
├── CARDAPIO-SEMANAL-SPEC.md          ← this spec file
└── README.md                         ← brief project description
```

### Workflow to update dishes:
1. Edit `data/cardapio_template.xlsx` (on PC, phone, or Google Sheets)
2. `git add . && git commit -m "novo prato" && git push`
3. App automatically reads the updated file on next load

---

## Design Quality Checklist

- [ ] No generic AI aesthetics — this should feel handcrafted and intentional
- [ ] Botanical Garden theme consistently applied throughout
- [ ] Typography hierarchy is clear and elegant
- [ ] Animations are smooth and purposeful (not gratuitous)
- [ ] Empty states are well-designed (no blank voids)
- [ ] Touch targets are comfortable (44px minimum)
- [ ] Bottom sheet modal feels native on mobile
- [ ] Shopping list is scannable and pleasant to use in a store
- [ ] PDFs are clean, professional, and match the app's aesthetic
- [ ] Loading/generating states have feedback (not frozen UI)
- [ ] The app feels like a premium product, not a prototype
- [ ] xlsx loads correctly and parsing handles edge cases (empty rows, extra spaces)
- [ ] Graceful error state if xlsx fails to load
