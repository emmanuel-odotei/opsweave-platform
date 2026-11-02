# Opsweave Portal — SPEC

The Portal is the entry point at `http://localhost:8080/`. It has no backend of its own: it calls Auth and Roster (REST) and the Router (GraphQL `viewer`). Frontend architecture rules are in `docs/ARCHITECTURE.md` §13; look and feel is in `design/DESIGN_SYSTEM.md`.

## Pages

### Public pages (no token)
- `/login`: email and password, "Forgot password?", and errors for invalid credentials and locked accounts (with an unlock time). A hero panel carries the Opsweave woven-thread motif.
- `/set-password?token=`: checks the token first, then a password field with live strength rules (ARCHITECTURE §3) and a confirm field, then sends the user to login.
- `/forgot-password`: an email field. Always shows the same confirmation message.

### Home (`/`, authenticated)
Laid out top to bottom:
1. **Greeting header:** "Good morning, Ama", today's date, and a hero band.
2. **App launcher:** a tile for each app in the token's `apps`, with an icon, name, one-line purpose and a badge (that app's task count). Finance & Ops shows one tile with sub-links to Ledgerline, Requisio and Paystream when the user has any of them.
3. **Summary widgets:** a responsive grid of the `summary` numbers from each app (one `viewer` GraphQL query plus `GET /roster/me/summary`). A widget whose app fails to load shows a small error state, not a broken page.
4. **My tasks:** a merged list from every app's `tasks`, newest first, each with the app's icon and a deep link. Filter chips per app.
5. **Announcements:** pinned first, then the latest 5, with "View all" leading to an announcements page.
6. **Notifications:** a bell in the top bar with a dropdown of the merged list, "mark all read" (calls each app's mutation or endpoint), and an unread count.

### Top bar (on every page)
- **Global search** with ⌘K / Ctrl+K: searches the employee directory (`/roster/directory`) and shows avatar, name, title, department and office. Selecting a result opens a profile card.
- **App switcher** (grid icon).
- **Notifications bell.**
- **Profile menu:** avatar and name, "My profile" (→ Roster), "Change password", the light/dark/system theme toggle, "Log out" (`POST /auth/logout`, then clear memory, then `/login`).

### Pages Portal links to but doesn't own
"My profile" and "My payslips" live in Roster web.

## Shared app shell (every app, from `packages/ui`)
- **Layout:** a left sidebar (collapsible, with the app's sections) and the top bar above.
- **Access states:**
  - User has app access but no access level → a full-page "You don't have access to this app yet" view, naming the app's admins (the first 3 members of its ADMIN level, from that app's `viewer` data).
  - User has no app access → redirect to the Portal with a toast.
- **Session:** on load the shell silently calls `POST /auth/refresh`. When a call returns 401, it refreshes once, retries, and otherwise goes to `/login?next=`.
- **Finance & Ops shell:** the sidebar groups Ledgerline, Requisio and Paystream, showing only the sections the user has access to. Each section has its own dashboard, its main lists, then Access levels and Audit, each shown only with the matching permission.

## Tests
- RTL tests:
  - the launcher shows only apps in the token
  - widgets degrade when one subgraph errors
  - the theme toggle persists in `localStorage` under the key `ow-theme`
  - logout clears state
- Playwright: login → home → open Ledgerline from a tile → back → log out.
