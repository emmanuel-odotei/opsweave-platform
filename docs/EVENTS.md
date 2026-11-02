# MosaicHub — Kafka Event Catalogue

Behaviour rules (outbox, idempotency, DLT, replay) are in `docs/ARCHITECTURE.md` §5. This file defines **what** flows. Each topic has a JSON Schema at `/contracts/<topic>.schema.json`, and the schemas are the executable source of truth. When an event changes, update the schema, this file, and both producer and consumer tests in the same commit.

## Envelope (every message)

```json
{
  "eventId": "7b0c…-uuid",
  "action": "CREATED",
  "schemaVersion": 1,
  "occurredAt": "2026-03-01T09:15:22Z",
  "correlationId": "c-…",
  "producer": "roster",
  "key": "<entity id, also the Kafka message key>",
  "payload": { }
}
```

- `action` describes what happened. Consumers branch on it, but **`payload` always carries the full current record**. The only exception is `paystream.payslips` with `action: RESET`, whose payload is the reset scope.
- Adding an optional field doesn't need a version bump. Any breaking change bumps `schemaVersion`, and consumers must handle both versions until every producer has migrated.

## Topics

| Topic | Producer | Cleanup | Key | Actions | Consumers |
|---|---|---|---|---|---|
| `roster.organizations` | Roster | compact | orgId | CREATED, UPDATED, DEACTIVATED | all services |
| `roster.offices` | Roster | compact | officeId | CREATED, UPDATED, DEACTIVATED | all services |
| `roster.departments` | Roster | compact | departmentCode | CREATED, UPDATED, DEACTIVATED | all services |
| `roster.employees` | Roster | compact | userId | CREATED, UPDATED, DEACTIVATED, REACTIVATED | all services (including Auth) |
| `roster.employee-compensation` | Roster | compact | userId | CREATED, UPDATED | Paystream only |
| `paystream.payslips` | Paystream | delete, 30-day retention | `<departmentCode>:<year>-<month>:<userId>` for PUBLISHED; `<departmentCode>:<year>-<month>` for RESET | PUBLISHED, RESET | Roster only |
| `crewboard.clients` | Crewboard | compact | clientId | CREATED, UPDATED, ARCHIVED | Billora |
| `crewboard.projects` | Crewboard | compact | projectId | CREATED, UPDATED, ARCHIVED | Billora |
| `billora.invoices` | Billora | compact | invoiceId | CREATED, UPDATED, SENT, PAYMENT_RECORDED, PAID, VOIDED | Crewboard |

Every topic also has a `<topic>.dlt` (delete policy, 14-day retention).

**Partitioning note.** For `paystream.payslips`, each department's run is published from one transaction. A RESET's key would hash to a different partition from the PUBLISHED messages, so their order isn't guaranteed. Roster therefore handles a RESET by deleting **by scope** (department + year + month) and ignores a PUBLISHED message whose `publishedAt` is older than that scope's last reset. Both events carry `publishedAt` / `resetAt` for this.

## Payloads

### `roster.organizations`
`{ orgId, name, code, country, status }`

### `roster.offices`
`{ officeId, orgId, name, city, status }`

### `roster.departments`
`{ departmentCode, name, headUserId | null, status }`

### `roster.employees`
```json
{
  "userId": "uuid",
  "employeeNumber": "MH-FIN-0042",
  "firstName": "Kofi", "lastName": "Boateng",
  "email": "kofi.boateng@mosaichub.com",
  "jobTitle": "Head of Finance",
  "orgId": "uuid", "officeId": "uuid", "departmentCode": "FIN",
  "managerUserId": "uuid | null",
  "hireDate": "2024-02-01",
  "status": "ACTIVE",
  "appAccess": [ { "app": "LEDGERLINE", "admin": false }, { "app": "REQUISIO", "admin": true } ],
  "hasAvatar": true
}
```
- Never include salary, bank details or personal contact details on this topic.
- On a department transfer, `employeeNumber` and `departmentCode` both change (see the Roster SPEC).

### `roster.employee-compensation`
```json
{
  "userId": "uuid",
  "currency": "GHS",
  "basicSalary": "8500.0000",
  "allowances": [ { "name": "Transport", "amount": "600.0000", "taxable": true } ],
  "effectiveFrom": "2026-03-01",
  "bank": { "bankName": "…", "branch": "…", "accountName": "…", "accountNumber": "…" }
}
```
- `bank` is `null` until the employee fills in their profile.
- This topic has a single consumer, Paystream. It isn't encrypted in transit in this local demo; the README explains that production would add TLS and Kafka ACLs restricting it to Paystream.

### `paystream.payslips` — `PUBLISHED` (one message per employee)
```json
{
  "departmentCode": "FIN", "year": 2026, "month": 3,
  "userId": "uuid", "employeeNumber": "MH-FIN-0042",
  "runId": "uuid", "publishedAt": "…",
  "currency": "GHS",
  "earnings":   [ { "name": "Basic salary", "amount": "8500.00" } ],
  "deductions": [ { "name": "SSNIT (5.5%)", "amount": "467.50" }, { "name": "PAYE", "amount": "1294.12" } ],
  "grossPay": "9100.00", "totalDeductions": "1761.62", "netPay": "7338.38",
  "bankAccountMasked": "****4821"
}
```

### `paystream.payslips` — `RESET` (one message per department run)
`{ "departmentCode": "FIN", "year": 2026, "month": 3, "resetAt": "…", "reason": "Wrong overtime figures" }`

Roster deletes every stored payslip that matches the **department code stored on the payslip** plus year and month.

### `crewboard.clients`
`{ clientId, name, industry, country, billingEmail, defaultCurrency, status }`

### `crewboard.projects`
`{ projectId, clientId, code, name, status, startDate, endDate, projectLeadUserId }`

### `billora.invoices`
`{ invoiceId, number, clientId, projectId | null, currency, total, amountPaid, status, issueDate, dueDate }`
