# Opsweave Platform — Portfolio Plan

## 1. Purpose
Emmanuel's professional projects live in private client repositories. This monorepo is a **clean-room reimplementation** of the *patterns* behind that work, set inside a fictional company called **Opsweave**:
- RBAC data filtering
- multi-step approvals
- audit logging
- event-driven microservices on Kafka
- GraphQL federation

It is never hosted. Recruiters run it locally with `make up` (which wraps `docker compose --profile all up`).

Every README carries this statement:
> This is a personal reimplementation, written from scratch with fictional data, of the kinds of systems I build professionally. It contains no code, data or business rules from any employer or client.

The AI chatbot (CVA) is out of scope and will be a later project.

## 2. The company
- **Opsweave** is a fictional holding company. Employee emails follow `firstname.lastname@opsweave.com`; a duplicate becomes `firstname.lastname2@opsweave.com`, then `3`, and so on.
- **Organizations:** Opsweave Ghana (offices: Accra HQ, Kumasi) and Opsweave Nigeria (office: Lagos).
- **Departments** are company-wide, created by HR, each with a unique code: `FIN` Finance, `OPS` Operations, `HR` People, `ENG` Engineering, `PRC` Procurement, `SAL` Sales.
- **Name check:** a web search in November 2026 found no company or product called "Opsweave". "MosaicHub" was dropped because a real business-software company had used it. Before the repo goes public, Emmanuel also searches "Opsweave" in the WIPO Global Brand Database; if a conflict appears, rename everywhere in one commit.

## 3. Product map

| Product | Folder | Stack | API | Frontend |
|---|---|---|---|---|
| Opsweave Auth | `services/auth` | Spring Boot | REST | login & password pages inside Portal |
| Opsweave Roster (employees) | `services/roster` | Spring Boot | REST | `apps/roster-web` |
| Opsweave Ledgerline (vouchers) | `services/ledgerline` | Spring Boot | GraphQL subgraph | `apps/finance-web` |
| Opsweave Requisio (procurement) | `services/requisio` | Spring Boot | GraphQL subgraph | `apps/finance-web` |
| Opsweave Paystream (payroll) | `services/paystream` | Spring Boot | GraphQL subgraph | `apps/finance-web` |
| Opsweave Billora (invoicing) | `services/billora` | Spring Boot | GraphQL subgraph | `apps/billora-web` |
| Opsweave Crewboard (resourcing) | `services/crewboard` | Node + TypeScript + Express | GraphQL subgraph | `apps/crewboard-web` |
| Gateway | `services/gateway` | Spring Cloud Gateway | edge for everything | — |
| Apollo Router | `infra/router` | Apollo Router | federates the 5 subgraphs | — |
| Opsweave Portal | `apps/portal` | React | uses all of the above | `apps/portal` |

Where the rest lives:
- Platform-wide rules: `docs/ARCHITECTURE.md`
- Kafka contracts: `docs/EVENTS.md`
- Look and feel: `design/DESIGN_SYSTEM.md`
- Teaching rules: `docs/TEACHING.md`

## 4. Milestones and tickets
Pace is 2 hours per weekday, about 10 hours a week, for about 21 weeks. Each ticket takes 1–3 sessions. The `[FE: …]` tag says who writes the frontend code in that milestone (defined in `docs/TEACHING.md`). Tick a ticket only when its tests pass.

### M0 — Platform skeleton (week 1)
- [ ] **M0.1** Monorepo layout:
  - Maven parent POM plus a `libs/java/platform-common` module
  - pnpm workspace for `apps/*` and `packages/*`
  - `.editorconfig`, `.gitignore`, Conventional Commits
- [ ] **M0.2** Resolve the versions, pin them, and record them in `docs/VERSIONS.md` (rules in CLAUDE.md).
- [ ] **M0.3** `docker-compose.yml` with the infrastructure containers and the Compose profiles from ARCHITECTURE §10:
  - Postgres with an init script that creates one database and one user per service
  - Kafka in KRaft mode, plus Kafka UI
  - MinIO, with an init job that creates the buckets
  - Mailpit
- [ ] **M0.4** Gateway skeleton that routes a health check. `Makefile` targets: `up`, `down`, `dev-<service>`, `seed`, `supergraph`, `reset-data`.
- [ ] **M0.5** GitHub Actions:
  - per-service build and test
  - frontend lint, typecheck and test
  - Docker image build

  Root README with CI badges and the clean-room statement.
- [ ] **M0.6** `/contracts` folder with the event-envelope JSON Schema and schema-validation test helpers in Java and TypeScript.

### M1 — Auth, Roster and the Portal shell (weeks 2–5) `[FE: Claude builds and teaches]`
- [ ] **M1.1** Build the shared pieces in `platform-common` (ARCHITECTURE §4–§6):
  - Problem Details error handler
  - correlation-ID filter
  - outbox table and relay
  - idempotent consumer base
  - event envelope
  - access-control module
- [ ] **M1.2** Roster: organizations, offices and departments (CRUD) and their compacted topics.
- [ ] **M1.3** Roster: employees CRUD, company-email generation, employee numbers, app access with ADMIN flags, `roster.employees` events.
- [ ] **M1.4** Roster: compensation (HR only), self-service profile with bank details, `roster.employee-compensation`, field encryption.
- [ ] **M1.5** Auth:
  - consume employees
  - invite → set password
  - login and refresh-token rotation
  - lockout and forgot password
  - JWKS endpoint
  - service tokens
- [ ] **M1.6** Gateway: JWT validation, `apps`-claim checks, all routes.
- [ ] **M1.7** Roster: access levels, audit log, announcements, directory search, avatars in MinIO, `GET /me/summary`.
- [ ] **M1.8** Frontend foundations:
  - `packages/ui`: Tailwind preset built from DESIGN_SYSTEM, plus shadcn/ui
  - `packages/auth`
  - `packages/api`
