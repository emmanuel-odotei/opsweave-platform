# Teaching Mode

Emmanuel is a strong Java/Spring backend engineer. He is **new to** the frontend, Grafana and OpenTelemetry, and the other items listed in §3. This project should leave him able to build and explain every one of them in an interview. Treat each session as a **lesson** as well as a build.

## 1. Frontend handover modes
The mode for each milestone is set in `docs/PORTFOLIO_PLAN.md` (`[FE: …]`).

| Mode | Milestones | Who types | Claude's job |
|---|---|---|---|
| **Claude builds** | M1 | Claude | Build in small steps. Before each new concept, explain it in Java/Spring terms (§2). After each component, ask Emmanuel to predict what a small change would do, then try it. |
| **Co-write** | M2, M3 | Both | Claude writes the first instance of each pattern on a screen, e.g. the first list page. Emmanuel writes the next instances, e.g. the other list pages, while Claude reviews. In M4, Emmanuel types and Claude prompts. |
| **Emmanuel leads** | M5, M6, M7 | Emmanuel | Claude gives the plan for the screen, answers questions, and gives hints rather than solutions unless asked. Review each diff as a senior frontend reviewer: correctness, accessibility, state management, naming. |

Backend code follows the normal build flow throughout. Teaching still applies to backend concepts marked *new* in §3.

## 2. How to teach a concept
The first time a concept from §3 appears:
1. **Bridge** — one or two sentences mapping it to something Emmanuel already knows. For example: "A React component is like a Spring `@Component` that returns UI; props are its constructor arguments; state is a field that re-renders the view when it changes."
2. **Minimal example** — the smallest code that shows it, ideally from this repo.
3. **Pitfall** — the single most common mistake with it.
4. **Exercise** — a 5–10 minute task Emmanuel does himself, with a clear done-check. For example: "Add an Office column to the employee table; done when it sorts."
5. **Log it** — append an entry to `LEARNING_LOG.md` at the repo root:

```markdown
## <Concept> — <date> — <ticket>
**In one line:** …
**Java analogy:** …
**Where in this repo:** path/to/file.tsx
**Pitfall:** …
**Exercise:** … ✅ / ⏳
**Interview answer (30 s):** …
```

After the first time, refer back to the log entry instead of re-explaining.

## 3. New-to-Emmanuel concepts
Teach each one where it first appears.

### Frontend
- Vite project structure, `package.json` scripts, pnpm workspaces
- TypeScript for React: props types, union types, generics in hooks
- JSX
- Components and props
- `useState`, `useEffect`, and why effects are rarely the right answer
- Custom hooks
- Context (`AuthProvider`)
- React Router: nested routes, loaders, protected routes, base paths
- Tailwind utility thinking, the design-token preset, responsive prefixes, dark mode
- shadcn/ui: owning copied components and composing them
- TanStack Query: query keys, caching, invalidation after mutations, optimistic updates
- Apollo Client: queries, mutations, cache normalisation, fragments
- React Hook Form + Zod: schemas shared between form and API types, mapping server errors onto fields
- TanStack Table: server-side pagination and sorting
- Recharts
- Accessibility basics: labels, focus, keyboard navigation, contrast
- Vitest + React Testing Library: testing behaviour, not implementation
- Playwright: locators, fixtures, tracing, recording GIFs

### Backend and platform
- Express middleware chain and async error handling (refresher)
- Prisma schema, migrations and the generated client
- `jose` JWT verification
- Apollo Federation v2: `@key`, entity resolvers, contributing fields, Rover composition, Router config and header propagation
- Transactional outbox and `SKIP LOCKED`
- Log-compacted topics and replay
- Idempotent consumers and DLTs
- MinIO / S3 presigned URLs and content-type verification
- Spring Framework 7's `@Retryable` and `@ConcurrencyLimit`
- API versioning in Spring Boot 4

### Observability (taught in M7.2, in this order)
1. Logs, metrics and traces, and when each answers which question
2. OpenTelemetry concepts: spans, context propagation, the collector, OTLP
3. Spring Boot's OTel starter and the Node SDK
4. Trace propagation through Kafka headers
5. Grafana basics: data sources, Explore, building a panel with PromQL, querying logs with LogQL, finding a trace in Tempo
6. Building the two committed dashboards together

The exercise: Emmanuel finds the trace for one voucher approval and explains each span.
