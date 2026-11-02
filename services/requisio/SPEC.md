# Mosaic Requisio (Procurement) — SPEC

- **Owns:** vendors, department budgets, purchase requisitions, purchase orders, goods receipts, procurement documents.
- **Publishes:** nothing.
- **Consumes:** `roster.employees`, `roster.organizations`, `roster.offices`, `roster.departments`.
- **API:** GraphQL subgraph.
- **UI:** the Requisio section of `apps/finance-web` (`/finance/requisio`).

## Permissions (`RequisioPermission`)
Scope kind `ORG_OFFICE_DEPARTMENT` means the access level's org, office **and** department access all apply.

| Permission | Scope | Allows |
|---|---|---|
| `VIEW_REQUISITION` | ORG_OFFICE_DEPARTMENT | see requisitions in scope (requesters always see their own) |
| `CREATE_REQUISITION` | ORG_OFFICE_DEPARTMENT | raise requisitions |
| `APPROVE_REQUISITION` | ORG_OFFICE_DEPARTMENT | approve, return or reject (also requires being head of the requisition's department) |
| `MANAGE_PURCHASE_ORDERS` | ORG_OFFICE_DEPARTMENT | create, issue and cancel POs |
| `RECEIVE_GOODS` | ORG_OFFICE | record goods receipts |
| `MANAGE_VENDORS` | NONE | vendor directory |
| `MANAGE_BUDGETS` | DEPARTMENT | set yearly department budgets and the block/warn setting |
| `VIEW_DASHBOARD` | ORG_OFFICE_DEPARTMENT | procurement dashboard |
| `MANAGE_ACCESS_LEVELS` | NONE | access levels |
| `VIEW_AUDIT` | NONE | audit log |

## Entities and rules

### Vendor
- Fields: name, category, contact name, email, phone, address, tax ID, status (`ACTIVE` / `INACTIVE`).
- Inactive vendors can't receive new POs.

### Budget
- One per (`departmentCode`, `year`) in GHS. Each has an `overBudgetPolicy` of `WARN` (default) or `BLOCK`.
- `committed` = the GHS-equivalent sum of `APPROVED` requisitions for that department and year. GHS conversion uses a simple rates table seeded for USD and EUR and editable by `MANAGE_BUDGETS`.

### Requisition (`REQ-<year>-<5 digits>`; this is the **REQID**)
- **Fields:**
  - `title`, `justification`
  - department, org and office (requester's by default)
  - currency
  - items (`description`, `quantity`, `unitEstimate`) and `estimatedTotal`
  - `preferredVendorId` (optional), `neededBy`
  - attachments (quotes)
- **Status flow:**
  ```
  DRAFT --submit--> SUBMITTED --approve--> APPROVED
    ^                  |--return (reason)--> DRAFT
    |                  |--reject (reason)--> REJECTED (final)
  DRAFT / SUBMITTED --cancel--> CANCELLED (requester)
  ```
- **Budget check:** on submit and approve, compute `committed + this` against the budget.
  - Under `WARN`, show a banner and record `overBudget = true`.
  - Under `BLOCK`, refuse with `BUDGET_EXCEEDED`.
  - A missing budget counts as a warning ("No budget set for FIN 2026").
- **Approver:** needs `APPROVE_REQUISITION` in scope and must be the department head; never the requester. If the requester *is* the head, approval falls to anyone holding `APPROVE_REQUISITION` for that department, other than the requester.
- **Notifications:** on submit → department head; on decision → requester; on approval → holders of `MANAGE_PURCHASE_ORDERS` for that department.

### Purchase order (`PO-<year>-<5 digits>`)
- Created from an `APPROVED` requisition: vendor required, lines copied from the requisition with editable unit prices.
- **Status flow:** `DRAFT → ISSUED → PARTIALLY_RECEIVED → RECEIVED`. `CANCELLED` is possible from `DRAFT` or `ISSUED`.
- **Issue:** emails the vendor contact (to Mailpit) with the PO PDF attached. The PDF is generated for that email and not stored.
- **Goods receipt:** received quantity per line, an optional delivery-note upload, and a received date. Status updates automatically.
- `GET` PO PDF downloads are generated on request.

### For Ledgerline (service token only)
- `approvedRequisitionsForVouchers(orgIds, officeIds, search)` requires `typ: service` and scope `requisitions:read-approved`. It returns `APPROVED` requisitions in those orgs and offices: `reqId`, `title`, `estimatedTotal`, `currency`, `departmentCode`, `orgId`, `officeId`, and `vendorName` (from the latest PO, otherwise the preferred vendor).
- `requisitionForVoucher(reqId)` uses the same guard, for re-validation when a voucher is saved.

## GraphQL (sketch)
```graphql
type Requisition @key(fields: "reqId") {
  reqId: ID!            # "REQ-2026-00042"
  title: String!  status: RequisitionStatus!  estimatedTotal: Money!
  department: DepartmentRef!  office: OfficeRef!  requester: EmployeeRef!
  items: [RequisitionItem!]!  purchaseOrders: [PurchaseOrder!]!  overBudget: Boolean!
}
type Query {
  requisitions(filter: RequisitionFilter, page: PageInput): RequisitionPage!
  requisition(reqId: ID!): Requisition
  purchaseOrders(filter: POFilter, page: PageInput): POPage!
  vendors(search: String, page: PageInput): VendorPage!
  budgets(year: Int!): [Budget!]!
  requisioDashboard(year: Int!): RequisioDashboard!
  approvedRequisitionsForVouchers(orgIds: [ID!]!, officeIds: [ID!]!, search: String): [Requisition!]!  # service only
  requisitionForVoucher(reqId: ID!): Requisition                                                     # service only
}
type Viewer @key(fields: "userId") { userId: ID! requisio: RequisioViewer }
```
Mutations follow the convention: `saveRequisitionDraft`, `submitRequisition`, `approveRequisition`, `returnRequisition`, `rejectRequisition`, `cancelRequisition`, `createPurchaseOrder`, `issuePurchaseOrder`, `cancelPurchaseOrder`, `recordGoodsReceipt`, `saveVendor`, `saveBudget`, upload ticket and confirm, and access-level mutations.

## Screens
1. **Dashboard:**
   - budget vs committed per department (bar chart)
   - requisitions by status
   - average approval time
   - top vendors by PO value
2. **Requisitions:** tabs for Mine, Awaiting my approval and All in scope.
3. **Requisition form:** item editor, quote uploads, live budget indicator.
4. **Requisition detail:**
   - timeline
   - linked POs
   - **voucher usage from Ledgerline** (`remainingBalance`, `voucherCount` via federation)
5. **Purchase orders:** list and detail, with issue, receive and PDF actions.
6. **Vendors**, **Budgets** (year picker, policy toggle), **Access levels**, **Audit**.

## Tests
- Budget WARN vs BLOCK, including currency conversion.
- Approver rules, including the requester-is-head case.
- PO receiving: partial receipts adding up to the full quantity.
- Service-only queries reject user tokens.
- Scope filtering with department access.
