# Mosaic Billora (Invoicing) — SPEC

- **Owns:** client invoices, billing-agent assignments, payments, reminders, exchange rates.
- **Publishes:** `billora.invoices`.
- **Consumes:** `roster.employees`, `roster.organizations`, `roster.offices`, `roster.departments`, `crewboard.clients`, `crewboard.projects`.
- **API:** GraphQL subgraph.
- **UI:** `apps/billora-web` (`/billora`).

## Permissions (`BilloraPermission`, scope `ORG_OFFICE` unless noted)
Scope uses the invoice's issuing org and office. A billing agent always sees the invoices assigned to them.

| Permission | Allows |
|---|---|
| `VIEW_INVOICE` | see invoices in scope |
| `CREATE_INVOICE` | create and edit drafts |
| `SEND_INVOICE` | send an invoice (emails the client) |
| `RECORD_PAYMENT` | record payments |
| `VOID_INVOICE` | void an invoice that has no payments |
| `ASSIGN_AGENT` | assign or reassign billing agents |
| `VIEW_DASHBOARD` | receivables dashboard |
| `MANAGE_EXCHANGE_RATES` (NONE) | edit rates |
| `MANAGE_ACCESS_LEVELS` (NONE), `VIEW_AUDIT` (NONE) | standard |

A **billing agent** is any employee who holds `RECORD_PAYMENT` for the invoice's org and office. `ASSIGN_AGENT` users choose agents from that list.

## Invoice (`INV-<year>-<5 digits>`)
- **Fields:**
  - client (required, from the local copy of `crewboard.clients`), project (optional, must belong to that client)
  - issuing org and office
  - currency (GHS, USD or EUR; defaults to the client's `defaultCurrency`)
  - line items (`description`, `quantity`, `unitPrice`, `taxRatePct`)
  - `subtotal`, `taxTotal`, `total`, `amountPaid`, `balance`
  - `issueDate`, `paymentTermsDays` (default 30), `dueDate`
  - `notes`, `agentUserId`
- **Status flow:**
  ```
  DRAFT --send--> SENT --payment (partial)--> PARTIALLY_PAID --payment (rest)--> PAID
  SENT / PARTIALLY_PAID --(daily job: past due, balance > 0)--> OVERDUE --payment--> PARTIALLY_PAID / PAID
  DRAFT / SENT (no payments) --void (reason)--> VOID
  ```
- **Send:**
  - fixes the number and issue date
  - emails the client's `billingEmail` (to Mailpit) with the invoice PDF attached, generated for that email
  - publishes `SENT`
- **Payments:** amount ≤ balance; fields `date`, `method` (BANK_TRANSFER, MOBILE_MONEY, CHEQUE, CARD), `reference`. Each payment publishes `PAYMENT_RECORDED`, or `PAID` when the balance reaches 0.
- **Overdue job** (daily at 07:00 Africa/Accra):
  - marks invoices `OVERDUE`
  - sends reminders 3 days before the due date and 1, 7 and 14 days after it, to the client and in-app to the agent
  - each reminder is recorded on the invoice timeline
- **Archived clients:** an archived client can't receive new invoices.
- **Exchange rates:** base currency GHS. Manual rates with an effective date. Used for dashboard totals and for `Project.invoicedTotal`, which is reported in GHS.

## Federation
- `Invoice.client: Client!` and `Invoice.project: Project` are returned as entity references, and **Crewboard** resolves them live. The local copies serve validation, PDFs and the case where Crewboard is down (show the names from the local copy and drop live-only fields).
- Billora contributes `Project.invoicedTotal: Money!` and `Project.paidTotal: Money!` for Crewboard's screens.

## GraphQL (sketch)
```graphql
type Query {
  invoices(filter: InvoiceFilter, page: PageInput): InvoicePage!   # status, clientId, projectId, agentId, overdue, dateRange, search, view: MINE | ALL_IN_SCOPE
  invoice(id: ID!): Invoice
  exchangeRates: [ExchangeRate!]!
  billoraDashboard(range: DateRange!): BilloraDashboard!
}
type Mutation {
  saveInvoiceDraft(input: InvoiceInput!): InvoicePayload!
  sendInvoice(id: ID!): InvoicePayload!
  recordPayment(invoiceId: ID!, input: PaymentInput!): InvoicePayload!
  voidInvoice(id: ID!, reason: String!): InvoicePayload!
  assignInvoiceAgent(id: ID!, agentUserId: ID!): InvoicePayload!
  saveExchangeRate(input: ExchangeRateInput!): ExchangeRatePayload!
}
extend type Client @key(fields: "id") { id: ID! @external }
extend type Project @key(fields: "id") { id: ID! @external invoicedTotal: Money! paidTotal: Money! }
type Viewer @key(fields: "userId") { userId: ID! billora: BilloraViewer }
# summary { myOverdueCount, myOutstandingTotal, sentThisMonth, collectedThisMonth }
```

## Screens
1. **Dashboard:**
   - outstanding by ageing bucket (0–30, 31–60, 61–90, 90+)
   - revenue invoiced vs collected per month (GHS)
   - top clients
   - collection rate per agent
2. **Invoices:** tabs for My assigned and All in scope; overdue highlighted.
3. **Invoice editor:** client and project combobox, line items with tax, live totals, a document-style preview pane.
4. **Invoice detail:** a document view with actions (send, record payment, void, assign agent), payment list and timeline (sent, reminders, payments).
5. **Exchange rates**, **Access levels**, **Audit**.

## Tests
- Totals maths with tax and rounding (half-even, 2 decimal places on display).
- Every transition, including the overdue job (with a controllable clock) and reminder schedule.
- Payment over-balance rejection.
- Archived-client guard.
- Contract tests for `billora.invoices`.
- Federation: `Project.invoicedTotal` resolver, plus fallback when Crewboard is down.
