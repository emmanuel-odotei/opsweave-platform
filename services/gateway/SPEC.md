# Gateway and Apollo Router — SPEC

## Gateway (`services/gateway`, Spring Cloud Gateway, WebFlux)
This is the only entry point, at `http://localhost:8080`.

### Routes

| Path | Target | Gateway check |
|---|---|---|
| `/api/v1/auth/**` | auth:8081 | none (Auth protects its own endpoints) |
| `/api/v1/roster/**` | roster:8082 | valid JWT |
| `/graphql` | router:4000 | valid JWT, except `OPTIONS`. Apollo Sandbox is reachable at `/graphql` in the `local` profile only. |
| `/api/v1/*/docs/**`, `/v3/api-docs/**` | matching service | none in local |
| `/` and `/{roster,finance,billora,crewboard}/**` | web:8088 | none |

### Behaviour
- Validates the JWT against Auth's JWKS. Rejects `typ: service` tokens coming from the browser edge.
- Adds `X-Correlation-Id` if it's missing and passes it downstream.
- Passes the `Authorization` header through unchanged.
- App checks for REST paths happen in the services. The Gateway's job is authentication and routing; services own authorization.
- Security headers: CSP allowing self only, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors 'none'`.
- Route targets come from env (`ROSTER_URL=http://roster:8082`, …), so `make dev-roster` can point at `host.docker.internal:8082`.

## Apollo Router (`infra/router`)
- `router.yaml`:
  - propagate the `Authorization` and `X-Correlation-Id` headers to every subgraph
  - CORS off (single origin)
  - introspection and Sandbox enabled only when `APP_ENV=local`
  - query depth limit 10
  - OTLP tracing export in the `observability` profile
- `supergraph.yaml` lists the subgraphs: ledgerline, requisio, paystream, billora, crewboard.
- `make supergraph` runs `rover supergraph compose` and commits `supergraph.graphql`. CI fails if the committed file differs from a fresh composition.
- If a subgraph is down, queries that touch it return partial data with errors. The frontend shows that section's error state instead of failing the whole page.

## Tests
- Gateway: route tests with WireMock backends, JWT rejection, correlation-ID injection.
- Router: a composition check in CI.
