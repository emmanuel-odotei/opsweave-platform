# Mosaic Crewboard (Resourcing) — SPEC

- **Owns:** clients, projects, employee-to-project assignments, skills.
- **Publishes:** `crewboard.clients`, `crewboard.projects`.
- **Consumes:** `roster.employees`, `roster.organizations`, `roster.offices`, `roster.departments`, `billora.invoices`.
- **API:** GraphQL subgraph (Apollo Server on Express).
- **UI:** `apps/crewboard-web` (`/crewboard`).

This is the only Node service. It follows the same platform rules as the Java services: the access model, outbox, idempotent consumers, Problem Details for its health and error routes, correlation IDs and OTel.

## Code layout
```
services/crewboard/
  prisma/schema.prisma, prisma/migrations/
  src/
    config/        env parsing with Zod (fails fast)
    auth/          jose JWKS verification, AccessContext (mirrors Java)
    access/        permission union type, scope filters, seeded ADMIN level
    graphql/       schema/*.graphql, resolvers/, federation entities
    modules/       clients/ projects/ assignments/ skills/ (service + repository per module)
    kafka/         outbox relay, consumers, envelope + schema validation
    middleware/    correlation id, request logging, error handler
    observability/ OTel SDK bootstrap
    server.ts
  test/            Vitest unit + Testcontainers integration
```
The layering mirrors Spring: resolver ≈ controller, service ≈ @Service, repository ≈ Spring Data, with Prisma as the ORM. Teach this mapping in M5.1.

## Permissions (`CrewboardPermission`, scope `ORG_OFFICE` unless noted)

| Permission | Allows |
|---|---|
| `VIEW_PROJECT` | see clients, projects and assignments for projects in scope (by the project's delivering org) |
| `MANAGE_CLIENTS` (NONE) | create, edit and archive clients |
| `MANAGE_PROJECTS` | create, edit and change status of projects in scope |
| `MANAGE_ASSIGNMENTS` | assign employees in scope to projects |
| `VIEW_CAPACITY` | bench and utilisation views for employees in scope (by the employee's org and office) |
| `MANAGE_SKILLS` (NONE) | skill catalogue, and editing anyone's skills |
| `VIEW_DASHBOARD` | delivery and revenue dashboard |
| `MANAGE_ACCESS_LEVELS` (NONE), `VIEW_AUDIT` (NONE) | standard |

Anyone with Crewboard access and at least one level sees **My assignments** and can edit **their own skills**.

## Entities and rules
- **Client:**
  - Fields: `name`, `industry`, `country`, `billingEmail`, `defaultCurrency`, `status` (`ACTIVE` | `ARCHIVED`), `contacts[]` (name, email, phone, role).
  - Any create or change publishes on `crewboard.clients`.
- **Project:**
  - Fields:
    - `code` (`PRJ-<year>-<4 digits>`), `name`, `clientId`, `deliveringOrgId`, `description`
    - `status` (`PLANNED` | `ACTIVE` | `ON_HOLD` | `COMPLETED` | `ARCHIVED`)
    - `startDate`, `endDate`, `projectLeadUserId`
    - `budgetHours` (optional)
  - Publishes on `crewboard.projects`.
  - A project can't be created for an archived client, and a client can't be archived while it has active projects.
- **Assignment:**
  - Fields: `projectId`, `userId`, `role` (free text with suggestions), `allocationPct` (5–100 in steps of 5), `startDate`, `endDate` (within the project's dates), `billable`.
  - Only `ACTIVE` employees can be assigned.
  - When an employee is deactivated, their future assignments end on the deactivation date and the project lead is notified.
- **Over-allocation:** for every day in the new assignment's range, the sum of the employee's overlapping allocations. Above 100% the save returns the warning `OVER_ALLOCATED` with the conflicting assignments; saving again with `confirmOverAllocation: true` stores it with `overAllocated = true`.
- **Skills:** a catalogue (name, category), and employee skills with a level from 1 to 5.
- **Bench:** employees in scope with an average allocation under 50% over a chosen window (default: the next 4 weeks), filterable by skill and office.
- **Revenue (from `billora.invoices`):** a local `project_invoices` read model powers the dashboard's revenue-over-time and revenue-per-client charts. The **live** per-project totals on the project page come from Billora through federation (`Project.invoicedTotal` and `paidTotal`). Both are deliberate: one shows Kafka read models, the other shows federated fields.

## GraphQL (sketch)
```graphql
type Client  @key(fields: "id") { id: ID! name: String! status: ClientStatus! defaultCurrency: Currency! projects: [Project!]! }
type Project @key(fields: "id") { id: ID! code: String! name: String! client: Client! status: ProjectStatus!
                                  lead: EmployeeRef assignments: [Assignment!]! }
type Query {
  clients(filter: ClientFilter, page: PageInput): ClientPage!
  client(id: ID!): Client
  projects(filter: ProjectFilter, page: PageInput): ProjectPage!
  project(id: ID!): Project
  myAssignments: [Assignment!]!
  capacity(window: DateRange!, officeId: ID, skill: String): [EmployeeCapacity!]!
  bench(window: DateRange!, skill: String): [EmployeeCapacity!]!
  crewboardDashboard(range: DateRange!): CrewboardDashboard!
  skills: [Skill!]!
}
type Mutation {
  saveClient(input: ClientInput!): ClientPayload!
  archiveClient(id: ID!): ClientPayload!
  saveProject(input: ProjectInput!): ProjectPayload!
  setProjectStatus(id: ID!, status: ProjectStatus!): ProjectPayload!
  saveAssignment(input: AssignmentInput!, confirmOverAllocation: Boolean): AssignmentPayload!
  endAssignment(id: ID!, endDate: Date!): AssignmentPayload!
  setMySkills(skills: [EmployeeSkillInput!]!): SkillsPayload!
  setEmployeeSkills(userId: ID!, skills: [EmployeeSkillInput!]!): SkillsPayload!
}
type Viewer @key(fields: "userId") { userId: ID! crewboard: CrewboardViewer }
# summary { myAllocationPct, myActiveProjects, benchCount (if VIEW_CAPACITY), projectsEndingSoon }
```

## Screens
1. **Dashboard:**
   - utilisation % over time
   - billable vs non-billable allocation
   - projects by status
   - revenue per client (from the read model)
   - projects ending in the next 30 days
2. **Clients:** list and detail (contacts, projects, revenue).
3. **Projects:** list and detail, with a team table, an **allocation timeline** (horizontal bars per person) and live invoiced/paid totals.
4. **Assign employee:** a drawer with an employee picker that shows current allocation and skills, plus an over-allocation warning dialog.
5. **Capacity and bench:** a heatmap grid of people × weeks, coloured by allocation %.
6. **My assignments** and **My skills**.
7. **Access levels**, **Audit**.

## Tests
- Over-allocation calculation across overlapping ranges (table-driven).
- Deactivation ending future assignments.
- Archive guards.
- Outbox relay and consumer idempotency against real Kafka and Postgres (Testcontainers).
- `jose` verification against a mock JWKS.
- Federation `__resolveReference`.
- Coverage ≥ 85%.
