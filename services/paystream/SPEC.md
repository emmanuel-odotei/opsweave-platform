# Mosaic Paystream (Payroll) — SPEC

- **Owns:** payroll runs (one per department per month), payslip lines, tax and statutory configuration, local copies of compensation and bank details.
- **Publishes:** `paystream.payslips` (`PUBLISHED`, `RESET`).
- **Consumes:** `roster.employees`, `roster.organizations`, `roster.offices`, `roster.departments`, `roster.employee-compensation`.
- **API:** GraphQL subgraph.
- **UI:** the Paystream section of `apps/finance-web` (`/finance/paystream`).

Ordinary employees do **not** use Paystream. They see their payslips in Roster. Only people HR gives Paystream access to, and to whom a Paystream admin grants levels, can open it.

## Permissions (`PaystreamPermission`, scope `DEPARTMENT` unless noted)

| Permission | Allows |
|---|---|
| `VIEW_PAYROLL` | see department tabs and runs in scope |
| `GENERATE_PAYROLL` | generate and regenerate, edit ad-hoc items while `GENERATED` |
| `REVIEW_PAYROLL` | mark reviewed, return to `GENERATED` |
| `APPROVE_PAYROLL` | approve a reviewed run |
| `RESET_PAYROLL` | reset an approved run (reason required) |
| `PUBLISH_PAYROLL` | publish an approved, unpublished run |
| `VIEW_PAYSLIPS` | the all-payslips page for departments in scope |
| `VIEW_DASHBOARD` | the payroll dashboard |
| `MANAGE_TAX_SETTINGS` (NONE) | PAYE bands, SSNIT rates |
| `MANAGE_ACCESS_LEVELS` (NONE), `VIEW_AUDIT` (NONE) | standard |

## Payroll run
- **Identity:** one per (`departmentCode`, `year`, `month`), unique.
- **Fields:**
  - `status`: `GENERATED` | `REVIEWED` | `APPROVED`
  - `published` (boolean), `publishedAt`
  - `revision` (starts at 1, +1 on every reset)
  - `stale` (true after a reset, until regenerated)
  - `generatedBy`, `reviewedBy`, `approvedBy`, timestamps
  - `resetHistory[]` (`at`, `by`, `reason`, `revision`)

### Flow
```
(none) --generate--> GENERATED --mark reviewed--> REVIEWED --approve--> APPROVED --publish--> (published = true)
                       ^  ^                           |                    |
                       |  +------ return (reason) ----+                    |
                       +------------- reset (reason) ----------------------+
                         sets stale = true, published = false, revision + 1, emits RESET
```
- **Publish button** is enabled only when `status = APPROVED && published = false`.
- **Reset** is allowed only when `status = APPROVED`, whether published or not. It **always** emits `RESET` for (department, year, month), so Roster deletes any payslips it holds for that scope.
- **Regenerate** is allowed in `GENERATED`. It clears `stale` and recomputes every line from current compensation, **keeping ad-hoc items**.
- `stale` blocks "mark reviewed".
- **Separation of duties:** the approver must not be the person who generated the run.

### Generation
- **Members:** employees whose local `departmentCode = D` and `status = ACTIVE` at generation time, with `hireDate ≤ last day of the month`.
- **Proration:** someone hired during the month is paid basic salary × (days employed ÷ days in month).
- **Exceptions** are shown in a panel:
  - **No compensation:** the employee is excluded.
  - **No bank details:** the employee is included, with a warning badge.

  Review is blocked while any employee is excluded, unless the reviewer ticks "acknowledge exceptions".
- **Per line:**
  - earnings: basic (prorated), allowances from compensation, ad-hoc earnings (overtime, bonus)
  - deductions: employee SSNIT, PAYE, ad-hoc deductions (e.g. a loan)
  - totals: gross, total deductions, net
  - employer SSNIT, shown as employer cost only

### Tax configuration (illustrative, editable, labelled as such in the UI and README)
- SSNIT: employee 5.5% of basic, employer 13% of basic.
- Taxable income = taxable earnings − employee SSNIT.
- Seeded monthly PAYE bands (GHS):

  | Band | Rate |
  |---|---|
  | first 490 | 0% |
  | next 110 | 5% |
  | next 130 | 10% |
  | next 3,166.67 | 17.5% |
  | next 16,000 | 25% |
  | next 30,520 | 30% |
  | remainder | 35% |

  The UI says: "Illustrative rates for demo purposes; not tax advice."
- Changing the bands affects only runs generated or regenerated afterwards.

### Publish
- One transaction does three things:
  1. set `published = true` and `publishedAt`
  2. write one outbox row per payslip with `PUBLISHED` (payload in EVENTS.md)
  3. write an audit entry
- Republishing after a reset sends the new revision's payslips.

## All-payslips page
- Lists payslips from the **current revision** of runs the user can view.
- Filters: employee name (search), month, year, department tab.
- Columns: employee, number, department, period, gross, deductions, net, run status, published.
- A row opens a preview drawer. "Download PDF" (needs `VIEW_PAYSLIPS`) is generated on request.

## GraphQL (sketch)
```graphql
type Query {
  payrollDepartments: [DepartmentRef!]!                       # tabs in scope
  payrollRun(departmentCode: ID!, year: Int!, month: Int!): PayrollRun
  payrollRuns(departmentCode: ID!, year: Int!): [PayrollRun!]!
  payslips(filter: PayslipFilter!, page: PageInput): PayslipPage!   # name, month, year, departmentCode
  taxSettings: TaxSettings!
  paystreamDashboard(year: Int!): PaystreamDashboard!
}
type Mutation {
  generatePayroll(departmentCode: ID!, year: Int!, month: Int!): PayrollRunPayload!
  regeneratePayroll(runId: ID!): PayrollRunPayload!
  setAdHocItems(lineId: ID!, items: [AdHocItemInput!]!): PayrollRunPayload!
  markPayrollReviewed(runId: ID!, acknowledgeExceptions: Boolean): PayrollRunPayload!
  returnPayroll(runId: ID!, reason: String!): PayrollRunPayload!
  approvePayroll(runId: ID!): PayrollRunPayload!
  resetPayroll(runId: ID!, reason: String!): PayrollRunPayload!
  publishPayroll(runId: ID!): PayrollRunPayload!
  saveTaxSettings(input: TaxSettingsInput!): TaxSettingsPayload!
}
type Viewer @key(fields: "userId") { userId: ID! paystream: PaystreamViewer }
# summary { awaitingMyReview, awaitingMyApproval, approvedUnpublished, netPayThisMonth }
```

## Screens
1. **Payroll:**
   - a department tab bar (only departments in scope), then a month/year picker
   - **run panel:**
     - status stepper (Generated → Reviewed → Approved, plus a Published badge)
     - action buttons according to permissions and state
     - stale banner after a reset
     - exceptions panel
     - lines table, with a drawer for each line's breakdown and ad-hoc items
     - reset dialog that requires a reason
2. **All payslips:** the filters above, table, preview drawer.
3. **Dashboard:**
   - monthly payroll cost per department (stacked bars; dark tones allowed)
   - gross / deductions / net split
   - employer SSNIT cost
   - headcount paid
4. **Tax settings** (band editor with an example calculation), **Access levels**, **Audit**.

## Tests
- Every transition and permission check, including that the approver ≠ generator.
- Reset → regenerate → approve → republish emits RESET then PUBLISHED with the new revision.
- The Publish button rule, tested in both the API and the UI.
- Proration and PAYE band maths (parameterised tests).
- Exceptions blocking review.
- Contract tests for both event shapes.
