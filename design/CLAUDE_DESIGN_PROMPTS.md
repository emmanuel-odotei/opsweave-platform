# Claude Design Prompts

**How to use:** run these in Claude Design in order. Prompt 0 creates the shared design system; each later prompt builds one app on top of it.

**Before each prompt,** paste or attach `design/DESIGN_SYSTEM.md`.

**After each prompt:**
- iterate until you're happy with the result
- export the screens (light and dark) to `design/<app>/`, or hand them off to Claude Code
- note any token changes you made back in `DESIGN_SYSTEM.md`

**Usage note for every prompt:** everything is fictional. Use realistic Ghanaian and Nigerian names, GHS amounts and dates in 2026. Never use real company names or logos.

---

## 0 — Design system and shell
> Create a design system and app shell for **Opsweave**, a fictional company's internal business platform. Follow the attached DESIGN_SYSTEM.md exactly for colours, type, radius, shadows and components. The feel is a calm, premium fintech product: lots of whitespace, crisp Inter typography, precise alignment, soft depth, restrained cool colours. Dark tones appear only in hero bands, strong buttons and finance charts.
> Deliver:
> 1. **Component sheet:** buttons (primary, strong, secondary, ghost, destructive), inputs, select, combobox, date picker, checkbox, switch, status chips for every status listed, table with pagination, KPI card, chart card, empty state, approval timeline, toast, dialog, drawer, tabs, stepper.
> 2. **App shell:** collapsible left sidebar, top bar with global search (⌘K), app switcher, notifications bell with dropdown, and profile menu (with a theme toggle).
> 3. **The Opsweave mark** (three interlaced strands, as described in DESIGN_SYSTEM.md) and wordmark, plus the "Opsweave + Product" naming lockup.
>
> Show both light and dark modes.

## 1 — Opsweave Portal
> Using the Opsweave design system, design the **Opsweave Portal**:
> - **Login:** split layout with a deep hero panel (subtle woven-thread pattern and a short tagline), a form card, the forgot-password link, and an error state for a locked account.
> - **Set password:** token is valid; live password-rule checklist.
> - **Forgot password:** confirmation state.
> - **Home**, top to bottom:
>   1. greeting hero band
>   2. app launcher tiles with badge counts (Roster, Finance & Ops with sub-links, Billora, Crewboard)
>   3. summary widget grid
>   4. "My tasks" list with app filter chips
>   5. announcements
> - **Notifications dropdown** open.
> - **Global search** palette open, showing employee results.
> - **The same home screen for a user with only Roster access.**

## 2 — Opsweave Roster
> Using the Opsweave design system, design **Opsweave Roster**, the employee and organisation-structure app for HR, plus each employee's self-service profile:
> - **Employees list:** search, filter chips, status badges, add button.
> - **Add employee wizard:** Basics, then Placement (org, office, department, manager), then App access, with a per-app ADMIN toggle. The generated email `firstname.lastname@opsweave.com` and number `OW-FIN-0042` preview live.
> - **Employee detail** with tabs: Overview, Employment (with transfer history and a scheduled transfer banner), App access, Compensation (masked bank details with a reveal for permitted users), Audit.
> - **Organisation structure:** organisations, then office cards, then a departments table (code, head, headcount).
> - **Access levels:** editor with permission checkboxes grouped by area and organisation/office scope pickers; members list.
> - **My profile:** personal details, bank details form, avatar upload, change password.
> - **My payslips:** grouped by year, with download buttons, plus an empty state.
> - **"You don't have access yet"** page.

## 3 — Opsweave Finance & Ops (Ledgerline, Requisio, Paystream in one app)
> Using the Opsweave design system, design the **Finance & Ops** app. The sidebar groups three sections: **Opsweave Ledgerline** (payment vouchers), **Opsweave Requisio** (procurement) and **Opsweave Paystream** (payroll). Data-dense but calm. Finance charts may use the deep navy series and one dark KPI hero card.
>
> **Ledgerline:**
> - Dashboard: KPI cards, spend by month, spend by office, pending ageing.
> - Vouchers list with tabs: Mine, Awaiting my approval, All.
> - Voucher form: type selector; REQID combobox showing title, amount and remaining balance; line items with a live total; receipt drop zone; an "Requisitions temporarily unavailable" inline state.
> - Voucher detail: two-step approval timeline, actions, receipts, comments.
>
> **Requisio:**
> - Dashboard: budget vs committed per department, requisitions by status.
> - Requisition form: live budget indicator, and an over-budget warning banner.
> - Requisition detail: linked purchase orders and a "voucher usage" card.
> - Purchase-order detail: receive-goods dialog.
> - Vendors and budgets.
>
> **Paystream:**
> - Payroll page: department tabs, month picker, a status stepper (Generated, Reviewed, Approved) with a separate Published badge, and action buttons, with the Publish button disabled until approved. Also show: a stale banner after a reset, an exceptions panel, a lines table, a line breakdown drawer, and a reset dialog with a required reason.
> - All-payslips page: filters for employee name, month and year.
> - Payroll dashboard.
> - Tax settings editor labelled "illustrative".

## 4 — Opsweave Billora
> Using the Opsweave design system, design **Opsweave Billora**, the client invoicing app:
> - **Dashboard:** receivables ageing buckets, invoiced vs collected by month, top clients, agent collection rate.
> - **Invoices list:** My assigned / All tabs, with overdue rows highlighted subtly.
> - **Invoice editor:** split view, with the form on the left and a live document-style invoice preview on the right; client and project combobox; line items with tax.
> - **Invoice detail:** document view, action bar (send, record payment, void, assign agent), payments list, timeline of sent / reminder / payment events.
> - **Record-payment dialog** and **exchange-rates page.**

## 5 — Opsweave Crewboard
> Using the Opsweave design system, design **Opsweave Crewboard**, the app for clients, projects and employee assignments:
> - **Dashboard:** utilisation over time, billable vs non-billable, projects ending soon, revenue per client.
> - **Clients:** list and detail.
> - **Projects:** list and detail, with a team table, a horizontal allocation timeline per person, and live invoiced/paid totals.
> - **Assign-employee drawer:** a people picker showing current allocation % and skills, plus an over-allocation warning dialog.
> - **Capacity heatmap:** people × weeks, coloured by allocation.
> - **Bench view** with skill filters.
> - **My assignments** and **My skills.**
