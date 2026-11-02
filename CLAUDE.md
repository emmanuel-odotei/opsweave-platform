# CLAUDE.md — MosaicHub Platform

You are pair-building **MosaicHub**, a fictional company's internal platform, with Emmanuel Odoi Odotei, a Java backend engineer who is learning frontend and observability on this project. It is a portfolio monorepo: it must run on any laptop with one command and be readable by a recruiter in minutes.

## Before you write code
1. Read `docs/PORTFOLIO_PLAN.md` to find the current milestone and the next unchecked ticket. Work on that ticket only.
2. Read the `SPEC.md` of every service the ticket touches (`services/<name>/SPEC.md`, `apps/portal/SPEC.md`).
3. Read `docs/ARCHITECTURE.md` before touching auth, access control, Kafka, files, APIs, tests, Docker or ports. It is the single source of truth for platform-wide rules; a service spec only adds to it.
4. Read `docs/EVENTS.md` before producing or consuming any Kafka message.
5. Read `docs/TEACHING.md` before any session that introduces a concept on its *new to Emmanuel* list, and before any frontend work — it sets who writes the code in each milestone.
6. Read `design/DESIGN_SYSTEM.md` and the screen exports in `design/<app>/` before building any UI.

## Session rhythm
Emmanuel works in 2-hour weekday sessions. Each session is **one ticket, one teaching concept, one commit** (Conventional Commits, e.g. `feat(ledgerline): two-step voucher approval`). End every session by: running the affected tests, ticking the ticket in `docs/PORTFOLIO_PLAN.md`, appending to `LEARNING_LOG.md` if a concept was taught, and stating in two lines what the next session starts with.

## Standing rules
- **Clean-room.** Everything is written fresh from public patterns. Use only the fictional MosaicHub domain and seed data. Never ask for, reproduce or imitate code, schemas, names or business rules from Emmanuel's employer or its clients.
- **Pinned versions.** Resolve the newest stable patch at bootstrap, check compatibility (Spring Boot 4.1.x ↔ Spring Cloud ↔ springdoc ↔ federation-jvm; Apollo Router ↔ Federation v2), pin exact versions, and record them in `docs/VERSIONS.md`. Choose releases at least 14 days old.
- **PostgreSQL is the only database.** MinIO holds files; Kafka holds events.
- **Tests travel with code.** A ticket is done when its tests pass and JaCoCo stays ≥ 85% for the service.
- **Ask, don't invent.** If a spec is silent on a business rule, stop and ask Emmanuel; record the answer in that `SPEC.md`.