- [ ] **M1.9** Portal: login, set password, forgot password, app launcher, profile menu, theme toggle.
- [ ] **M1.10** Roster web:
  - employee list, detail and create screens
  - organization-structure screens
  - access levels
  - My profile
  - My payslips (empty state for now)
- [ ] **M1.11** Seed script and demo users. First Playwright test: HR creates an employee → invite arrives in Mailpit → set password → log in.

### M2 — Ledgerline and the Finance & Ops app (weeks 6–8) `[FE: co-write]`
- [ ] **M2.1** Apollo Router in Compose; `make supergraph` with Rover. Ledgerline subgraph skeleton with federation and the `Viewer` entity.
- [ ] **M2.2** Ledgerline: consumers for employees, organizations, offices and departments; the seeded ADMIN level; the access-level API.
- [ ] **M2.3** Vouchers:
  - draft and submit
  - two-step approval and reject
  - mark paid
  - audit log and emails
- [ ] **M2.4** Receipt upload and download through presigned URLs.
- [ ] **M2.5** Finance web shell: a section per app and the "no access yet" state. Then the Ledgerline screens.
- [ ] **M2.6** Make the repo public with a "work in progress" README and the first screenshots.

### M3 — Requisio (weeks 9–10) `[FE: co-write]`
- [ ] **M3.1** Requisio subgraph: consumers, ADMIN level, vendors, budgets.
- [ ] **M3.2** Requisitions with approval, purchase orders, goods receipt, PO PDF, documents.
- [ ] **M3.3** Requisitions in vouchers:
  - service tokens and `approvedRequisitionsForVouchers` in Requisio
  - Ledgerline `requisitionOptions` and federated `Requisition.remainingBalance`
  - REQID validation when a voucher is saved
- [ ] **M3.4** Requisio screens, and the REQID picker in the voucher form.

### M4 — Paystream (weeks 11–13) `[FE: co-write, Emmanuel drives]`
- [ ] **M4.1** Paystream subgraph: consumers (including compensation), ADMIN level, tax-band and SSNIT configuration.
- [ ] **M4.2** Payroll runs per department per month: generate, regenerate, submit for review, return, approve, reset, publish.
- [ ] **M4.3** `paystream.payslips` PUBLISHED and RESET events. Roster stores and deletes payslips accordingly and generates the payslip PDF.
- [ ] **M4.4** Payroll page with department tabs, the all-payslips page, and "My payslips" in Roster.

### M5 — Crewboard (weeks 14–16) `[FE: Emmanuel leads, Claude reviews]`
- [ ] **M5.1** Express + TypeScript project:
  - Prisma
  - JWT verification with `jose`
  - Kafka client and outbox
  - Apollo subgraph
- [ ] **M5.2** Clients, projects, assignments and skills; publish clients and projects.
- [ ] **M5.3** Allocation rules, bench view, timeline, project revenue from `billora.invoices`.
- [ ] **M5.4** Crewboard web.

### M6 — Billora (weeks 17–19) `[FE: Emmanuel leads, Claude reviews]`
- [ ] **M6.1** Billora subgraph: consumers for employees, clients and projects; ADMIN level; exchange rates.
- [ ] **M6.2** Invoicing:
  - invoices, billing agents and payments
  - overdue job and reminder emails
  - invoice PDF
  - publish `billora.invoices`
- [ ] **M6.3** Federation: `Invoice.client` and `Invoice.project` resolved by Crewboard; Billora contributes `Project.invoicedTotal`.
- [ ] **M6.4** Billora web.

### M7 — Portal dashboards and polish (weeks 20–21) `[FE: Emmanuel leads]`
- [ ] **M7.1** Portal home: widgets, My tasks, notifications, announcements, global search.
- [ ] **M7.2** Observability profile, with teaching (ARCHITECTURE §9). Capture a trace screenshot for the README.
- [ ] **M7.3** Full Playwright suite running in CI against the `all` profile.
- [ ] **M7.4** Final READMEs:
  - per service and root
  - screenshots and GIFs
  - "what this demonstrates" table
  - a 3-minute walkthrough video script

## 5. README requirements
**Root README:**
- one-paragraph pitch and the clean-room statement
- Mermaid architecture diagram
- quickstart (`git clone`, `make up`, a table of URLs)
- the demo-users table
- a "what this demonstrates" table mapping features to CV skills
- CI badges and links to every service README

**Each service README:**
- what the service owns
- its API link (Swagger UI or Apollo Sandbox)
- events it produces and consumes
- its permissions table
- one screenshot or GIF
- how to run it alone with `make dev-<service>`

## 6. Demo users
Seeded when `DEMO_SEED=true`. Every demo user's password is `Opsweave@2026`. Auth skips the invite step for seeded users only.

| Email | Role in the demo |
|---|---|
| ama.mensah@opsweave.com | Super Admin: HR, ADMIN in every app |
| kofi.boateng@opsweave.com | Head of FIN: Ledgerline step-1 approver, Requisio approver |
| efua.asante@opsweave.com | Finance officer: Ledgerline finance approver, marks vouchers paid |
| yaw.owusu@opsweave.com | Payroll officer (Paystream) |
| chidi.okeke@opsweave.com | OPS staff, Lagos: requester in Ledgerline and Requisio |
| abena.darko@opsweave.com | Billing manager (Billora) |
| kwame.addo@opsweave.com | Resource manager (Crewboard) |
| esi.quaye@opsweave.com | Engineer with Roster access only |
