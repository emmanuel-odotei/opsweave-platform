# Mosaic Ledgerline (Vouchers) — SPEC

- **Owns:** payment vouchers, their approvals, receipts and comments.
- **Publishes:** nothing.
- **Consumes:** `roster.employees`, `roster.organizations`, `roster.offices`, `roster.departments`.
- **Calls:** Requisio through the Router, using a service token.
- **API:** GraphQL subgraph.
- **UI:** the Ledgerline section of `apps/finance-web` (`/finance/ledgerline`).

## Permissions (`LedgerlinePermission`, all `ORG_OFFICE` scoped unless noted)

| Permission | Allows |
|---|---|
| `VIEW_VOUCHER` | see vouchers whose org and office are within scope (requesters always see their own) |
| `CREATE_VOUCHER` | create vouchers for offices within scope |
| `APPROVE_VOUCHER_DEPT` | step-1 approval (also requires being head of the voucher's department) |
| `APPROVE_VOUCHER_FINANCE` | step-2 approval |
| `MARK_VOUCHER_PAID` | mark approved vouchers paid |
| `VIEW_DASHBOARD` | Ledgerline dashboard for vouchers in scope |
| `MANAGE_ACCESS_LEVELS` (NONE) | access levels and members |
| `VIEW_AUDIT` (NONE) | audit log |

## Voucher

### Fields
- **Number:** `VCH-<year>-<5-digit sequence>`
- **Type:** `EXPENSE_REIMBURSEMENT`, `VENDOR_PAYMENT` or `CASH_ADVANCE`
- **Requester:** userId; `departmentCode` copied from the requester at creation
- **Org and office:** default to the requester's own; may be any within `CREATE_VOUCHER` scope
- **Payee:** for reimbursements and advances, the requester by default; for vendor payments, free text with the vendor name taken from the requisition
- **Money:** currency (GHS, USD or EUR), line items (`description`, `category`, `amount`), and `total` (computed)
- **Requisition link:** `reqId` plus a snapshot (`reqTitle`, `reqAmount`, `reqCurrency`) taken at save time. **Required for `VENDOR_PAYMENT`**, optional otherwise.
- **Status:** see the flow below
- **History:** an approval trail, comments, and receipts (`file_attachments`)

### Status flow
```
DRAFT --submit--> SUBMITTED --dept approve--> DEPT_APPROVED --finance approve--> APPROVED --mark paid--> PAID
  ^                  |                            |
  +--return for changes (reason)------------------+
SUBMITTED / DEPT_APPROVED --reject (reason)--> REJECTED   (final)
```

### Rules
- **Submit** needs at least one line item and a total > 0. Vendor payments also need a REQID and at least one receipt or invoice attachment.
- **Step 1:** the approver needs `APPROVE_VOUCHER_DEPT` in scope **and** must be `headUserId` of the voucher's department. If the requester *is* the department head, step 1 is recorded as `AUTO_SKIPPED` and the voucher goes straight to `DEPT_APPROVED`.
- **Step 2:** the approver needs `APPROVE_VOUCHER_FINANCE` in scope and must be neither the requester nor the step-1 approver.
- **Mark paid** records the `paymentReference`, `paidOn` date and payer.
- **Return for changes** sends the voucher back to `DRAFT`, keeps its history, and notifies the requester.
- **Emails and notifications:**
  - on submit → department head
  - on step 1 → finance approvers in scope
  - on approve, reject, return or paid → requester

### Requisition link (federation and service token)
- **Picker query.** `requisitionOptions(search)` calls the Router with a service token (`requisitions:read-approved`) to query Requisio's `approvedRequisitionsForVouchers(orgIds, officeIds, search)`. It passes the caller's `CREATE_VOUCHER` scope and then adds `remainingBalance`. The Requisio call uses `@Retryable` (3 attempts, backoff). If Requisio is unavailable, it returns the error `REQUISIO_UNAVAILABLE` and the form shows "Requisitions are temporarily unavailable."
- **Remaining balance** = `reqAmount − Σ totals` of this requisition's vouchers in `SUBMITTED`, `DEPT_APPROVED`, `APPROVED` or `PAID` status.
- **On save or submit:**
  - re-check the requisition through the same call: it must still be approved, in the same currency, and `total ≤ remainingBalance`
  - store the snapshot
  - drafts don't reserve balance; submitting does
- **Federation.** Ledgerline contributes `Requisition.remainingBalance` and `Requisition.voucherCount`, so Requisio's screens show them with no extra call.

## GraphQL (sketch)
```graphql
type Query {
  vouchers(filter: VoucherFilter, page: PageInput): VoucherPage!
  voucher(id: ID!): Voucher
  requisitionOptions(search: String): [RequisitionOption!]!
  ledgerlineDashboard(range: DateRange!): LedgerlineDashboard!
  ledgerlineAccessLevels: [AccessLevel!]!
  ledgerlineAuditLog(filter: AuditFilter, page: PageInput): AuditPage!
}
type Mutation {
  saveVoucherDraft(input: VoucherInput!): VoucherPayload!
  submitVoucher(id: ID!): VoucherPayload!
  approveVoucher(id: ID!, comment: String): VoucherPayload!
  returnVoucher(id: ID!, reason: String!): VoucherPayload!
  rejectVoucher(id: ID!, reason: String!): VoucherPayload!
  markVoucherPaid(id: ID!, paymentReference: String!, paidOn: Date!): VoucherPayload!
  requestReceiptUpload(voucherId: ID!, fileName: String!, contentType: String!, size: Int!): UploadTicket!
  confirmReceiptUpload(fileId: ID!): VoucherPayload!
  addVoucherComment(id: ID!, body: String!): VoucherPayload!
  saveLedgerlineAccessLevel(input: AccessLevelInput!): AccessLevelPayload!
  setLedgerlineAccessLevelMembers(levelId: ID!, userIds: [ID!]!): AccessLevelPayload!
}
extend type Requisition @key(fields: "reqId") { reqId: ID! @external remainingBalance: Money! voucherCount: Int! }
type Viewer @key(fields: "userId") { userId: ID! ledgerline: LedgerlineViewer }
# LedgerlineViewer { permissions, summary { awaitingMyApproval, myDrafts, approvedUnpaidTotal, paidThisMonth }, tasks, notifications }
```

`VoucherFilter`: `status[]`, `type`, `orgId`, `officeId`, `departmentCode`, `requesterId`, `dateFrom`, `dateTo`, `search`, `view` (`MINE` | `AWAITING_ME` | `ALL_IN_SCOPE`).

## Screens
1. **Dashboard** (`VIEW_DASHBOARD`):
   - KPI cards: awaiting approval, approved but unpaid, paid this month
   - spend by month (bar chart)
   - spend by office and by category
   - ageing of pending vouchers
2. **Vouchers:** tabs for My vouchers, Awaiting my approval and All in scope. TanStack Table with filters and status chips.
3. **New/Edit voucher:**
   - type selector
   - REQID combobox showing title, amount and remaining balance (vendor payments only)
   - line-item editor with a live total
   - receipt drop zone
4. **Voucher detail:**
   - header with status and actions allowed for this viewer
   - approval timeline
   - receipts preview and download
   - comments
5. **Access levels** and **Audit log**, using the same shared components as every app.

## Tests
- Every transition, including permission and separation-of-duty failures.
- Step-1 auto-skip when the requester heads the department.
- Scope filtering across org and office combinations.
- Remaining balance with concurrent submits: a pessimistic lock on (`req_id`) rows in a `requisition_reservations` table.
- `REQUISIO_UNAVAILABLE` handling, using WireMock for the Router.
- Federation entity-resolver test.
