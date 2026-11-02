# MosaicHub Design System

One palette and one component language for every app. Products are told apart by **name and icon only**.

**Style direction:** calm, premium fintech product. Generous whitespace, crisp typography, precise alignment, soft layered shadows, restrained colour. Never noisy.

**Use of colour:**
- Dark tones appear only in hero bands, primary buttons where emphasis is needed, and finance charts.
- Everywhere else: light, cool surfaces.

This file is the source for `packages/ui/tailwind-preset.ts` and the shadcn/ui CSS variables. Update both together.

## Colour tokens

| Token | Light | Dark | Use |
|---|---|---|---|
| `bg` | `#F8FAFC` | `#0B1220` | page background |
| `surface` | `#FFFFFF` | `#111A2E` | cards, tables, dialogs |
| `surface-muted` | `#F1F5F9` | `#16213A` | table header, subtle fills |
| `border` | `#E2E8F0` | `#1E293B` | dividers, inputs |
| `text` | `#0F172A` | `#E2E8F0` | primary text |
| `text-muted` | `#475569` | `#94A3B8` | secondary text |
| `text-subtle` | `#64748B` | `#64748B` | hints, placeholders |
| `primary` | `#2563EB` | `#3B82F6` | primary actions, links, focus ring |
| `primary-hover` | `#1D4ED8` | `#60A5FA` | |
| `primary-soft` | `#EFF6FF` | `#172554` | selected rows, soft badges |
| `secondary` | `#0D9488` | `#14B8A6` | secondary emphasis, positive metrics |
| `secondary-soft` | `#F0FDFA` | `#042F2E` | |
| `accent` | `#6366F1` | `#818CF8` | rare highlights only (new-feature dot, illustrations) |
| `deep` | `#0B1E3F` | `#0B1E3F` | hero bands, strong buttons, chart series 1 |
| `success` / `warning` / `danger` / `info` | `#16A34A` / `#D97706` / `#DC2626` / `#0284C7` | `#22C55E` / `#F59E0B` / `#F87171` / `#38BDF8` | status chips and alerts only |

**Hero gradient:** a soft mesh going from `#0B1E3F` through `#1E3A8A` to `#0D9488` at about 25% opacity over `deep`. A faint mosaic-tile pattern sits on top at 6% opacity. White text.

**Chart series, in order:** `#0B1E3F`, `#2563EB`, `#0D9488`, `#38BDF8`, `#6366F1`, `#94A3B8`. In dark mode, series 1 becomes `#93C5FD`.

**Contrast:** every text/background pair must meet WCAG AA, checked in CI with an axe run in Playwright.

## Typography
- **Inter** for all UI text, with `font-feature-settings: "tnum"` on numbers and tables. **JetBrains Mono** for codes (`VCH-2026-00042`, `MH-FIN-0042`).
- **Scale:**

  | Use | Size / line height | Weight |
  |---|---|---|
  | Display (hero only) | 40/48 | 600 |
  | H1 | 28/36 | 600 |
  | H2 | 22/30 | 600 |
  | H3 | 18/26 | 600 |
  | Body | 15/24 | 400 |
  | Small | 13/20 | 400 |
  | Caption | 12/16 | 500, uppercase, tracking 0.04em |

## Shape, space, depth and motion
- **Spacing:** 4 px grid. Page padding 32 px on desktop, 16 px on mobile. Card padding 24 px. Content width up to 1280 px.
- **Radius:** inputs and buttons 8 px, cards 12 px, dialogs 16 px, chips fully rounded.
- **Shadows:**
  - `sm`: `0 1px 2px rgba(15,23,42,.06)`
  - `md`: `0 4px 12px rgba(15,23,42,.08)`
  - `lg`: `0 12px 32px rgba(15,23,42,.12)`
  - In dark mode, use borders instead of shadows.
- **Motion:** 150–200 ms ease-out for hovers, drawers and dialogs. Respect `prefers-reduced-motion`.

## Components and patterns
- **Buttons:**
  - Primary: `primary` background, white text.
  - Strong: `deep` background, used only in heroes or for final irreversible confirmations such as Publish payroll.
  - Secondary: outline.
  - Ghost.
  - Destructive: `danger`.
- **Status chips** (soft fill + strong text):
  - Draft → slate
  - Submitted / Generated → info
  - Dept approved / Reviewed / Partially paid → indigo
  - Approved / Sent → primary
  - Paid / Received / Completed / Published → success
  - Overdue → warning
  - Rejected / Void / Cancelled → danger
  - Stale → warning outline
- **Tables:** sticky header on `surface-muted`, 48 px rows, zebra striping off, hover on `primary-soft` at 50%, numbers right-aligned in tabular figures, row actions in a kebab menu, server-side pagination.
- **Forms:** label above the field, helper text below, errors in `danger` with an icon, required fields marked with an asterisk. Wizards use a top stepper.
- **Empty states:** a simple line illustration in `text-subtle`, one sentence, one primary action.
- **Timelines** (approvals, invoice history): a vertical line with an avatar or icon per step, actor, action and a relative time with a tooltip giving the absolute time.
- **Dashboards:** KPI cards along the top (label, big number, delta chip), then 2–3 charts in cards. Finance dashboards may use `deep` as the main series and for one dark KPI hero card.
- **Icons:** lucide-react. App marks:
  - Roster → `users`
  - Ledgerline → `receipt`
  - Requisio → `package`
  - Paystream → `wallet`
  - Billora → `file-text`
  - Crewboard → `layout-grid`
  - Portal → the MosaicHub mark
- **MosaicHub mark:** a 2×2 grid of rounded squares in `deep`, `primary`, `secondary` and `#38BDF8`, followed by the wordmark "MosaicHub" in Inter at weight 600. Each product name is set as "Mosaic" in `text-muted` followed by the product name in `text`, e.g. **Mosaic** Ledgerline.
