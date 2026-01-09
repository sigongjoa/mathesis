# Common Design System & Style Guide

## 1. Design Tokens

### 1.1 Colors
- **Primary**: `#4F46E5` (Indigo 600)
- **Secondary**: `#10B981` (Emerald 500)
- **Neutral**: Slate Scale (`#F8FAFC` to `#0F172A`)
- **Semantic**:
    - Success: `#22C55E`
    - Warning: `#F59E0B`
    - Error: `#EF4444`
    - Info: `#3B82F6`

### 1.2 Typography
- **Font Family**: `Inter`, sans-serif
- **Scale**:
    - H1: 32px / 700 / 1.2
    - H2: 24px / 600 / 1.3
    - H3: 20px / 600 / 1.4
    - Body: 16px / 400 / 1.5
    - Small: 14px / 400 / 1.5

### 1.3 Spacing
- Base unit: `4px`
- Scale: 4, 8, 12, 16, 24, 32, 48, 64px

---

## 2. Component Guidelines

### 2.1 Buttons
- **Primary**: Solid background, white text.
- **Secondary**: Outline border, primary text.
- **Ghost**: Transparent background, useful for icon buttons.

### 2.2 Cards
- White background, `rounded-lg`, generic shadow (`shadow-md`).
- Padding: `p-6` (24px).

### 2.3 Forms
- Inputs: `border-slate-300`, `focus:ring-2 focus:ring-primary`.
- Validation: Inline error messages below input.

---

## 3. Iconography
- Library: **Heroicons** (Outline for UI, Solid for active states).
- Size: Standard `w-6 h-6` (24px).
